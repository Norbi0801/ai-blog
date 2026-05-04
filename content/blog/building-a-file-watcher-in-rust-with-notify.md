+++
title = "Building a file watcher in Rust with notify"
date = 2025-06-24
description = "How the notify crate wraps inotify, FSEvents, and ReadDirectoryChangesW behind a single API, why naive watching produces duplicate events, and how to wire it into a tokio app cleanly."

[taxonomies]
tags = ["rust", "tokio", "filesystem", "tools"]
+++

Almost every dev tool you use watches files. `cargo watch` reruns your tests when source changes. `vite` and `webpack-dev-server` reload the browser when CSS changes. `mdbook serve` rebuilds your book. `tail -f` follows a log. `rust-analyzer` reindexes when you edit. `tsc --watch`, `nodemon`, `entr`, `watchexec` - all of them sit on the same primitive: ask the kernel to tell us when something on disk changes, then react.

The Rust ecosystem standardized on the [`notify`](https://github.com/notify-rs/notify) crate for this. It's used by [alacritty](https://github.com/alacritty/alacritty), [deno](https://github.com/denoland/deno), [mdBook](https://github.com/rust-lang/mdBook), [rust-analyzer](https://github.com/rust-lang/rust-analyzer), [watchexec](https://github.com/watchexec/watchexec), and [zed](https://github.com/zed-industries/zed). The current version at the time of writing is 8.2.0, and the API has stabilized after years of churn.

This post walks through what `notify` actually does under the hood, why a naive use of it will fire your callback five times for one save, and how to turn it into a clean async stream you can `select!` over with the rest of your tokio app.

<!-- more -->

## What the kernel actually gives you

There is no portable filesystem-change API. Each OS exposes its own:

- **Linux** uses [`inotify`](https://man7.org/linux/man-pages/man7/inotify.7.html), a kernel subsystem that returns events through a file descriptor you can `read()` or `epoll`. You add watches with `inotify_add_watch(fd, path, mask)` and get back a watch descriptor. Events arrive as `struct inotify_event` records packed into a single buffer. Each watch is per-directory, not recursive - if you want a tree, you walk it and add a watch per directory.
- **macOS** uses [FSEvents](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/FSEvents_ProgGuide/Introduction/Introduction.html), a higher-level system that batches events per directory and delivers them through a Core Foundation run loop callback. FSEvents is recursive by design and includes coalescing windows. There is also `kqueue`, which is finer-grained but doesn't scale to large trees because it requires one open file descriptor per watched item.
- **Windows** uses [`ReadDirectoryChangesW`](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-readdirectorychangesw), an overlapped I/O call that fills a buffer with `FILE_NOTIFY_INFORMATION` records. It supports recursion via the `bWatchSubtree` flag.
- **BSD** uses kqueue with `EVFILT_VNODE`.

These APIs disagree on almost everything: granularity (one event vs. coalesced batch), recursion (per-directory vs. tree), what counts as a "modify" (open-with-write? close-after-write? actual byte change?), and how renames are reported. On Linux a rename is two events (`MOVED_FROM` and `MOVED_TO`) tied together by a cookie. On macOS it's one event with a "renamed" flag. On Windows it's `FILE_ACTION_RENAMED_OLD_NAME` and `FILE_ACTION_RENAMED_NEW_NAME` in succession.

`notify` papers over this. You write your code against `Watcher` and `Event`, and the crate picks the right backend at runtime via `RecommendedWatcher`. The cost of that abstraction is that the event stream you get is the *intersection* of guarantees - you cannot rely on rename-pairing on macOS the way you can on Linux, and you should treat `EventKind::Modify` as "something probably changed, go check."

## The minimum example

```toml
# Cargo.toml
[dependencies]
notify = "8.2"
```

```rust
use notify::{recommended_watcher, RecursiveMode, Watcher, Event};
use std::path::Path;
use std::sync::mpsc::channel;

fn main() -> notify::Result<()> {
    let (tx, rx) = channel::<notify::Result<Event>>();

    let mut watcher = recommended_watcher(tx)?;
    watcher.watch(Path::new("./src"), RecursiveMode::Recursive)?;

    for res in rx {
        match res {
            Ok(event) => println!("{:?}", event),
            Err(e) => eprintln!("watch error: {:?}", e),
        }
    }
    Ok(())
}
```

Run it, then `touch src/foo.rs` in another terminal. You'll see something like:

```
Event { kind: Create(File), paths: ["./src/foo.rs"], attrs: {} }
Event { kind: Modify(Metadata(Any)), paths: ["./src/foo.rs"], attrs: {} }
```

Now save that file in your editor. You will probably see *three to five* events, not one. That's the first problem.

## Why a single save fires multiple events

Editors do not write files the way you think they do. Most modern editors (vim, VS Code, IntelliJ, Helix, Zed) use the *atomic save* pattern:

1. Write the new content to a temp file (`foo.rs~` or `foo.rs.swp` or `.foo.rs.tmp.XYZ`).
2. `fsync` it to flush to disk.
3. `rename` the temp over the original.
4. Sometimes update the original file's metadata.

From `inotify`'s point of view, that's a `CREATE` of the temp file, several `MODIFY` events as bytes are written, a `CLOSE_WRITE`, then a `MOVED_FROM`/`MOVED_TO` pair as the rename swaps in. Your "one save" produces a fan of low-level events. Other editors (the old ones, or `echo > file`) write in place, which gives you a `MODIFY` plus a metadata update. Both patterns are valid, and both cause the same headache: you get a flurry of events for a single logical change.

If you naively rebuild on every event you'll trigger your build pipeline several times per save. That's where debouncing comes in.

## Debouncing with `notify-debouncer-full`

`notify` ships two companion crates: [`notify-debouncer-mini`](https://crates.io/crates/notify-debouncer-mini) (lightweight, just collapses events per path within a window) and [`notify-debouncer-full`](https://crates.io/crates/notify-debouncer-full) (tracks rename cookies, deduplicates, and gives you ordered batches).

For most build tool use cases, `notify-debouncer-full` is what you want.

```toml
[dependencies]
notify-debouncer-full = "0.6"
```

```rust
use notify_debouncer_full::{new_debouncer, DebounceEventResult};
use notify::RecursiveMode;
use std::path::Path;
use std::sync::mpsc::channel;
use std::time::Duration;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let (tx, rx) = channel::<DebounceEventResult>();

    let mut debouncer = new_debouncer(
        Duration::from_millis(250),  // batch window
        None,                         // tick rate (None = default)
        tx,
    )?;

    debouncer.watch(Path::new("./src"), RecursiveMode::Recursive)?;

    for result in rx {
        match result {
            Ok(events) => {
                for ev in events {
                    println!("{:?} -> {:?}", ev.kind, ev.paths);
                }
            }
            Err(errors) => eprintln!("debounce errors: {:?}", errors),
        }
    }
    Ok(())
}
```

The `Duration::from_millis(250)` is the key. The debouncer collects events for that long after the last one arrives, then ships the batch as a single `Vec<DebouncedEvent>`. A 250ms window is the sweet spot for editors: long enough to catch the temp-file dance, short enough that the user doesn't notice the lag. Cargo watch defaults to 2 seconds because it expects a build pipeline to take much longer; pick the window for your workload.

`notify-debouncer-full` also keeps a small in-memory model of the watched tree so it can pair `Remove`+`Create` events that look like a rename, and so it can report a single logical "modify" for an atomic save instead of the underlying create-write-rename storm.

## Wiring it into tokio

`notify` runs its callback on a background OS thread that the watcher spawns internally. You can't call `tokio` primitives from inside that callback because that thread isn't a tokio runtime worker. The bridge is a channel: have the callback push into a tokio `mpsc` and consume it from your async code.

```rust
use notify_debouncer_full::{new_debouncer, DebouncedEvent, DebounceEventResult};
use notify::RecursiveMode;
use std::path::Path;
use std::time::Duration;
use tokio::sync::mpsc;

pub fn watch(path: &Path) -> notify::Result<mpsc::Receiver<Vec<DebouncedEvent>>> {
    let (tx, rx) = mpsc::channel::<Vec<DebouncedEvent>>(64);

    let mut debouncer = new_debouncer(
        Duration::from_millis(250),
        None,
        move |res: DebounceEventResult| {
            if let Ok(events) = res {
                // blocking_send is fine here - we're on a dedicated thread
                let _ = tx.blocking_send(events);
            }
        },
    )?;

    debouncer.watch(path, RecursiveMode::Recursive)?;

    // Leak the debouncer for the program lifetime, or stash it somewhere.
    // If it's dropped, the OS-level watch goes away.
    Box::leak(Box::new(debouncer));

    Ok(rx)
}
```

Two things to call out. First, `blocking_send` is intentional: the callback runs on the watcher's own thread, which is not async, so you cannot `.await`. Use `try_send` if you'd rather drop events under backpressure than block. Second, the watcher *must outlive* the period during which you want events. Dropping it unregisters the OS-level watch silently. In a real app you'd hold it in your application state, not leak it.

Now you can `select!` over filesystem events alongside HTTP requests, signals, anything:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut fs_events = watch(Path::new("./src"))?;
    let mut shutdown = tokio::signal::ctrl_c();

    loop {
        tokio::select! {
            Some(batch) = fs_events.recv() => {
                handle_changes(batch).await;
            }
            _ = &mut shutdown => {
                println!("bye");
                break;
            }
        }
    }
    Ok(())
}

async fn handle_changes(batch: Vec<DebouncedEvent>) {
    let changed: Vec<_> = batch.iter()
        .flat_map(|e| e.paths.clone())
        .filter(|p| p.extension().map(|x| x == "rs").unwrap_or(false))
        .collect();

    if changed.is_empty() { return; }
    println!("rebuilding due to: {:?}", changed);
    // spawn cargo, kick off your pipeline, etc.
}
```

That's the entire pattern most "watch and reload" tools use. From here it's just policy: which extensions to react to, which paths to ignore (`target/`, `.git/`, `node_modules/`), whether to debounce at the app level on top of the debouncer, whether to coalesce builds.

## Recursive watching: it's not free

`RecursiveMode::Recursive` is one line in your code, but the work behind it depends on the platform.

On **macOS**, FSEvents is natively recursive. The kernel maintains a per-directory event stream and the cost of "recursive" is roughly the cost of "watch one path." Big trees are cheap.

On **Windows**, `ReadDirectoryChangesW` takes a `bWatchSubtree` flag and the kernel handles recursion. Same story: cheap.

On **Linux**, `inotify` is *per-directory only*. To watch a tree, `notify` walks it and calls `inotify_add_watch` for every directory. Each watch consumes a slot in `/proc/sys/fs/inotify/max_user_watches`, which defaults to 8192 on most distros and 524288 on newer ones. A monorepo with `node_modules/` can blow past the default in seconds. The error you get is `ENOSPC: No space left on device`, which is misleading - it has nothing to do with disk space. You raise the limit with:

```bash
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

This is also why production tools (rust-analyzer, VS Code) ship explicit ignore lists: every excluded directory is a watch you don't have to spend.

## Filtering events

The debouncer gives you everything in the watched tree. You almost certainly want to filter. The relevant fields on `DebouncedEvent`:

- `kind: EventKind` - `Create`, `Modify`, `Remove`, `Access`, `Other`. `Modify` has subkinds for `Data`, `Metadata`, `Name`. For "rebuild on source change" you typically only care about `Create`, `Remove`, and `Modify(Data)` or `Modify(Name)`.
- `paths: Vec<PathBuf>` - usually one path, two for renames on platforms that report them as one event.

A practical filter for a build tool:

```rust
fn is_interesting(event: &DebouncedEvent) -> bool {
    use notify::EventKind::*;
    use notify::event::ModifyKind;

    matches!(
        event.kind,
        Create(_) | Remove(_) | Modify(ModifyKind::Data(_)) | Modify(ModifyKind::Name(_))
    ) && event.paths.iter().any(|p| {
        let s = p.to_string_lossy();
        !s.contains("/target/")
            && !s.contains("/.git/")
            && !s.contains("/node_modules/")
            && p.extension().map(|x| x == "rs" || x == "toml").unwrap_or(false)
    })
}
```

`Modify(Metadata(_))` is noisy: it fires for `chmod`, `touch`, and on some systems just for opening a file. Drop it unless you specifically care about permissions changing.

## Use cases beyond build tools

The same primitive shows up in surprising places.

**Log monitoring.** A `tail -f` clone is a watcher on a single file, plus a seek to the end and a read of the new bytes whenever you get a `Modify(Data)`. The trick is handling log rotation: your `File` handle still points at the old, unlinked inode while the new logs go to a fresh file with the same name. You detect rotation by watching the *directory* for `Create` events on the same path, then re-opening.

**Hot config reload.** Watch your config file. On `Modify`, parse it into a candidate value, validate, and atomically swap it into an `ArcSwap` that the rest of the app reads. Failed parses log a warning but keep the old config live. This is how a lot of long-running servers (envoy, traefik, nginx with `reload`) avoid restarts.

**Indexers.** rust-analyzer watches your source tree to know what to reparse. The interesting design choice is that it does *not* trust the events for correctness - it uses them as hints to invalidate cached parse trees, then re-reads from disk. Filesystem events can be lost (`inotify` queues are bounded), so anything that needs to be correct should use them as a "go check" signal, not as ground truth.

**Live reload for static sites.** Watch `content/` and `templates/`, rebuild on change, and push a websocket message to connected browsers telling them to reload. Zola, Hugo, and mdBook all do this. The piece that's often missing from tutorials: rebuild and websocket-push need to be debounced together, otherwise you'll send three reloads for one save and the browser will flicker.

## What to remember

- `notify` wraps inotify, FSEvents, and ReadDirectoryChangesW behind one API. The platform differences leak through in event granularity, especially around renames.
- A single editor save produces a burst of low-level events. Always debounce. `notify-debouncer-full` with a 100-300ms window is the right default for interactive tools.
- The watcher callback runs on a dedicated OS thread, not a tokio worker. Bridge to async with a channel and `blocking_send`.
- Recursive watching is free on macOS and Windows but expensive on Linux because `inotify` is per-directory. Watch your `max_user_watches` and exclude `target/`, `node_modules/`, `.git/`.
- Treat events as hints, not ground truth. The kernel queues are bounded and events can be dropped under load. If correctness matters, re-read the file when you act.

The beauty of standing on top of `notify` is that you write your tool once and it works on every OS your users have. The pain is that the abstraction is leaky in exactly the places you'd expect: the kernel APIs disagree, and no library can paper over that completely. Build with the leaks in mind, debounce hard, and treat the event stream as a notification that something *might* have changed - not as a transaction log.
