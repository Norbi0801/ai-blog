+++
title = "Systemd for developers - writing service files"
date = 2026-02-20
description = "Run your Rust binary as a proper Linux service with systemd - unit files, journal logs, socket activation, timers, dependency ordering, and security hardening."

[taxonomies]
tags = ["rust", "devops", "linux", "systemd"]
+++

You built the binary. You handled [graceful shutdown](/blog/graceful-shutdown-in-async-rust-handling-sigterm-properly). You stuffed it into a [tiny Docker image](/blog/docker-multi-stage-builds-for-rust-from-2gb-to-20mb). But maybe you're not running Kubernetes. Maybe you rented a Hetzner box for four euros a month, rsync'd your binary over, and now you're staring at a terminal wondering how to keep it running after you close your SSH session.

`nohup ./myapp &` is not the answer. Neither is `screen` or `tmux`. Your binary needs a process manager - something that starts it on boot, restarts it when it crashes, captures its logs, and lets you control it with a single command. On any modern Linux distribution, that process manager is systemd.

This isn't a systemd deep dive for sysadmins. This is the minimum you need as a developer to ship a Rust service on a Linux box and sleep at night.

<!-- more -->

## The unit file

Systemd manages processes through unit files - plain text config files that describe what to run and how. For a long-running service, you write a `.service` file. Drop it into `/etc/systemd/system/` and systemd picks it up.

Here's a minimal unit file for a Rust web service:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Rust web service
After=network.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/myapp
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Three sections. Each one has a specific job.

**[Unit]** - metadata and ordering. `Description` is what shows up in `systemctl status`. `After=network.target` means "don't start this until the network stack is up." This is ordering only - it doesn't express a hard dependency (more on that later).

**[Service]** - the actual process configuration. This is where you spend most of your time.

**[Install]** - controls what happens when you `systemctl enable` the service. `WantedBy=multi-user.target` means "start this service when the system reaches multi-user mode" - which is normal boot on any server without a graphical desktop.

## The [Service] section in detail

### Type

`Type=simple` is the default and the right choice for most Rust services. It means: the process started by `ExecStart` *is* the service. systemd considers the service "started" as soon as the process is forked.

Other types exist but you rarely need them:

- `Type=exec` - like `simple`, but systemd waits until the binary is actually executing (not just forked). Slightly more correct but the practical difference is negligible.
- `Type=notify` - the service tells systemd when it's ready by calling [`sd_notify(3)`](https://www.freedesktop.org/software/systemd/man/latest/sd_notify.html). Useful if your app has a long startup phase (loading a large model, running migrations) and you want systemd to know when it's actually accepting traffic. The [`sd-notify`](https://crates.io/crates/sd-notify) crate provides this from Rust.
- `Type=forking` - for old-school daemons that fork and exit the parent. You shouldn't write new services this way.

For a typical Axum/Actix/Warp service that binds a port in `main()`, `Type=simple` works fine.

### ExecStart

The path to your binary. Must be an absolute path - no relative paths, no `~`, no shell expansion. If your binary needs arguments:

```ini
ExecStart=/opt/myapp/myapp --config /etc/myapp/config.toml --port 3000
```

One `ExecStart` per service. If you need to run setup commands before the main process, use `ExecStartPre`:

```ini
ExecStartPre=/opt/myapp/migrate
ExecStart=/opt/myapp/myapp
```

`ExecStartPre` commands must exit successfully (exit code 0) or the service won't start. Prefix with `-` to ignore failures: `ExecStartPre=-/usr/bin/mkdir -p /var/cache/myapp`.

### User and Group

Never run your service as root. Create a dedicated system user with no login shell and no home directory:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin myapp
```

Then set `User=myapp` and `Group=myapp` in the unit file. The process drops privileges before executing your binary. If your app needs to bind to port 80 or 443, don't run as root - use a reverse proxy (Caddy, nginx) in front of it, or grant the `CAP_NET_BIND_SERVICE` capability:

```ini
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

### Environment variables

Three ways to pass environment variables:

**Inline** - for a few variables:

```ini
Environment=RUST_LOG=info
Environment=DATABASE_URL=postgres://localhost/myapp
```

**File** - for secrets or many variables:

```ini
EnvironmentFile=/etc/myapp/env
```

Where `/etc/myapp/env` is a plain `KEY=VALUE` file:

```
RUST_LOG=info
DATABASE_URL=postgres://localhost/myapp
JWT_SECRET=your-secret-here
```

Set permissions to `640` owned by root so the service user can read it but other users can't:

```bash
sudo chmod 640 /etc/myapp/env
sudo chown root:myapp /etc/myapp/env
```

**From credentials** - systemd 250+ supports `LoadCredential` and `SetCredential` for passing secrets without environment variables. The credential is available as a file at runtime under `$CREDENTIALS_DIRECTORY`. This is more secure than environment variables (which leak into `/proc/<pid>/environ`) but requires your app to read credentials from files instead of env vars.

### Restart policy

`Restart=on-failure` restarts the service when it exits with a non-zero exit code, gets killed by a signal (like SIGSEGV), or hits a watchdog timeout. It does *not* restart on clean exit (exit code 0) or when you run `systemctl stop`.

Other options:

| Value | Restarts on crash | Restarts on clean exit | Restarts on `systemctl stop` |
|-------|:-:|:-:|:-:|
| `no` | - | - | - |
| `on-failure` | yes | - | - |
| `on-abnormal` | yes (signals only) | - | - |
| `always` | yes | yes | yes |

`on-failure` is the sane default. Use `always` for services that should never be down - but be careful, it'll restart even after `systemctl stop` (which is confusing).

`RestartSec=5` adds a 5-second delay between restarts. Without it, a service that crashes on startup goes into a tight restart loop and can saturate your CPU. If you want exponential backoff, systemd doesn't have it natively, but you can combine `RestartSec` with the rate limiter:

```ini
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=300
StartLimitBurst=5
```

This says: "if the service starts more than 5 times within 300 seconds, stop trying and mark it as failed." The `StartLimit*` directives go in the `[Unit]` section, not `[Service]`.

## Start, stop, status

Enable the service (start on boot) and start it now:

```bash
sudo systemctl enable --now myapp.service
```

The `--now` flag combines `enable` (symlink for boot) with `start` (run it right now). Without `--now`, enabling only sets up the boot symlink - the service won't start until next reboot or manual `systemctl start`.

After you edit the unit file, reload systemd's view of it:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp.service
```

`daemon-reload` is necessary after any change to the unit file. Forgetting this is a classic mistake - you edit the file, restart the service, and wonder why your changes aren't taking effect.

Check what's happening:

```bash
sudo systemctl status myapp.service
```

This shows you the service state, PID, memory usage, CPU time, and the last few log lines. The output looks like:

```
● myapp.service - My Rust web service
     Loaded: loaded (/etc/systemd/system/myapp.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-07-14 10:30:00 UTC; 2h ago
   Main PID: 12345 (myapp)
      Tasks: 8 (limit: 4567)
     Memory: 24.0M
        CPU: 1.234s
     CGroup: /system.slice/myapp.service
             └─12345 /opt/myapp/myapp

Jul 14 10:30:00 srv01 myapp[12345]: listening on 0.0.0.0:3000
```

## Journal logs

Systemd captures everything your process writes to stdout and stderr and stores it in the [journal](https://www.freedesktop.org/software/systemd/man/latest/systemd-journald.service.html) - a structured, indexed, binary log. No more piping to files, no more logrotate configs. (As of systemd v259, the journal is persistent by default - earlier versions stored logs only in memory and lost them on reboot unless you explicitly configured persistence.)

View logs for your service:

```bash
# Last 100 lines
journalctl -u myapp.service -n 100

# Follow (like tail -f)
journalctl -u myapp.service -f

# Since last boot
journalctl -u myapp.service -b

# Time range
journalctl -u myapp.service --since "2026-07-14 10:00" --until "2026-07-14 11:00"

# Only errors (priority 3 = err)
journalctl -u myapp.service -p err
```

The `-u` flag filters by unit name. Without it, you get *all* system logs.

### JSON output

```bash
journalctl -u myapp.service -o json-pretty -n 5
```

This dumps each log entry as a JSON object with all the metadata - timestamp, PID, hostname, priority level, systemd unit, and the message. Pipe it to `jq` for filtering:

```bash
journalctl -u myapp.service -o json | jq 'select(.PRIORITY == "3")'
```

### Sending structured logs from Rust

If you just use `println!` or `tracing` with the default formatter, your logs land in the journal as plain text - they work, but you lose structured fields. The [`tracing-journald`](https://crates.io/crates/tracing-journald) crate sends tracing spans and events directly to journald's native socket with full structured metadata:

```rust
use tracing_subscriber::layer::SubscriberExt;
use tracing_subscriber::util::SubscriberInitExt;

fn init_logging() {
    let journald = tracing_journald::layer()
        .expect("failed to connect to journald");

    tracing_subscriber::registry()
        .with(journald)
        .with(tracing_subscriber::filter::EnvFilter::from_default_env())
        .init();
}
```

Now when you log:

```rust
tracing::info!(
    user_id = "u_12345",
    request_id = %req_id,
    latency_ms = elapsed.as_millis() as u64,
    "request completed"
);
```

The `user_id`, `request_id`, and `latency_ms` fields are stored as native journal fields, not embedded in the message string. You can query them:

```bash
journalctl -u myapp.service USER_ID=u_12345 -o json-pretty
```

This is the journal's killer feature over plain text logs. You get structured querying without a separate log aggregation stack.

For development, you probably still want human-readable terminal output. Stack both layers:

```rust
fn init_logging() {
    let registry = tracing_subscriber::registry()
        .with(tracing_subscriber::filter::EnvFilter::from_default_env());

    if tracing_journald::layer().is_ok() {
        // Running under systemd - use journald
        let journald = tracing_journald::layer().unwrap();
        registry.with(journald).init();
    } else {
        // Development - use pretty terminal output
        let fmt = tracing_subscriber::fmt::layer()
            .with_target(true)
            .with_thread_ids(true);
        registry.with(fmt).init();
    }
}
```

`tracing_journald::layer()` connects to `/run/systemd/journal/socket`. If that socket doesn't exist (you're on macOS, or in a container without journald), it returns an error and you fall back to terminal output.

## Dependencies - After, Wants, Requires

When your service needs another service to be running first (a database, a message queue, a sidecar), you express that with dependency directives.

**`After=`** - pure ordering. "Start my service after this other service." Doesn't start the dependency - just ensures order if both are being started.

**`Wants=`** - weak dependency. "Try to start this other service when my service starts, but don't fail if it can't start."

**`Requires=`** - strong dependency. "This other service must be running. If it stops, stop me too."

In practice, you almost always use `After` + `Wants` together:

```ini
[Unit]
Description=My Rust web service
After=network.target postgresql.service
Wants=postgresql.service
```

This says: "start PostgreSQL when I start, wait for it to come up, then start me. If PostgreSQL can't start, start me anyway" (the `Wants` behavior). If you want a hard failure, use `Requires` instead of `Wants` - but that's aggressive. Your app should handle a missing database gracefully (return 503, retry) rather than refusing to start.

`After=network.target` is the most common ordering dependency. It waits for the network stack to be configured. If you need DNS resolution to work too (e.g., your app connects to `db.internal.example.com` on startup), use `After=network-online.target` and `Wants=network-online.target` instead - `network.target` only means the interfaces are up, not that they have routes and DNS.

## Socket activation

Socket activation is one of systemd's most useful and least understood features. The idea: systemd opens the listening socket *before* your service starts and passes the file descriptor to your process. This gives you three things:

1. **Zero-downtime restarts.** The socket stays open while your process restarts. Incoming connections queue in the kernel's TCP backlog. No dropped connections.
2. **On-demand startup.** The service only starts when the first connection arrives. Great for low-traffic services that shouldn't waste memory sitting idle.
3. **Privilege separation.** systemd opens the socket as root (binding to port 80/443), then starts your service as an unprivileged user that inherits the file descriptor.

You need two unit files - a `.socket` and a `.service`:

```ini
# /etc/systemd/system/myapp.socket
[Unit]
Description=My Rust web service socket

[Socket]
ListenStream=0.0.0.0:3000
NoDelay=true

[Install]
WantedBy=sockets.target
```

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Rust web service
Requires=myapp.socket

[Service]
Type=simple
User=myapp
Group=myapp
ExecStart=/opt/myapp/myapp
# Don't open a socket yourself - use the one from systemd
NonBlocking=true
```

systemd binds `0.0.0.0:3000` and passes the file descriptor as fd 3 (the first fd after stdin/stdout/stderr). The matching is by name: `myapp.socket` automatically activates `myapp.service`.

On the Rust side, the [`listenfd`](https://crates.io/crates/listenfd) crate by Armin Ronacher handles the fd handshake. It reads the `LISTEN_FDS` and `LISTEN_PID` environment variables that systemd sets:

```rust
use listenfd::ListenFd;
use tokio::net::TcpListener;
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "ok" }));

    // Try to get a socket from systemd first
    let mut listenfd = ListenFd::from_env();
    let listener = match listenfd.take_tcp_listener(0).unwrap() {
        Some(std_listener) => {
            std_listener.set_nonblocking(true).unwrap();
            TcpListener::from_std(std_listener).unwrap()
        }
        None => {
            // No systemd socket - bind our own (development mode)
            TcpListener::bind("0.0.0.0:3000").await.unwrap()
        }
    };

    tracing::info!(
        addr = %listener.local_addr().unwrap(),
        "listening"
    );

    axum::serve(listener, app).await.unwrap();
}
```

The fallback to `TcpListener::bind` means the same binary works both under systemd (production) and standalone (development). No feature flags, no conditional compilation.

Enable the socket, not the service:

```bash
sudo systemctl enable --now myapp.socket
```

The service starts automatically on the first incoming connection. You can verify with `systemctl status myapp.socket` - it will show "listening" even before the service process exists.

For development without systemd, the companion tool [`systemfd`](https://github.com/mitsuhiko/systemfd) emulates the fd-passing protocol:

```bash
cargo install systemfd
systemfd --no-pid -s http::3000 -- cargo watch -x run
```

This opens port 3000, passes the fd to your app, and `cargo watch` restarts it on source changes. The socket stays open across restarts - no "address already in use" errors during development.

## Timers - cron replacement

If you need scheduled tasks (database cleanup, report generation, cache warming), systemd timers replace cron with better logging, dependency management, and error handling.

A timer is two files: a `.timer` that defines the schedule and a `.service` that defines what to run.

```ini
# /etc/systemd/system/myapp-cleanup.timer
[Unit]
Description=Run database cleanup every hour

[Timer]
OnCalendar=hourly
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/myapp-cleanup.service
[Unit]
Description=Database cleanup job

[Service]
Type=oneshot
User=myapp
ExecStart=/opt/myapp/myapp --cleanup
Environment=RUST_LOG=info
```

`Type=oneshot` means the service runs the command and exits. systemd considers it "active" while the command runs and "inactive" after it completes.

`Persistent=true` means: if the system was off when the timer should have fired, run the job immediately on next boot. Without this, missed runs are silently skipped.

`RandomizedDelaySec=300` adds up to 5 minutes of random delay. This prevents the thundering herd problem when you have 50 servers all running the same hourly job at exactly `:00`.

Calendar expressions follow a `DayOfWeek Year-Month-Day Hour:Minute:Second` format:

```ini
OnCalendar=hourly                     # every hour at :00
OnCalendar=daily                      # midnight
OnCalendar=Mon *-*-* 06:00:00        # every Monday at 6 AM
OnCalendar=*-*-* *:00/15:00          # every 15 minutes
OnCalendar=*-*-01 00:00:00           # first day of every month
```

Test your expression with `systemd-analyze calendar`:

```bash
$ systemd-analyze calendar "*-*-* *:00/15:00"
  Original form: *-*-* *:00/15:00
Normalized form: *-*-* *:00/15:00
    Next elapse: Mon 2026-07-14 10:45:00 UTC
       (in UTC): Mon 2026-07-14 10:45:00 UTC
       From now: 12min left
```

Enable and check:

```bash
sudo systemctl enable --now myapp-cleanup.timer
systemctl list-timers --all | grep myapp
```

Why timers over cron? Cron gives you a time, a command, and an email to root if it fails. Timers give you: journal integration (all logs captured), dependency ordering (run after another service), resource limits (same sandboxing as services), status tracking (`systemctl status myapp-cleanup.service` shows last run result), and `Persistent=true` for missed runs.

## Hardening

Here's where systemd goes from "process babysitter" to "security sandbox." A single `[Service]` section can lock down your process tighter than most container runtimes. Every option below is a line in your unit file - zero code changes, zero performance cost.

Start with this block and remove what breaks your app:

```ini
[Service]
# ... your ExecStart, User, etc ...

# Filesystem
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/myapp
PrivateTmp=yes

# Kernel
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectControlGroups=yes
ProtectClock=yes
ProtectHostname=yes

# Privilege
NoNewPrivileges=yes
PrivateDevices=yes
RestrictSUIDSGID=yes
LockPersonality=yes

# Network
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
# System calls
SystemCallArchitectures=native
SystemCallFilter=@system-service
MemoryDenyWriteExecute=yes

# Misc
RestrictNamespaces=yes
RestrictRealtime=yes
```

What each directive does:

**`ProtectSystem=strict`** mounts the entire filesystem hierarchy read-only. Your service can only write to paths listed in `ReadWritePaths`. This is the single most impactful hardening option. If your app only writes to `/var/lib/myapp` and `/tmp`, declare that explicitly and everything else becomes immutable.

**`ProtectHome=yes`** makes `/home`, `/root`, and `/run/user` inaccessible. A web service has no business reading user home directories.

**`NoNewPrivileges=yes`** prevents the process (and any child process) from gaining new privileges through `setuid`/`setgid` binaries or filesystem capabilities. Once privileges are dropped, they stay dropped.

**`PrivateDevices=yes`** creates a private `/dev` with only pseudo-devices (`/dev/null`, `/dev/zero`, `/dev/random`). No access to real hardware.

**`PrivateTmp=yes`** gives the service its own `/tmp` and `/var/tmp`, invisible to other processes. Prevents temp file attacks.

**`SystemCallFilter=@system-service`** whitelists a predefined set of syscalls that normal services need and blocks everything else. If an attacker gets code execution in your process, they can't call `mount`, `reboot`, `kexec_load`, or other dangerous syscalls.

**`MemoryDenyWriteExecute=yes`** prevents creating memory mappings that are both writable and executable. This blocks most runtime code generation exploits. Rust binaries don't need W+X memory under normal operation.

**`RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX`** limits which socket types the process can create. A web service needs TCP/UDP (AF_INET, AF_INET6) and maybe Unix sockets (AF_UNIX). It doesn't need Bluetooth (AF_BLUETOOTH), Netlink (AF_NETLINK), or raw packets (AF_PACKET).

### Measuring your hardening score

`systemd-analyze security` scores your unit file from 0 (fully locked down) to 10 (completely exposed):

```bash
$ systemd-analyze security myapp.service
  NAME                                  DESCRIPTION                EXPOSURE
...
  NoNewPrivileges=                      Service process can acquire OK
                                        new privileges
  ProtectSystem=                        Service has strict read-only OK
                                        access to the OS file hierarchy
...

-> Overall exposure level for myapp.service: 2.1 SAFE
```

An unhardened service scores around 9.5. With the block above, you'll be around 2.0. Not every option works for every service - some apps need to write to `/etc`, some need raw sockets, some need to load kernel modules. Start strict, run `systemd-analyze security`, and relax only what you must.

## Complete template for a Rust web service

Here's the unit file I use as a starting point. It covers everything from this post - proper user isolation, environment file, restart policy, journal logging, and the full hardening block:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Rust web service
After=network-online.target postgresql.service
Wants=network-online.target postgresql.service
StartLimitIntervalSec=300
StartLimitBurst=5

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp

# Binary and environment
ExecStart=/opt/myapp/myapp
EnvironmentFile=/etc/myapp/env
Environment=RUST_BACKTRACE=1

# Restart policy
Restart=on-failure
RestartSec=5
TimeoutStopSec=30

# Hardening
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/myapp
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectControlGroups=yes
ProtectClock=yes
ProtectHostname=yes
NoNewPrivileges=yes
RestrictSUIDSGID=yes
LockPersonality=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
RestrictNamespaces=yes
RestrictRealtime=yes
SystemCallArchitectures=native
SystemCallFilter=@system-service
MemoryDenyWriteExecute=yes

# Resource limits
LimitNOFILE=65536
MemoryMax=512M
CPUQuota=200%

[Install]
WantedBy=multi-user.target
```

`TimeoutStopSec=30` is how long systemd waits after sending SIGTERM before sending SIGKILL. If you followed the [graceful shutdown post](/blog/graceful-shutdown-in-async-rust-handling-sigterm-properly), you set your internal shutdown timeout to 25 seconds - 5 seconds less than this value, giving your app time to finish before the SIGKILL hammer drops.

`LimitNOFILE=65536` raises the file descriptor limit. The default (1024) is too low for a web service handling concurrent connections. Each TCP connection is a file descriptor.

`MemoryMax=512M` kills the process if it exceeds 512 MB of RAM. This prevents a memory leak from taking down the whole server. Set it to something reasonable for your workload.

`CPUQuota=200%` limits the service to 2 CPU cores on a multi-core machine. Prevents a runaway computation from starving other services.

## Deployment workflow

A complete deploy script for a single binary on a VPS:

```bash
#!/bin/bash
set -euo pipefail

HOST="srv01.example.com"
BINARY="target/x86_64-unknown-linux-musl/release/myapp"
REMOTE_DIR="/opt/myapp"
SERVICE="myapp.service"

# Build (locally or in CI)
cargo build --release --target x86_64-unknown-linux-musl

# Upload the binary
scp "$BINARY" "$HOST:$REMOTE_DIR/myapp.new"

# Atomic swap and restart
ssh "$HOST" "
    sudo mv $REMOTE_DIR/myapp.new $REMOTE_DIR/myapp
    sudo chmod 755 $REMOTE_DIR/myapp
    sudo systemctl restart $SERVICE
    sleep 2
    sudo systemctl is-active $SERVICE
"
```

The `mv` is atomic on the same filesystem - the old binary keeps running until `systemctl restart` sends SIGTERM. After restart, `is-active` checks that the new version came up successfully. If it didn't, the last restart's logs are in `journalctl -u myapp.service -n 50`.

For zero-downtime deploys without a load balancer, combine this with socket activation. The socket stays open during the restart window, queuing incoming connections. Your users see a brief increase in latency (while the new process starts up), not connection errors.

## Debugging service failures

When your service won't start, the diagnosis sequence is:

```bash
# What happened?
sudo systemctl status myapp.service

# Full logs from last attempt
journalctl -u myapp.service -n 100 --no-pager

# Did it exit? What was the code?
systemctl show myapp.service -p ExecMainStatus,ExecMainCode

# Is the unit file valid?
systemd-analyze verify myapp.service

# Dependency chain
systemctl list-dependencies myapp.service
```

Common failures:

**Exit code 203** - `ExecStart` path doesn't exist or isn't executable. Check the path, check `chmod +x`.

**Exit code 217** - namespace setup failed. Usually means a hardening option is incompatible with your kernel version. Try removing `ProtectSystem`, `PrivateDevices`, etc. one at a time.

**Exit code 200** - namespace setup failed due to permission errors. If using `User=`, some hardening options need `AmbientCapabilities` or must be relaxed.

**"Address already in use"** - another process is holding the port. Check with `ss -tlnp | grep 3000`. If you're using socket activation, make sure the service isn't *also* trying to bind the port itself.

Systemd is the interface between your binary and the operating system. Writing a proper unit file takes ten minutes and pays off every time your service crashes at 3 AM, restarts itself, and you find out from the journal the next morning instead of from an angry user.
