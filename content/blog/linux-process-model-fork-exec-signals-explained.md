+++
title = "Linux process model - fork, exec, signals explained"
date = 2025-11-22
description = "A deep look at how processes actually work on Linux - fork, exec, signals, zombies, /proc - and what Rust's Command::new does under the hood."

[taxonomies]
tags = ["linux", "systems-programming", "rust", "unix"]
+++

You run `./server &` in one terminal, press Ctrl+C in another, and somehow the right thing happens. The shell survives, the server stops, and the kernel cleans up. Nothing about that is magic - it is a pile of syscalls, signal delivery rules, and process accounting that has barely changed since 1975. If you write server software without understanding these rules you will eventually ship a daemon that leaks zombies, ignores SIGTERM, or never reaps its children.

This post walks through the Linux process model from the bottom up. We will trace the exact syscalls that happen when a process is born, how it gets replaced by another program, how signals are delivered and handled, what `/proc` actually contains, and how Rust's `Command::new(...).spawn()` maps to all of it. Some of this is also touched on in [Writing a shell in Rust](/blog/writing-a-shell-in-rust/), but here we stay at the OS layer.

<!-- more -->

## What a process actually is

A process on Linux is a `struct task_struct` inside the kernel. Every running program has one. It tracks the PID, the parent PID, open file descriptors, the memory map, signal handlers, credentials, CPU scheduling state, and about 250 other fields. The definition lives in [`include/linux/sched.h`](https://github.com/torvalds/linux/blob/master/include/linux/sched.h) and is over 2000 lines long.

From userspace you never touch `task_struct` directly. You see it through three interfaces:

- **Syscalls** - `fork`, `execve`, `wait4`, `kill`, `clone`, `prctl`, ...
- **`/proc/<pid>/`** - a synthetic filesystem that exposes one directory per process
- **Signals** - asynchronous notifications the kernel delivers into your process

Every process has at least these identifiers:

- `PID` - process ID, unique at a given moment, recycled eventually
- `PPID` - parent PID, the process that created it
- `PGID` - process group ID, used by terminals for job control
- `SID` - session ID, used for controlling terminal assignment
- `UID`, `GID` - user and group, for permission checks

Run `ps -o pid,ppid,pgid,sid,cmd` in a shell pipeline and you can see all four at once.

## fork - the syscall that clones a process

`fork()` is the only syscall on Linux that makes a new process. There is literally no other way. Even `posix_spawn` eventually calls `clone()` (fork's more flexible sibling) inside glibc.

The signature is deceptively simple:

```c
#include <unistd.h>
pid_t fork(void);
```

Call it once, return twice. The kernel duplicates the calling process and you now have two processes executing the same code. The return value tells them apart:

- In the parent, `fork()` returns the child's PID (a positive integer)
- In the child, `fork()` returns `0`
- On failure, `fork()` returns `-1` and sets `errno`

Here is the smallest possible demo in C:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    pid_t pid = fork();
    if (pid == 0) {
        printf("child, pid=%d ppid=%d\n", getpid(), getppid());
    } else {
        printf("parent, pid=%d child=%d\n", getpid(), pid);
        wait(NULL);
    }
    return 0;
}
```

Compile and run. You get two lines of output from one `main()` function. The child has a different PID, a different PPID (pointing to the parent), but the same open file descriptors, the same environment, the same memory contents at the moment of the fork.

### Copy-on-write is why fork is cheap

Naively, fork sounds expensive. If your process has a 4 GB heap, does the kernel copy 4 GB? No. The kernel marks every page as copy-on-write (COW). Both processes share the physical pages read-only. The first time either one writes to a page, the CPU triggers a page fault, the kernel makes a private copy for that process, and execution continues.

This is why `fork()` is typically microseconds, not milliseconds, even for processes with huge address spaces. It also means a forked child that immediately calls `exec` (throwing away the memory) pays almost nothing for the fork itself. That is the common case.

You can see COW at work by comparing `RssAnon` (resident anonymous pages) in `/proc/<pid>/status` before and after a forked child starts writing to memory.

## exec - replacing the program, not the process

`fork` gives you a copy of the current program. To actually run a different binary you call one of the `exec` family:

```c
int execve(const char *path, char *const argv[], char *const envp[]);
int execvp(const char *file, char *const argv[]);  // PATH lookup
int execl(const char *path, const char *arg0, ...);
```

All of them are wrappers around the `execve` syscall. `execve` is special: on success it does not return. The kernel replaces your current process image - memory, code, heap, stack - with the new program, resets the CPU state, and jumps to the new entry point. The PID stays the same. File descriptors stay open by default (unless marked `FD_CLOEXEC`). Signal handlers reset to their defaults because the code that implemented them no longer exists.

The classic pattern is fork + exec in the child, wait in the parent:

```c
pid_t pid = fork();
if (pid == 0) {
    execvp("ls", (char *const[]){"ls", "-la", NULL});
    perror("execvp");  // only reached if exec failed
    _exit(127);
} else {
    int status;
    waitpid(pid, &status, 0);
    printf("child exited with %d\n", WEXITSTATUS(status));
}
```

### Why split fork and exec?

Every other operating system has some equivalent of `CreateProcess` that takes the program to run and returns a new process. Unix split it in two on purpose. Between the fork and the exec, the child is still running the parent's code - which means you can manipulate file descriptors, set signal handlers, change the working directory, call `setuid`, drop capabilities, or install seccomp filters. Then you exec with all of that already in place.

That is how shells implement redirects and pipes. The shell forks, the child calls `dup2(pipe_write_end, STDOUT_FILENO)`, closes the original, and then execs. By the time the new program starts it has no idea its stdout is actually a pipe. It just writes to file descriptor 1 like normal.

## Process groups, sessions, and terminals

Every process belongs to a process group. Every process group belongs to a session. When you type Ctrl+C in a terminal, the terminal driver looks up the foreground process group of that session and sends `SIGINT` to every process in it. That is why Ctrl+C kills an entire pipeline, not just `cat`.

A new shell created with `setsid()` becomes the leader of a new session and detaches from any controlling terminal. This is how you daemonize a process. The canonical sequence is:

1. `fork()` and exit the parent (so the process is not a process group leader)
2. `setsid()` in the child (new session, new process group, no controlling tty)
3. `fork()` again (so we cannot accidentally acquire a tty later)
4. `chdir("/")`, `umask(0)`, close stdin/stdout/stderr or point them at `/dev/null`

Modern init systems like systemd remove the need for the double-fork dance because they start services directly with `exec`, keep them in the foreground, and capture stdout/stderr into the journal. Writing a proper daemonize by hand is mostly legacy.

## Signals

A signal is a small integer the kernel delivers to a process. When delivered, the process either runs a handler function, takes a default action (usually terminate, sometimes ignore, sometimes stop), or has the signal queued if it is currently blocked.

The ones you actually need to know:

| Signal    | Num | Default | Can catch? | Typical use                             |
|-----------|-----|---------|------------|-----------------------------------------|
| `SIGINT`  | 2   | Term    | yes        | Ctrl+C from terminal                    |
| `SIGTERM` | 15  | Term    | yes        | Polite "please shut down"               |
| `SIGKILL` | 9   | Term    | no         | Forceful kill, cannot be caught         |
| `SIGHUP`  | 1   | Term    | yes        | Controlling terminal closed; reload cfg |
| `SIGCHLD` | 17  | Ignore  | yes        | A child changed state                   |
| `SIGPIPE` | 13  | Term    | yes        | Wrote to a pipe with no readers         |
| `SIGSTOP` | 19  | Stop    | no         | Pause process, cannot be caught         |
| `SIGCONT` | 18  | Cont    | yes        | Resume a stopped process                |
| `SIGSEGV` | 11  | Core    | yes        | Invalid memory access                   |

`SIGKILL` and `SIGSTOP` are uncatchable. The kernel acts on them directly. Everything else you can catch with a handler, ignore, or block temporarily with `sigprocmask`.

Signal handlers run asynchronously on whatever thread the kernel picks, interrupting normal code flow. This makes them brutally restrictive - you can only call async-signal-safe functions (see [`signal-safety(7)`](https://man7.org/linux/man-pages/man7/signal-safety.7.html)). `printf`, `malloc`, most of libc is off limits. The modern pattern is to install a minimal handler that writes one byte to a `signalfd` or a self-pipe, then handle the signal in your main event loop.

### Graceful shutdown

For a server you almost always want to catch `SIGTERM` (and `SIGINT` during development), stop accepting new connections, drain in-flight requests, close sockets, flush logs, and exit 0. If you ignore `SIGTERM`, your orchestrator (systemd, Kubernetes, Docker) will wait a few seconds and then send `SIGKILL`. That means no flush, no drain, dropped requests.

In Rust with tokio:

```rust
use tokio::signal::unix::{signal, SignalKind};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut term = signal(SignalKind::terminate())?;
    let mut int = signal(SignalKind::interrupt())?;

    tokio::select! {
        _ = run_server() => {},
        _ = term.recv() => eprintln!("got SIGTERM, shutting down"),
        _ = int.recv()  => eprintln!("got SIGINT, shutting down"),
    }
    drain_and_exit().await
}
# async fn run_server() {}
# async fn drain_and_exit() -> std::io::Result<()> { Ok(()) }
```

Under the hood tokio sets up a signalfd and turns signal delivery into a normal async event, sidestepping the async-signal-safety minefield.

## Zombie and orphan processes

When a child exits, the kernel does not immediately free its `task_struct`. It keeps the exit status and a few accounting fields around so the parent can ask for them with `wait()` or `waitpid()`. Between "child exited" and "parent called wait" the child is a **zombie**. You will see it in `ps` with status `Z` and process name in angle brackets like `<defunct>`.

Zombies use almost no memory, but they occupy a PID slot. A process that spawns thousands of children without waiting for them will eventually exhaust PIDs and fail to fork anything new.

Two ways to avoid zombies:

1. Call `waitpid` (or `wait3`/`wait4`) for every child you spawn
2. Set `SIGCHLD` to `SIG_IGN` (or use `SA_NOCLDWAIT`), which tells the kernel "I do not care about exit status, reap them for me"

An **orphan** is the opposite problem: the parent died while the child is still running. Orphans get reparented to PID 1 (init or systemd), which always has a wait loop running. So orphans always get reaped eventually, they just change their PPID. This is exactly how nohup/disown/daemons work.

## The /proc filesystem

`/proc` is a synthetic filesystem exposed by the kernel. It is not on disk. Every read returns fresh data generated on demand. There is one directory per PID:

```
$ ls /proc/$$/
attr/    cmdline  cwd@     environ  exe@    fd/      limits   maps
mem      mounts   net/     ns/      pagemap root@    stat     status
...
```

A few that matter:

- `/proc/<pid>/cmdline` - null-separated argv
- `/proc/<pid>/environ` - null-separated environment
- `/proc/<pid>/status` - human-readable state, UIDs, memory, signals
- `/proc/<pid>/maps` - memory map (every mmap region)
- `/proc/<pid>/fd/` - symlinks to every open file descriptor
- `/proc/<pid>/exe` - symlink to the binary that was exec'd
- `/proc/<pid>/stat` - space-separated numeric fields used by `ps` and `top`

Almost every process inspection tool - `ps`, `top`, `htop`, `lsof`, `pstree`, `pmap` - is just a pretty wrapper around `/proc`. If you ever need a specific piece of info and the tool does not expose it, `cat /proc/<pid>/...` probably has it.

## How Rust's Command::new maps to all this

`std::process::Command` is Rust's portable subprocess API. On Linux, [the actual implementation](https://github.com/rust-lang/rust/blob/master/library/std/src/sys/process/unix/unix.rs) tries `posix_spawn` first when it is safe, otherwise falls back to `fork` + `exec`. What `spawn()` does is roughly:

1. Resolve the program path (respecting `PATH` if you used `Command::new("ls")`)
2. Serialize argv and envp into C strings
3. Either:
   - Call `posix_spawnp` if the configuration is simple (no pre-exec closure, no chroot, no setuid), OR
   - `fork()`, then in the child: set up stdin/stdout/stderr pipes with `dup2`, change working directory if requested, clear signal masks, then `execvp`
4. Return a `Child` handle wrapping the child PID

That `Child` owns the PID. When you call `.wait()`, it calls `waitpid`. When you drop a `Child` without calling `.wait()` or `.kill()`, the child keeps running and eventually becomes a zombie until PID 1 reaps it. Rust does not auto-reap for you.

```rust
use std::process::Command;

let mut child = Command::new("sleep")
    .arg("1")
    .spawn()
    .expect("failed to spawn");

let status = child.wait().expect("wait failed");
println!("sleep exited with {status}");
```

Under `strace -f` this is a `clone3`, then an `execve("/usr/bin/sleep", ...)`, then the parent blocks in `wait4`. Exactly the pattern every Unix shell has used for 50 years.

## Why this matters for servers

Every production server you write on Linux runs inside this model. A few concrete consequences:

- **Accept SIGTERM properly.** If your server does not, orchestrators send SIGKILL after a grace period and you drop in-flight work. Catch it, stop accepting new requests, drain, exit 0.
- **Reap your children.** If you spawn subprocesses (shelling out, running sandboxes, calling ffmpeg), always `wait()` or hand them off to an explicit reaper. A long-running server that leaks zombies will eventually fail to fork.
- **PID 1 is special inside containers.** In a container your process is PID 1. PID 1 has no default signal handlers - if you do not explicitly handle SIGTERM, the kernel drops it. This is why Docker images often use `tini` as PID 1: it reaps zombies and forwards signals.
- **fork in multi-threaded processes is dangerous.** Only the calling thread survives in the child. Any mutex held by another thread is now held forever. Use `posix_spawn` or fork-then-immediate-exec, never fork-then-do-work in a multithreaded program.
- **File descriptors leak across exec by default.** If you open a socket, fork, and exec something untrusted, that process inherits the socket. Set `FD_CLOEXEC` (or use `O_CLOEXEC` on open) on anything you do not want to leak. Rust does this by default on most fds.

None of this is new. Most of it predates Linux itself, going back to [Research Unix v6](https://www.bell-labs.com/usr/dmr/www/1stEdman.html) in the 1970s. But it is still the substrate under every systemd unit, every Docker container, every Kubernetes pod, every `cargo run` you type. The better you understand it the less of your code is magic.
