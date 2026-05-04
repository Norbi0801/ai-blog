+++
title = "AWS Firecracker - how Lambda runs your code in Rust microVMs"
date = 2025-06-04
description = "Inside Firecracker: the Rust-based microVM monitor that boots in 125ms, uses 3MB of memory overhead, and powers every AWS Lambda invocation."

[taxonomies]
tags = ["rust", "aws", "virtualization", "systems-programming"]
+++

When you deploy a Rust function to Lambda - if you followed along with [the previous post](/blog/running-rust-in-aws-lambda---serverless-without-the-cold-sta) - your binary runs inside what AWS calls an "execution environment." But what is that, exactly? Not a container. Not a traditional VM. It's a microVM created by [Firecracker](https://github.com/firecracker-microvm/firecracker), a virtual machine monitor (VMM) written in Rust, open-sourced by AWS in 2018.

Firecracker powers every Lambda invocation and every Fargate task. Trillions of requests per month, hundreds of thousands of customers. And the entire thing is about 50,000 lines of Rust - compared to QEMU's roughly 2 million lines of C.

This post is about what happens below your handler: how Firecracker creates isolated environments, why it's written in Rust, and what design decisions make a 125ms boot time possible.

<!-- more -->

## The problem: containers aren't enough, VMs are too slow

Before Firecracker, AWS Lambda ran customer code in containers. Containers share the host kernel - they use Linux namespaces and cgroups for isolation, but the boundary between your code and the host is a syscall interface, not a hardware boundary. One kernel vulnerability and the isolation breaks.

Traditional VMs (QEMU/KVM) provide real hardware-level isolation. Each guest gets its own kernel, its own virtual hardware. But they're heavy. QEMU emulates hundreds of devices - USB controllers, VGA adapters, floppy drives, sound cards, PCI buses. It takes over a second to boot, uses 130+ MB of memory overhead per VM, and you can maybe run a few hundred on a single host. For a service that needs to spin up thousands of isolated environments per second, that doesn't work.

AWS needed both: the security of a VM and the speed of a container. Firecracker is the answer.

## What Firecracker actually is

Firecracker is a VMM - a user-space process that creates and manages virtual machines using the Linux [KVM](https://www.kernel.org/doc/html/latest/virt/kvm/index.html) (Kernel-based Virtual Machine) hypervisor. Each Firecracker process manages exactly one microVM. Not multiple. One process, one VM.

The project started as a fork of Google's [CrosVM](https://chromium.googlesource.com/crosvm/crosvm/) (the Chrome OS VMM, also written in Rust), then AWS stripped it down to the bare minimum needed for serverless workloads.

The numbers from the official [SPECIFICATION.md](https://github.com/firecracker-microvm/firecracker/blob/main/SPECIFICATION.md) and the [NSDI 2020 paper](https://www.usenix.org/system/files/nsdi20-paper-agache.pdf) by Agache et al.:

| Metric | Firecracker | QEMU |
|--------|------------|------|
| Boot time | ~125ms | ~1,250ms |
| Memory overhead per VM | ~3-5 MB | ~131 MB |
| Codebase size | ~50K lines (Rust) | ~2M lines (C) |
| Device model | ~10 virtio devices | Hundreds (incl. legacy) |
| VM creation rate | 150 VMs/sec/host | Much slower |

A host with 256 GB of RAM can theoretically manage over 50,000 microVMs. AWS has tested up to 20x memory oversubscription without issues in production.

## The three-thread architecture

Every Firecracker process has three types of threads:

**1. API thread** - runs a REST HTTP server over a Unix domain socket. This is your control plane. You configure the VM before boot (kernel image, root filesystem, network interfaces, memory) and control it after boot (pause, resume, snapshot). It uses Firecracker's own `micro_http` library - not hyper, not actix. Just a minimal HTTP parser. This thread never touches the VM's fast path.

**2. VMM thread** - the main event loop. Runs an `EventManager` (epoll-based) that handles device emulation, I/O rate limiting, and the MMDS (MicroVM Metadata Service - think EC2's instance metadata, but for microVMs). Communication between the API thread and the VMM uses Rust's `mpsc` channels with `EventFd` notifications.

**3. vCPU threads** - one per guest CPU core, up to 32. Each thread runs a tight loop that calls [`KVM_RUN`](https://docs.kernel.org/virt/kvm/api.html#kvm-run) ioctl. The guest executes natively on the CPU in VMX non-root mode (Intel) or its AMD equivalent. When the guest does something that needs host intervention (MMIO access, I/O port read, halt), KVM triggers a VM exit and the vCPU thread handles it.

Here's the core of a vCPU thread, simplified from the actual source in `src/vmm/src/vstate/vcpu/mod.rs`:

```rust
// Simplified from Firecracker source
loop {
    match vcpu_fd.run() {
        Ok(VcpuExit::MmioRead(addr, data)) => {
            // Guest read from a device register via MMIO
            mmio_bus.read(addr, data);
        }
        Ok(VcpuExit::MmioWrite(addr, data)) => {
            // Guest wrote to a device register
            mmio_bus.write(addr, data);
        }
        Ok(VcpuExit::Hlt) => {
            break; // Guest halted
        }
        Ok(VcpuExit::Shutdown) => {
            break; // Guest requested shutdown
        }
        // ... other exit reasons
        Err(e) => {
            // Handle KVM errors
        }
    }
}
```

The `vcpu_fd` is a safe Rust wrapper around the KVM file descriptor, from the [`kvm-ioctls`](https://github.com/rust-vmm/kvm-ioctls) crate. No raw ioctl calls, no manual memory management - the rust-vmm ecosystem provides typed, safe interfaces to KVM.

## The minimal device model

This is where Firecracker gets its speed. Traditional VMs emulate a full PC: BIOS, PCI bus, USB, VGA, IDE, SATA, legacy NICs. Firecracker emulates almost nothing:

**What's in (as of v1.15.0):**
- **virtio-net** - network via host TAP interfaces, with per-device rate limiting
- **virtio-block** - file-backed block storage
- **virtio-vsock** - host-guest communication (how Lambda's shim process talks to the outside)
- **virtio-balloon** - memory reclamation, free page reporting
- **virtio-rng** - entropy source
- **virtio-pmem** - persistent memory (added v1.14)
- **virtio-mem** - memory hot-plugging (added v1.14)
- **Serial console (8250 UART)** - guest output
- **i8042 keyboard controller** - x86 shutdown/reset only

**What's deliberately absent:**
No BIOS. No UEFI. No PCI bus. No USB. No VGA. No sound. No floppy. No IDE/SATA. No GPU passthrough. No SPICE/VNC remote display. No TPM. No architecture emulation.

All devices use [VirtIO](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html) over MMIO (memory-mapped I/O) transport - not PCI. This is a para-virtualized model: the guest kernel knows it's in a VM and cooperates with the host through shared memory ring buffers (virtqueues) instead of pretending to talk to real hardware.

Firecracker doesn't use a BIOS at all. It loads the guest kernel directly using the [64-bit Linux Boot Protocol](https://www.kernel.org/doc/html/latest/arch/x86/boot.html). The kernel loading code in `load_kernel()` validates ELF headers, reads program headers, and maps `PT_LOAD` segments into guest memory starting at the protected-mode entry point (0x100000). Boot goes straight from "KVM starts executing" to "Linux init runs."

For devices, KVM provides the interrupt infrastructure (PIC, IOAPIC, PIT, kvm-clock) and Firecracker wires its virtio devices into it via `irqfd` and `ioeventfd`:

```rust
// Simplified device registration flow
fn register_mmio_device(vmm: &mut Vmm, device: impl VirtioDevice) {
    let mmio_slot = allocate_mmio_address();

    // When guest writes to the Queue Notify register (offset 0x050),
    // KVM signals this eventfd directly - no VM exit needed
    let queue_evt = EventFd::new(EFD_NONBLOCK)?;
    vm_fd.register_ioeventfd(&queue_evt, mmio_slot + 0x050)?;

    // When the device wants to interrupt the guest,
    // writing to this fd injects the interrupt via KVM
    let irq_evt = EventFd::new(EFD_NONBLOCK)?;
    vm_fd.register_irqfd(&irq_evt, irq_line)?;

    mmio_bus.insert(device, mmio_slot)?;
}
```

The `ioeventfd` trick is important for performance. Without it, every time the guest notifies a device (writes to a doorbell register), you get a full VM exit - context switch from guest to host, thread wakeup, dispatch, context switch back. With `ioeventfd`, KVM intercepts the write in kernel space and signals an eventfd directly. The host device backend picks it up via epoll. No VM exit, dramatically less overhead.

## Rate limiting

Every virtio-net and virtio-block device in Firecracker has an optional rate limiter. It uses a token bucket algorithm with two buckets per device: one for operations per second, one for bandwidth. You configure it through the API:

```json
{
    "drive_id": "rootfs",
    "path_on_host": "/srv/rootfs.ext4",
    "is_root_device": true,
    "rate_limiter": {
        "bandwidth": {
            "size": 104857600,
            "refill_time": 1000
        },
        "ops": {
            "size": 10000,
            "refill_time": 1000
        }
    }
}
```

This caps the root drive at 100 MB/s and 10,000 IOPS. The rate limiter runs in the VMM thread as part of the epoll event loop - no separate thread, no syscall overhead. When a device exceeds its budget, the rate limiter simply stops processing its virtqueue until tokens refill.

This matters for multi-tenant hosts. Without rate limiting, one noisy VM could saturate the host's disk I/O and starve everyone else.

## Why Rust - not just "memory safety"

The usual pitch is "Rust prevents buffer overflows." That's true, but it undersells the actual reasoning. AWS engineers were building a security-critical hypervisor that sits between untrusted guest code and the host. The VMM parses data structures written by the guest - virtqueue descriptors, MMIO register values, device configuration. Any bug in that parsing is a potential VM escape.

In C, a single off-by-one error in a device emulation path could let a malicious guest read host memory. In QEMU's history, this has happened repeatedly - CVE-2015-3456 (VENOM, floppy controller buffer overflow), CVE-2020-14364 (USB EHCI out-of-bounds), and many others. QEMU's attack surface is enormous because it emulates hundreds of devices in 2 million lines of C.

Firecracker gets three things from Rust:

**1. Memory safety by default.** No buffer overflows, no use-after-free, no double-free. This is the table stakes argument.

**2. Thread safety at compile time.** The VMM thread, vCPU threads, and API thread share state through explicitly typed channels and `Arc<Mutex<T>>`. The compiler rejects data races before you can ship them. In a hypervisor, a race condition between device emulation and vCPU execution isn't just a bug - it's a security boundary violation.

**3. Formal verification with Kani.** AWS runs the [Kani model checker](https://model-checking.github.io/kani-verifier-blog/2023/08/31/using-kani-to-validate-security-boundaries-in-aws-firecracker.html) continuously in Firecracker's CI pipeline. Kani proves properties about Rust code that even the type system can't catch - logical errors, timing-dependent issues, spec conformance. There are 27 Kani harnesses across 3 verification suites.

Kani has found real bugs. A rounding error in the rate limiter that let guests exceed their bandwidth cap by 0.01%. A virtio vulnerability where a malicious guest could place queue components in the MMIO gap and crash the VMM. These aren't the kind of bugs you find with unit tests. They're the kind that only show up when a motivated attacker is probing your device emulation.

If you've read the [unsafe Rust post](/blog/unsafe-rust---what-it-enables-and-when-you-need-it), you know that `unsafe` blocks are where human-verified invariants live. In Firecracker, `unsafe` usage is minimal and concentrated in the KVM interface layer (the `kvm-ioctls` crate handles most of it) and guest memory access (the `vm-memory` crate). The device emulation code - the part that parses untrusted guest input - is safe Rust throughout.

## The security model: jailer + seccomp

Firecracker alone isn't the whole isolation story. The `jailer` binary wraps each Firecracker process in multiple layers of OS-level containment:

**Step 1: Resource isolation.** The jailer creates dedicated cgroups for each VM - CPU affinity via cpuset, CPU quota via the cpu subsystem. Each microVM gets its own resource budget. One VM can't monopolize the host's cores.

**Step 2: Namespace isolation.** New network and PID namespaces. The Firecracker process sees only its own network stack and its own process tree.

**Step 3: Filesystem isolation.** `pivot_root()` into a minimal directory containing only the Firecracker binary, `/dev/kvm`, and `/dev/net/tun`. The VMM can't see the host filesystem.

**Step 4: Privilege drop.** The jailer starts as root (needed to create cgroups and namespaces), then drops to an unprivileged UID/GID before `exec()`-ing into the Firecracker binary. From this point on, the VMM process runs completely unprivileged.

**Step 5: Seccomp filters.** Each thread type (VMM, API, vCPU) gets its own seccomp-BPF filter that allowlists only the syscalls it needs. Firecracker uses about 37 out of Linux's 313+ syscalls. Any disallowed syscall kills the process immediately. The filters are compiled from JSON into BPF bytecode at build time via `seccompiler-bin` and embedded in the binary.

A guest escape would need to break through: the guest kernel boundary, KVM hardware virtualization, seccomp filters, cgroup limits, namespace isolation, the chroot jail, and dropped privileges. Each layer is independent - compromising one doesn't help with the others.

## How Lambda uses Firecracker

The [Lambda worker architecture](https://www.bschaatsbergen.com/behind-the-scenes-lambda) runs on bare-metal [Nitro instances](https://aws.amazon.com/ec2/nitro/) in a separate AWS account. Each worker hosts hundreds to thousands of microVMs. The key component is the **MicroManager** - a process on each worker that manages the lifecycle of all Firecracker processes on that host.

The invocation flow:

1. **Frontend Worker** receives your request, authenticates it, validates the payload.
2. **Worker Manager** checks if a warm sandbox (existing microVM with your function loaded) is available.
3. On a cold start, the **Placement Service** picks a worker with available capacity. This takes about 20ms on average. It optimizes for even CPU distribution across the fleet.
4. **MicroManager** on the chosen worker creates a new Firecracker microVM, loads your function binary, and signals readiness.
5. A **shim process** inside the microVM communicates with MicroManager via TCP/IP over a vsock connection. This shim implements the Lambda Runtime API that your function polls (the `GET /2018-06-01/runtime/invocation/next` loop from the [previous post](/blog/running-rust-in-aws-lambda---serverless-without-the-cold-sta)).

The clever part: MicroManager keeps a **pool of pre-booted microVMs** ready to go. Even though Firecracker boots in 125ms, at Lambda's scale - trillions of requests per month - that 125ms matters in the tail. Pre-booted VMs just need a function binary loaded and can start accepting invocations immediately.

Each microVM provides a single slot for one concurrent invocation. If your function gets 100 concurrent requests, Lambda creates 100 microVMs. But within a single slot, invocations are serial - your function handles one event, returns, and waits for the next. This is why the handler in `lambda_runtime` is a sequential loop, not a thread-per-request model.

Workers operate on a 14-hour lease. After the lease expires, the worker is drained and terminated. Lambda intentionally concentrates load on the smallest possible set of busy workers (statistical multiplexing) rather than spreading thin across many workers. This improves resource utilization and cache efficiency.

## Snapshots and SnapStart

Firecracker can snapshot a running microVM and restore it later. A snapshot has three parts: guest memory (binary RAM dump), microVM state (serialized hardware and KVM state), and disk files (user-managed).

The restore mechanism is fast - [Marc Brooker](https://brooker.co.za/blog/2022/11/29/snapstart.html) (AWS principal engineer) reports about 4-10ms for a full Linux system restore. The memory file is mapped with `MAP_PRIVATE` and pages are faulted in on demand. Only the pages your function actually touches get loaded. Pages not accessed stay on disk. Writes create copy-on-write anonymous mappings.

This is the foundation of [Lambda SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html). For Java functions, Lambda takes a Firecracker snapshot after the JVM and your application have fully initialized - classes loaded, dependency injection wired, connection pools created. On the next cold start, instead of repeating all that initialization, Lambda restores the snapshot. Your function resumes from exactly where it was, mid-execution.

The result: Java cold starts drop from seconds to low hundreds of milliseconds. Not quite Rust-level, but a massive improvement.

There's a subtle problem with snapshots: uniqueness. If you snapshot a VM and restore 1,000 copies, all 1,000 have identical memory - identical PRNG state, identical UUIDs in flight, identical cryptographic keys in memory. AWS worked with the OpenSSL project, the Linux kernel team, and the Java security team to ensure that `/dev/urandom`, `java.security.SecureRandom`, and other entropy sources re-seed after restore. Without this, two restored VMs could generate the same "random" number.

## The rust-vmm ecosystem

Firecracker didn't stay a solo project. In December 2018, Amazon, Google, Intel, and Red Hat created the [rust-vmm](https://github.com/rust-vmm) project - a collection of shared Rust crates for building VMMs. The idea: the low-level KVM wrappers, memory management, and device model abstractions should be shared across projects rather than reimplemented.

Key crates:
- [`kvm-bindings`](https://crates.io/crates/kvm-bindings) - Rust FFI bindings for KVM data structures
- [`kvm-ioctls`](https://crates.io/crates/kvm-ioctls) - safe wrappers around KVM ioctl calls
- [`vm-memory`](https://crates.io/crates/vm-memory) - guest memory management with bounds checking
- [`vm-allocator`](https://crates.io/crates/vm-allocator) - MMIO/PIO address space management

Intel's [Cloud Hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor) (launched May 2019) is built on the same rust-vmm crates. So is [Dragonball](https://github.com/openanolis/dragonball-sandbox), Alibaba Cloud's VMM. The crates enforce safe guest memory access at the type level - you can't accidentally read past the end of a guest memory region because the API returns `Result` on every access, not a raw pointer.

This is the [Rust in production](/blog/rust-in-production---what-companies-actually-use-it-for) pattern playing out in virtualization: companies pick the components where memory safety translates directly into security guarantees, build shared infrastructure in Rust, and benefit from the compiler catching entire classes of vulnerabilities at build time.

## What this means for serverless

Firecracker made a specific tradeoff: give up hardware flexibility (no GPU passthrough, no legacy device support, no multi-architecture emulation) in exchange for boot speed, memory efficiency, and a minimal attack surface. That tradeoff only makes sense for workloads where you're creating and destroying VMs constantly - serverless functions, container tasks, short-lived batch jobs.

The practical impact:

**Density.** Thousands of isolated environments on a single host. This is what makes Lambda's pricing model work. If each function invocation needed 131 MB of overhead (QEMU), the cost per invocation would be dramatically higher.

**Boot speed.** 125ms for a microVM means Lambda's cold start is dominated by your function's initialization, not the infrastructure. For Rust functions, that initialization is basically zero - which is why the [cold start numbers](/blog/running-rust-in-aws-lambda---serverless-without-the-cold-sta) are 10-16ms, not 125ms+ (the microVM is pre-booted from the pool).

**Security without compromise.** Every Lambda function runs in its own VM with its own kernel. A vulnerability in your function's dependencies can't affect other customers, can't access the host, and can't escape the seccomp sandbox. This is stronger isolation than any container runtime provides, at comparable speed.

Firecracker proves something about Rust at the systems level: you can build security-critical infrastructure in 50,000 lines that replaces 2 million lines of C, boot faster, use less memory, and run at a scale where trillions of requests per month is normal operation. The 37 allowed syscalls, the formal verification with Kani, the safe device emulation code - these aren't theoretical benefits. They're the reason your Lambda function runs in an environment that's both fast and actually isolated.

The [source code](https://github.com/firecracker-microvm/firecracker) is on GitHub. If you've ever wondered what production Rust looks like at AWS-scale, the VMM event loop in `src/vmm/src/lib.rs` and the device emulation in `src/vmm/src/devices/` are worth reading. It's clean, well-documented, and a good example of safe abstractions built on top of carefully contained `unsafe` blocks.
