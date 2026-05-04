+++
title = "Writing a shell in Rust"
date = 2025-05-16
description = "Building a minimal Unix shell from scratch - fork, exec, pipes, redirects, signals - to understand what bash actually does under the hood."

[taxonomies]
tags = ["rust", "systems-programming", "linux", "unix"]
+++

Every time you type `ls -la | grep foo > out.txt` into your terminal, a surprising amount of machinery kicks into gear. Your shell parses the input, creates pipes, forks multiple child processes, wires up file descriptors, calls exec to replace each child with the right program, and waits for everything to finish. Most developers use a shell every day without thinking about any of this. Writing one from scratch is the fastest way to change that.

We're going to build a working Unix shell in about 300 lines of Rust. It will handle simple commands, builtins (`cd`, `exit`, `export`), pipes (`|`), I/O redirects (`>`, `<`, `>>`), background processes (`&`), and signal handling (Ctrl+C won't kill the shell). It won't be bash, but by the end you'll understand what bash does and why.

<!-- more -->

## What actually happens when you run a command

Before writing code, let's trace what bash does when you type `ls -la`:

1. **Read** - `read()` syscall to get bytes from stdin
2. **Parse** - split the line into a program name and arguments
3. **Fork** - `fork()` syscall creates a child process that's a copy of the shell
4. **Exec** - the child calls `execvp("ls", ["ls", "-la"])`, which replaces its memory with the `ls` binary
5. **Wait** - the parent calls `waitpid()` to block until the child exits
6. **Loop** - print the prompt again

This fork-then-exec pattern is how Unix has worked since the 1970s. The shell never "runs" `ls` directly - it clones itself, then the clone becomes `ls`. That's fundamental.

You can verify this yourself with `strace`:

```
$ strace -f -e trace=clone,execve,wait4 bash -c "ls"
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|...) = 31337
[pid 31337] execve("/usr/bin/ls", ["ls"], ...) = 0
wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], ...) = 31337
```

Three syscalls. That's the whole story.

## Project setup

```toml
# Cargo.toml
[package]
name = "rush"
version = "0.1.0"
edition = "2021"

[dependencies]
nix = { version = "0.29", features = ["process", "signal", "fs"] }
```

The [`nix`](https://crates.io/crates/nix) crate provides safe Rust wrappers around POSIX syscalls. We need `process` for `fork`/`exec`/`waitpid`, `signal` for signal handling, and `fs` for file operations. We could use raw `libc` calls, but nix gives us proper Rust enums and error handling without hiding what's happening underneath.

## Data structures

A shell command can be simple (`ls -la`) or complex (`cat file | sort | uniq -c > counts.txt`). We need types that capture this:

```rust
use std::ffi::CString;

#[derive(Debug)]
struct Command {
    program: CString,
    args: Vec<CString>,
    stdin_redirect: Option<String>,   // < file
    stdout_redirect: Option<Redirect>, // > or >>
    background: bool,
}

#[derive(Debug)]
enum Redirect {
    Overwrite(String), // > file
    Append(String),    // >> file
}

type Pipeline = Vec<Command>;
```

Why `CString`? Because `execvp` is a C function underneath. It expects null-terminated strings. Rust's `String` and `&str` are not null-terminated. `CString` adds the `\0` byte and guarantees no interior null bytes. If you forget this, you'll get a confusing runtime error when exec silently truncates your arguments.

The `Pipeline` is just a `Vec<Command>` because `ls | grep foo | wc -l` is three commands chained together. Each `|` creates a new command in the vector.

## Parsing input

Our parser needs to handle:
- Simple commands: `ls -la`
- Pipes: `ls | grep foo`
- Redirects: `echo hello > file.txt`, `cat < input.txt`, `echo hello >> file.txt`
- Background: `sleep 10 &`

This isn't a full shell grammar (no quotes, no globbing, no variables), but it covers the core mechanics. Here's the tokenizer and parser:

```rust
fn parse_line(line: &str) -> Vec<Pipeline> {
    let line = line.trim();
    if line.is_empty() {
        return vec![];
    }

    let mut pipelines = vec![];
    let mut current_pipeline: Pipeline = vec![];
    let mut tokens: Vec<String> = Vec::new();
    let mut stdin_redir: Option<String> = None;
    let mut stdout_redir: Option<Redirect> = None;
    let mut background = false;

    let parts: Vec<&str> = line.split_whitespace().collect();
    let mut i = 0;

    while i < parts.len() {
        match parts[i] {
            "|" => {
                if !tokens.is_empty() {
                    current_pipeline.push(build_command(
                        &tokens, stdin_redir.take(), stdout_redir.take(), false,
                    ));
                    tokens.clear();
                }
            }
            ">" => {
                if i + 1 < parts.len() {
                    stdout_redir = Some(Redirect::Overwrite(parts[i + 1].to_string()));
                    i += 1;
                }
            }
            ">>" => {
                if i + 1 < parts.len() {
                    stdout_redir = Some(Redirect::Append(parts[i + 1].to_string()));
                    i += 1;
                }
            }
            "<" => {
                if i + 1 < parts.len() {
                    stdin_redir = Some(parts[i + 1].to_string());
                    i += 1;
                }
            }
            "&" => {
                background = true;
            }
            token => {
                tokens.push(token.to_string());
            }
        }
        i += 1;
    }

    if !tokens.is_empty() {
        current_pipeline.push(build_command(
            &tokens, stdin_redir.take(), stdout_redir.take(), background,
        ));
    }
    if !current_pipeline.is_empty() {
        pipelines.push(current_pipeline);
    }

    pipelines
}

fn build_command(
    tokens: &[String],
    stdin_redirect: Option<String>,
    stdout_redirect: Option<Redirect>,
    background: bool,
) -> Command {
    let program = CString::new(tokens[0].as_str()).expect("invalid program name");
    let args: Vec<CString> = tokens
        .iter()
        .map(|t| CString::new(t.as_str()).expect("invalid argument"))
        .collect();
    Command {
        program,
        args,
        stdin_redirect,
        stdout_redirect,
        background,
    }
}
```

We split on whitespace and handle operators as special tokens. This is much simpler than what bash does - bash has a full lexer that handles single quotes, double quotes, escape sequences, heredocs, variable substitution, brace expansion, and a dozen other things. Our shell handles the structural operators and nothing else.

One important design choice: the first element in `args` is the program name itself. That matches the Unix convention - `argv[0]` is the program name. When you run `ls -la`, `execvp` receives `["ls", "-la"]`, not just `["-la"]`. Programs use `argv[0]` to determine their own name, which is why `busybox` can behave as different programs depending on what symlink you call it through.

## The REPL

The read-eval-print loop is the shell's heartbeat:

```rust
use std::io::{self, Write, BufRead};

fn main() {
    setup_signals();

    let stdin = io::stdin();
    loop {
        print!("rush> ");
        io::stdout().flush().unwrap();

        let mut line = String::new();
        match stdin.lock().read_line(&mut line) {
            Ok(0) => break,      // EOF (Ctrl+D)
            Ok(_) => {}
            Err(_) => continue,  // interrupted by signal
        }

        let pipelines = parse_line(&line);
        for pipeline in pipelines {
            if pipeline.len() == 1 && !try_builtin(&pipeline[0]) {
                execute_single(&pipeline[0]);
            } else if pipeline.len() > 1 {
                execute_pipeline(&pipeline);
            }
        }
    }
}
```

`read_line` returning `Ok(0)` means EOF - the user pressed Ctrl+D. The `Err` case handles `EINTR` - when a signal (like Ctrl+C) interrupts the read syscall. We just loop and show the prompt again.

Notice we check for builtins before forking. That matters - we'll get to why in a moment.

## Executing a simple command

Here's the core of any shell - the fork+exec pattern:

```rust
use nix::unistd::{fork, ForkResult, execvp, close};
use nix::sys::wait::{waitpid, WaitPidFlag};
use std::os::unix::io::RawFd;

fn execute_single(cmd: &Command) {
    match unsafe { fork() } {
        Ok(ForkResult::Child) => {
            setup_redirects(cmd);
            let args: Vec<&std::ffi::CStr> = cmd.args.iter().map(|a| a.as_c_str()).collect();
            execvp(&cmd.program, &args).expect("execvp failed");
            // execvp never returns on success - the process image is replaced
        }
        Ok(ForkResult::Parent { child }) => {
            if !cmd.background {
                waitpid(child, None).ok();
            } else {
                eprintln!("[bg] {}", child);
            }
        }
        Err(e) => eprintln!("fork failed: {}", e),
    }
}
```

`fork()` is `unsafe` because after calling it in a multithreaded program, the child process inherits copies of the parent's threads in an undefined state. In our single-threaded shell, this is fine, but the nix crate correctly marks it unsafe to force you to think about it.

After `fork()`, we're in two processes running the same code. The return value tells us which one we are:
- **Child** (`ForkResult::Child`): set up redirects, then call `execvp`. This replaces our entire process memory with the target program. If `execvp` returns, it means the program wasn't found.
- **Parent** (`ForkResult::Parent { child }`): if the command isn't backgrounded, call `waitpid` to block until the child exits.

What `execvp` does under the hood: it searches `$PATH` for the program (that's the `p` in `execvp` - path search), opens the binary, maps it into memory, sets up the stack with `argc`/`argv`/`envp`, and jumps to the entry point. The calling process's memory is entirely replaced. File descriptors stay open (unless marked `O_CLOEXEC`). That last detail is critical for how pipes work.

## Builtins: why cd can't be a subprocess

Some commands must run in the shell process itself. `cd` is the classic example:

```rust
use nix::unistd::chdir;
use std::env;

fn try_builtin(cmd: &Command) -> bool {
    let name = cmd.program.to_str().unwrap_or("");
    match name {
        "cd" => {
            let dir = cmd.args.get(1)
                .map(|a| a.to_str().unwrap_or("/"))
                .unwrap_or_else(|| {
                    env::var("HOME").ok().as_deref().unwrap_or("/")
                });
            if let Err(e) = chdir(dir) {
                eprintln!("cd: {}", e);
            }
            true
        }
        "exit" => std::process::exit(0),
        "export" => {
            if let Some(arg) = cmd.args.get(1) {
                let s = arg.to_str().unwrap_or("");
                if let Some((key, val)) = s.split_once('=') {
                    env::set_var(key, val);
                }
            }
            true
        }
        _ => false,
    }
}
```

Why can't `cd` be a child process? Because each process has its own working directory. If we forked, ran `chdir` in the child, and waited - the child's directory would change, then the child would exit, and the parent (our shell) would still be in the old directory. `chdir` must happen in the shell process itself.

The same logic applies to `export` - environment variables are per-process. A child inherits copies of the parent's environment, but modifying them in the child doesn't affect the parent. So `export FOO=bar` has to call `set_var` in the shell process.

Bash has about 60 builtins. Most exist for the same reason - they need to modify the shell's own state. `alias`, `source`, `set`, `read`, `jobs`, `fg`, `bg` - all builtins. You can check with `type cd` in bash:

```
$ type cd
cd is a shell builtin
```

## I/O redirects

Redirects work by manipulating file descriptors before exec. The child opens a file and uses `dup2` to make it replace stdin (fd 0) or stdout (fd 1):

```rust
use nix::fcntl::{open, OFlag};
use nix::sys::stat::Mode;
use nix::unistd::dup2;

fn setup_redirects(cmd: &Command) {
    if let Some(ref path) = cmd.stdin_redirect {
        let fd = open(
            path.as_str(),
            OFlag::O_RDONLY,
            Mode::empty(),
        ).expect("failed to open input file");
        dup2(fd, 0).expect("dup2 stdin failed");
        close(fd).ok();
    }

    if let Some(ref redir) = cmd.stdout_redirect {
        let (path, flags) = match redir {
            Redirect::Overwrite(p) => {
                (p.as_str(), OFlag::O_WRONLY | OFlag::O_CREAT | OFlag::O_TRUNC)
            }
            Redirect::Append(p) => {
                (p.as_str(), OFlag::O_WRONLY | OFlag::O_CREAT | OFlag::O_APPEND)
            }
        };
        let fd = open(path, flags, Mode::from_bits(0o644).unwrap())
            .expect("failed to open output file");
        dup2(fd, 1).expect("dup2 stdout failed");
        close(fd).ok();
    }
}
```

`dup2(fd, 0)` means "make file descriptor 0 (stdin) point to the same file as `fd`". After this call, when the exec'd program reads from stdin, it reads from our file instead of the terminal. We close the original `fd` afterward because we don't need two descriptors pointing to the same file.

The flags for output redirects determine the behavior:
- `>` uses `O_TRUNC` - truncate the file to zero length first
- `>>` uses `O_APPEND` - seek to the end before each write
- Both use `O_CREAT` - create the file if it doesn't exist

This all happens in the child process between `fork` and `exec`. The parent's file descriptors are unaffected. That's the beauty of the fork+exec model - the child can rearrange its own file descriptors however it wants before becoming a new program.

## Pipes

Pipes are where things get interesting. `ls | grep foo` requires:
1. Create a pipe (a kernel buffer with a read end and a write end)
2. Fork child 1 (`ls`), redirect its stdout to the pipe's write end
3. Fork child 2 (`grep`), redirect its stdin to the pipe's read end
4. Close the pipe ends in the parent
5. Wait for both children

Here's the implementation:

```rust
use nix::unistd::pipe;

fn execute_pipeline(commands: &[Command]) {
    let mut prev_read: Option<RawFd> = None;
    let mut children = Vec::new();

    for (i, cmd) in commands.iter().enumerate() {
        let is_last = i == commands.len() - 1;

        // create a pipe for everything except the last command
        let (pipe_read, pipe_write) = if !is_last {
            let (r, w) = pipe().expect("pipe failed");
            (Some(r), Some(w))
        } else {
            (None, None)
        };

        match unsafe { fork() } {
            Ok(ForkResult::Child) => {
                // if there's a previous pipe, wire it to stdin
                if let Some(pr) = prev_read {
                    dup2(pr, 0).expect("dup2 pipe stdin failed");
                    close(pr).ok();
                }
                // if there's a next pipe, wire stdout to it
                if let Some(pw) = pipe_write {
                    dup2(pw, 1).expect("dup2 pipe stdout failed");
                    close(pw).ok();
                }
                // close the read end of the current pipe in the child
                if let Some(pr) = pipe_read {
                    close(pr).ok();
                }

                setup_redirects(cmd);
                let args: Vec<&std::ffi::CStr> =
                    cmd.args.iter().map(|a| a.as_c_str()).collect();
                execvp(&cmd.program, &args).expect("execvp failed");
            }
            Ok(ForkResult::Parent { child }) => {
                children.push(child);
                // close pipe ends the parent doesn't need
                if let Some(pr) = prev_read {
                    close(pr).ok();
                }
                if let Some(pw) = pipe_write {
                    close(pw).ok();
                }
                prev_read = pipe_read;
            }
            Err(e) => {
                eprintln!("fork failed: {}", e);
                return;
            }
        }
    }

    // close any remaining read end
    if let Some(pr) = prev_read {
        close(pr).ok();
    }

    // wait for all children
    for child in children {
        waitpid(child, None).ok();
    }
}
```

The tricky part is closing file descriptors correctly. Every `pipe()` call creates two file descriptors. After forking, both the parent and child have copies of those descriptors. If you forget to close the write end of a pipe in the parent, the reading child will never see EOF because the kernel thinks someone might still write to the pipe. The reader blocks forever. This is the single most common bug when implementing pipes.

Let's trace through `ls | grep foo | wc -l`:

```
Iteration 0 (ls):
  create pipe A (read=3, write=4)
  fork child 0:
    stdout -> pipe A write (fd 4)
    close pipe A read (fd 3)
    exec "ls"
  parent:
    close pipe A write (fd 4)
    save pipe A read (fd 3) as prev_read

Iteration 1 (grep):
  create pipe B (read=5, write=6)
  fork child 1:
    stdin  -> pipe A read (fd 3, from prev_read)
    stdout -> pipe B write (fd 6)
    close pipe B read (fd 5)
    exec "grep" "foo"
  parent:
    close pipe A read (fd 3)
    close pipe B write (fd 6)
    save pipe B read (fd 5) as prev_read

Iteration 2 (wc):
  no new pipe (last command)
  fork child 2:
    stdin -> pipe B read (fd 5, from prev_read)
    exec "wc" "-l"
  parent:
    close pipe B read (fd 5)
    wait for all three children
```

Each child only keeps the file descriptors it needs. Everything else gets closed. This is how bash does it too - you can confirm with `strace -f bash -c "ls | grep foo | wc -l"` and watch the ballet of `pipe2`, `clone`, `dup2`, `close`, and `execve` syscalls.

## Background processes

Background execution (`sleep 10 &`) is simpler than you might expect - we just skip the `waitpid` call:

```rust
if !cmd.background {
    waitpid(child, None).ok();
} else {
    eprintln!("[bg] {}", child);
}
```

But this creates a problem: zombie processes. When a child exits, the kernel keeps its exit status around until the parent calls `waitpid`. If we never wait, the process entry stays in the kernel's process table forever (you'll see it as `<defunct>` in `ps`).

The fix is to periodically reap finished background processes. We can do this at the top of each REPL iteration:

```rust
use nix::sys::wait::WaitPidFlag;
use nix::unistd::Pid;

fn reap_zombies() {
    loop {
        match waitpid(Pid::from_raw(-1), Some(WaitPidFlag::WNOHANG)) {
            Ok(WaitStatus::Exited(pid, status)) => {
                eprintln!("[bg] {} exited ({})", pid, status);
            }
            Ok(WaitStatus::Signaled(pid, signal, _)) => {
                eprintln!("[bg] {} killed by {}", pid, signal);
            }
            _ => break,
        }
    }
}
```

`WNOHANG` makes `waitpid` return immediately if no child has exited, instead of blocking. `Pid::from_raw(-1)` means "any child process". We loop until there are no more finished children to reap.

A production shell would also handle `SIGCHLD` - the kernel sends this signal to the parent whenever a child changes state. Bash uses `SIGCHLD` to print the `[1]+ Done` messages asynchronously. Our shell takes the simpler approach of checking at each prompt.

## Signal handling

Without signal handling, pressing Ctrl+C sends `SIGINT` to the entire foreground process group - including your shell. The shell dies. That's not what you want.

```rust
use nix::sys::signal::{sigaction, SigAction, SigHandler, SaFlags, SigSet, Signal};

fn setup_signals() {
    let action = SigAction::new(
        SigHandler::SigIgn,
        SaFlags::empty(),
        SigSet::empty(),
    );
    unsafe {
        sigaction(Signal::SIGINT, &action).expect("failed to set SIGINT handler");
        sigaction(Signal::SIGQUIT, &action).expect("failed to set SIGQUIT handler");
        sigaction(Signal::SIGTSTP, &action).expect("failed to set SIGTSTP handler");
    }
}
```

`SigHandler::SigIgn` tells the kernel "ignore this signal". The shell won't die on Ctrl+C. But the child processes need to receive it. This works because `execvp` resets ignored signals to the default handler. So the shell ignores `SIGINT`, forks a child (which inherits the ignore), and the child calls `execvp` which resets `SIGINT` to the default (terminate). Now Ctrl+C kills the child but not the shell.

Wait - that's what happens with `SigDfl` disposition. `SigIgn` is actually *preserved* across exec. So we need one more step: reset signal handling in the child before exec:

```rust
// inside the child, before execvp:
let default = SigAction::new(SigHandler::SigDfl, SaFlags::empty(), SigSet::empty());
unsafe {
    sigaction(Signal::SIGINT, &default).ok();
    sigaction(Signal::SIGQUIT, &default).ok();
    sigaction(Signal::SIGTSTP, &default).ok();
}
```

This ensures Ctrl+C kills `grep` but not the shell. Bash does exactly this - it ignores signals for itself and restores defaults in children.

There's another subtlety here. In a real shell, the foreground process should be in its own process group, and the terminal's foreground process group should be set to match. This is how job control works - `fg`, `bg`, Ctrl+Z. Implementing full job control requires `setpgid`, `tcsetpgrp`, and careful handling of `SIGTTOU`/`SIGTTIN`. That's another 100+ lines that we'll skip, but it's worth knowing that bash [manages process groups](https://www.gnu.org/software/bash/manual/bash.html#Job-Control) to make this work.

## Putting it together

Here's the complete `main` function with zombie reaping:

```rust
fn main() {
    setup_signals();

    let stdin = io::stdin();
    loop {
        reap_zombies();
        print!("rush> ");
        io::stdout().flush().unwrap();

        let mut line = String::new();
        match stdin.lock().read_line(&mut line) {
            Ok(0) => break,
            Ok(_) => {}
            Err(_) => continue,
        }

        let pipelines = parse_line(&line);
        for pipeline in pipelines {
            if pipeline.len() == 1 && !try_builtin(&pipeline[0]) {
                execute_single(&pipeline[0]);
            } else if pipeline.len() > 1 {
                execute_pipeline(&pipeline);
            }
        }
    }
}
```

Build and run:

```
$ cargo build --release
$ ./target/release/rush
rush> echo hello world
hello world
rush> ls -la | grep Cargo
-rw-r--r--  1 user user   178 Apr  3 12:00 Cargo.toml
rush> echo line one > out.txt
rush> echo line two >> out.txt
rush> cat out.txt
line one
line two
rush> sleep 5 &
[bg] 31337
rush> pwd
/home/user/rush
rush> cd /tmp
rush> pwd
/tmp
rush> exit
```

## What we skipped (and what bash does)

Our shell is about 300 lines. Bash is around 140,000 lines. Here's what accounts for the gap:

**Quoting and escaping.** `echo "hello world"` should be one argument, not two. `echo it\'s` should produce `it's`. Bash handles single quotes (literal), double quotes (variable expansion inside), backticks, `$()`, ANSI-C quoting (`$'\n'`), and locale-specific quoting (`$"..."`). Our parser splits on whitespace and calls it a day.

**Variable expansion.** `$HOME`, `${HOME}`, `${HOME:-/default}`, `$?` (last exit code), `$$` (shell PID), `$!` (last background PID), `$@`, `"$@"`, `${var//pattern/replacement}`. Bash's parameter expansion is basically its own language.

**Globbing.** `*.rs` should expand to all Rust files. Bash implements this with `glob()` and handles `?`, `*`, `[...]`, `**` (with `globstar`), and extended globs like `!(pattern)`.

**Job control.** Full `fg`/`bg`/`jobs`/Ctrl+Z support requires process groups, terminal control, and `SIGTSTP`/`SIGCONT` handling. Each pipeline runs in its own process group, and the terminal's foreground group is swapped between the shell and the running pipeline.

**Here-documents and here-strings.** `cat <<EOF` and `grep <<< "string"` - these create temporary files or pipes to feed input to commands.

**Readline.** Line editing, history, tab completion, key bindings. Bash uses GNU Readline, which is itself about 30,000 lines of C. For a Rust shell, you'd reach for [`rustyline`](https://crates.io/crates/rustyline).

## What you learned

The core of any Unix shell is five syscalls: `fork`, `execvp`, `waitpid`, `pipe`, and `dup2`. Everything else is parsing and bookkeeping. Pipes work by creating a kernel buffer and wiring file descriptors across forked processes. Redirects work by replacing stdin/stdout with file descriptors to real files. Builtins exist because some operations have to modify the shell process itself.

The Unix process model - fork a copy of yourself, rearrange file descriptors, exec the target program - might seem roundabout compared to Windows' `CreateProcess`. But it's this separation of "create a process" and "load a program" that makes pipes, redirects, and job control compose so cleanly. Each concern is handled independently. That's the Unix philosophy at the syscall level.

If you want to go further, look at the source code of [`nushell`](https://github.com/nushell/nushell) (a modern shell written in Rust, ~200K lines), or read the [POSIX shell specification](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html) to see just how deep the rabbit hole goes. Or add quoted string support to our parser - that alone will teach you why shell parsing is one of the more cursed problems in computing.
