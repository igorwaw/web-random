---
title: "eBPF - running your own code inside the kernel, safely"
date: 2026-11-19T00:00:00
draft: true
tags:
  - linux
  - security
---

In my [kernel compiling post](../compiling-linux-kernel/) I mentioned that you can learn a lot from the kernel configuration. But what if you want to change how the kernel behaves - add a firewall rule nobody thought of, measure something that has no counter, or trace a single function? Traditionally that meant writing a kernel module: C code with full kernel privileges, where one bug crashes the whole machine. eBPF offers a different deal: you can run your own code inside the kernel, but only code that the kernel can prove is safe.

## From packet filter to general-purpose VM

BPF, the Berkeley Packet Filter, dates back to 1992. It was a tiny virtual machine for one job: deciding which network packets `tcpdump` should capture, without copying every packet to userspace. When you run `tcpdump 'tcp port 443'`, the expression is compiled to BPF bytecode and loaded into the kernel. You can see it:

```bash
tcpdump -d 'tcp port 443'
```

In 2014 (kernel 3.18) Alexei Starovoitov's **extended BPF** turned this into something much bigger: 64-bit registers, more instructions, calls to kernel helper functions, and **maps** - key-value data structures shared between the kernel program and userspace. And, most importantly, it could be attached to many more places than network sockets. The old version is now called classic BPF (cBPF), and "BPF" usually means eBPF.

## How it works

1. You write a program in restricted C (or use a higher-level tool that generates it).
2. Clang compiles it to BPF bytecode.
3. The program is loaded into the kernel with the `bpf()` system call.
4. The **verifier** checks it, and refuses to load it if it can't prove it's safe.
5. The JIT compiler translates it to native machine code.
6. It's attached to a **hook**, and runs every time that hook fires.

The hooks are what makes it powerful:

- **kprobes / kretprobes** - almost any kernel function, on entry or return.
- **tracepoints** - stable, documented trace points in the kernel (system calls, scheduler, block I/O...).
- **uprobes** - functions in userspace programs and libraries, e.g. `SSL_write` in OpenSSL, before the data is encrypted.
- **XDP** (eXpress Data Path) - network packets, straight from the driver, before the kernel's network stack even sees them.
- **tc** - packets in the traffic control layer.
- **cgroups** - per-container network and device policies.
- **LSM** - security decisions, like a programmable SELinux.

## The verifier

The verifier is what makes eBPF different from a kernel module. Before a program is loaded, it simulates every possible path through the code and checks that:

- the program always terminates (loops are allowed only if the verifier can prove they are bounded),
- every memory access is within bounds - you can't just follow a pointer, you have to check it first,
- it only calls the helper functions allowed for that hook type,
- it doesn't leak kernel addresses to userspace (for unprivileged programs),
- it doesn't exceed the complexity limit.

The result: a buggy eBPF program can give you wrong results, but it shouldn't crash the kernel. Writing code that satisfies the verifier is notoriously annoying - it rejects code that is obviously safe to a human, because it can't prove it - but that's the price of safety.

Well, "shouldn't". The verifier itself is complex code, and it has had its share of bugs - and a verifier bug means unprivileged users could run code that reads or writes kernel memory. Combined with Spectre-type attacks that turned BPF into a convenient tool for speculative execution gadgets, this led to **unprivileged BPF being disabled by default**. On most current distributions:

```bash
$ sysctl kernel.unprivileged_bpf_disabled
kernel.unprivileged_bpf_disabled = 2
```

Today, loading most BPF programs requires root or the `CAP_BPF` capability (plus others, depending on the program type).

## Who uses it

More software than you'd think:

- **Networking.** Cilium, the popular Kubernetes networking plugin, replaces iptables with eBPF programs. Meta's Katran load balancer and Cloudflare's DDoS mitigation run on XDP, dropping malicious packets before they cost any CPU time in the network stack.
- **Observability.** `bpftrace`, the BCC tools, and many commercial monitoring agents.
- **Security.** Falco and Tetragon watch process executions, file access and network connections at the kernel level, which is much harder to evade than userspace monitoring.
- **systemd** uses BPF for per-service IP firewalling (`IPAddressDeny=`) and device access control in cgroups.
- **sched_ext** (kernel 6.12) lets you write CPU schedulers as BPF programs, so new scheduling ideas can be tested without patching the kernel.
- It's not even Linux-only any more: Microsoft has an eBPF for Windows project.

## Try it: bpftrace one-liners

The easiest way to start is `bpftrace` - an awk-like language for tracing. Install it (`sudo apt install bpftrace`) and try a few one-liners. They all run until you press Ctrl+C, then print the results.

Which files are being opened, by which process:

```bash
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args.filename)); }'
```

Which processes make the most system calls:

```bash
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
```

Every new program being started, system-wide (great for finding out what a mysterious script actually runs):

```bash
sudo bpftrace -e 'tracepoint:sched:sched_process_exec { printf("%d %s\n", pid, str(args.filename)); }'
```

Who's writing to the disk, by bytes (note that writes often get issued by kernel threads later, so this shows `kworker` a lot - finding the real culprit is a good exercise):

```bash
sudo bpftrace -e 'tracepoint:block:block_rq_issue { @[comm] = sum(args.bytes); }'
```

A histogram of read sizes, for a specific process:

```bash
sudo bpftrace -e 'tracepoint:syscalls:sys_exit_read /comm == "firefox"/ { @bytes = hist(args.ret); }'
```

The `@` variables are BPF maps: the kernel side fills them in, and bpftrace prints them at exit. No kernel module, no reboot, and almost no overhead on the traced system.

For ready-made tools, the BCC collection (`bpfcc-tools` on Debian) has dozens of them: `execsnoop`, `opensnoop`, `biolatency`, `tcpconnect`, `tcplife`... Brendan Gregg's books and website are the best resource if you want to go deeper.

## The dark side

Anything that can observe and modify the system at the kernel level, invisibly to userspace, is attractive to attackers too. eBPF-based rootkits have been demonstrated at security conferences and found in the wild: they can hide files and processes, intercept SSH logins, or capture data passing through `SSL_write` before it's encrypted - all without loading a kernel module, which is what most rootkit detectors look for.

The good news is that loading BPF programs requires root, so this is a persistence and stealth technique, not a way in. And you can list what's loaded:

```bash
sudo bpftool prog list
sudo bpftool map list
```

On a modern desktop, don't be surprised to find a few dozen programs: systemd loads several. Knowing what's normal on your system is the first step to noticing what isn't.

## Should you care?

If you manage Linux systems, `bpftrace` and the BCC tools are worth learning - they answer questions that used to require guesswork or a debug kernel. If you write software, you probably won't write eBPF programs directly, but there's a good chance something on your servers already runs them.
