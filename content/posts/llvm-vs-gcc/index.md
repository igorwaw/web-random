---
title: "LLVM vs GNU GCC - compiler suites on Linux"
date: 2026-07-24T20:00:00
draft: true
tags: ["linux", "compilers"]
---

Every Linux distribution needs a C compiler - the kernel, most of userspace and the package manager itself are written in C or C++. For decades that meant GCC, full stop. Today, most distributions ship both GCC and LLVM/Clang, and quite a few packages - including parts of the Linux kernel and, on some systems, the base userland - are compiled with Clang instead. So why do we have two compiler suites doing largely the same job, and how different are they really?

## A brief history of GCC

GCC (originally the GNU C Compiler, now the GNU Compiler Collection) was started by Richard Stallman in 1987, as part of the GNU project. The goal was explicitly political as well as technical: a free software operating system needed a free compiler, since every other option at the time was proprietary. GCC was released under the GPL and became one of the earliest and most important pieces of the GNU project.

It quickly became the standard compiler on Unix-like systems, not just GNU/Linux. By the 1990s it was the default compiler on most Linux distributions and a common choice on BSD systems too, since it was free, portable and reasonably fast-moving compared to the vendor compilers it was competing with.

Development wasn't always smooth. In the late 1990s a group of developers, frustrated with the slow pace of the official GCC and the difficulty of getting patches accepted, forked the project as EGCS (Experimental/Enhanced GNU Compiler System). EGCS turned out to be technically superior, and in 1999 the Free Software Foundation made it the official GCC, resolving the fork. GCC has been developed under FSF and GNU project governance ever since, currently on a roughly yearly major release cadence.

## A brief history of LLVM/Clang

LLVM started in 2000 as a research project at the University of Illinois, led by Chris Lattner (initially as part of his studies, later continuing at Apple). The name originally stood for "Low Level Virtual Machine", although the project long outgrew that description and LLVM is now just a name, not an acronym.

Apple hired Lattner in 2005 and became the main driver behind LLVM for years - largely because Apple wanted a modern, liberally licensed compiler toolchain it could build tooling on top of, without depending on GCC's GPL and its at-the-time hostile stance towards a plugin architecture Apple wanted for Xcode. Clang, the C/C++/Objective-C frontend for LLVM, was announced in 2007 and reached production quality around 2010-2011, when it started shipping as the default compiler on macOS.

Since then LLVM has grown well beyond a single frontend and backend pair - it's now an umbrella project covering a shared set of reusable compiler and toolchain components (Clang for C-family languages, LLD as a linker, lldb as a debugger, compiler-rt for runtime support, and more), released under the Apache 2.0 license with LLVM exceptions. It's governed by the LLVM Foundation, with contributions from Apple, Google, Sony, ARM, Intel and many others.

## Languages and architectures

Both projects have grown to support far more than what they started with.

GCC's native frontends cover C, C++, Objective-C, Fortran, Ada, Go, D and Modula-2. Rust, despite frequent confusion, is not one of them - rustc uses LLVM as its backend (there's a separate, less mature `gccrs` effort to give Rust a GCC frontend). GCC supports an extremely wide range of target architectures, including plenty of embedded and legacy ones you won't find in LLVM: think AVR, m68k, SuperH or VAX, alongside the usual x86, ARM, RISC-V and POWER.

LLVM's own frontend is Clang, covering C, C++ and Objective-C. But because LLVM is designed as a set of reusable libraries rather than a single monolithic program, it has become the backend of choice for many other languages: Rust, Swift, Julia, Zig, Fortran (via Flang), and it's also an option for Fortran with GCC. Numerous JIT compilers and language runtimes embed LLVM directly, something that's much harder to do with GCC. LLVM's architecture support is smaller than GCC's but covers everything mainstream: x86, ARM/AArch64, RISC-V, POWER, WebAssembly and more, with less coverage of retro or niche embedded targets.

## Why do we need two compiler suites?

A few reasons, some historical, some very much current:

- **Licensing.** GCC is GPL, which means anything statically linking against its internals (as opposed to just invoking the `gcc` binary) inherits GPL obligations. LLVM's permissive licence lets companies embed it in proprietary tools without those obligations - this is a large part of why Apple and Sony use it so heavily.
- **Architecture.** LLVM was designed from day one as a library with a stable intermediate representation, meant to be reused piece by piece. GCC was designed as a compiler, and only later grew library-like interfaces. This makes LLVM much easier to build tooling on top of - static analysers, JITs, sanitizers, IDE integrations - without forking or reimplementing large chunks of it.
- **Diagnostics and tooling.** Clang built its reputation partly on much better error messages and diagnostics than GCC had at the time (GCC has since closed most of that gap). The sanitizers (ASan, UBSan, TSan) originated in the LLVM/Clang world.
- **Competition is healthy.** Having two independent, actively developed compilers catches different classes of bugs (miscompilations, undefined behaviour, security issues) and keeps both projects moving. The Linux kernel, for instance, is regularly built with both to catch bugs that only one compiler's warnings or optimisations would surface.
- **Nobody wants a single point of failure.** For something as fundamental as compiling your entire operating system, relying on a single project (and by extension, its maintainers, its licence and its roadmap) is a risk distributions would rather not take.

## Who actually uses what

The theory above plays out very differently depending on the platform.

### Linux kernel

Historically GCC-only - for a long time the kernel build system relied on GCC-specific extensions and wouldn't even compile with anything else. That changed with the ClangBuiltLinux project (started around 2018, later folded into the kernel community itself), which spent years fixing the kernel so it also builds cleanly with Clang. Mainline Linux has supported `CC=clang` for most architectures for a while now, and Google builds production Android kernels with Clang. GCC remains the default for most distributions' kernel packages, but Clang builds are a first-class, regularly-tested option, not an experiment. Rust support in the kernel (merged in 6.1) only exists via `rustc`, which is itself LLVM-based, so a Clang/LLVM toolchain is unavoidable if you want a kernel with Rust code enabled.

### Android

Android has fully moved to Clang/LLVM for native code. The NDK dropped GCC entirely years ago (it was deprecated, then removed), so any C/C++ code in an app or in the platform's native libraries (bionic libc, native services, and so on) is built with Clang. Android's Java/Kotlin side is unrelated to either - ART's own AOT/JIT compiler (`dex2oat` and friends) is an in-house compiler for Dalvik bytecode with no GCC or LLVM lineage (an early ART backend did briefly use LLVM around 2013, but it was replaced by a custom backend a year or so later for performance reasons).

### Apple

Apple is the biggest reason LLVM exists as a serious GCC alternative in the first place. Apple's last shipped GCC was version 4.2, the final release before GCC moved to GPLv3 - a licence Apple didn't want to distribute under - which left Apple stuck on an ageing compiler and strongly motivated to finish and adopt Clang. Xcode switched its default compiler to Clang in 2011, and GCC support was dropped from Xcode entirely not long after (the `gcc` command on macOS today is just a Clang alias). Swift, Apple's other language, is built on LLVM as well, sharing much of its optimiser and code generator with Clang.

### Windows

Neither GCC nor Clang is Microsoft's own compiler - that's MSVC (`cl.exe`), with its own separate codebase, ABI and C/C++ standard library. Both open source suites are available as alternatives, in slightly different flavours:

- **GCC on Windows** usually means MinGW-w64, which produces native Windows executables without any POSIX emulation layer (unlike Cygwin's GCC, which links against a POSIX compatibility DLL). It's a common choice for cross-compiling Windows binaries from Linux, and for standalone, self-contained builds it works fine. Where it gets awkward is interoperating with the native MSVC ecosystem: MinGW uses a different C++ ABI and runtime than MSVC, so mixing MinGW- and MSVC-built code (especially C++ objects across a DLL boundary) is unsupported or fragile, and it emits DWARF debug info that Visual Studio's own debugger can't read, so you're back to gdb. Historically there was also fragmentation across MinGW distributions around exception-handling models (SJLJ vs DWARF vs SEH) that caused its own compatibility headaches.
- **Clang on Windows** can act as a drop-in replacement for MSVC via `clang-cl`, a command-line driver that accepts `cl.exe`-compatible flags while still linking against the MSVC runtime and ABI. This lets projects that build with Visual Studio or MSBuild swap in Clang's frontend and diagnostics without changing their build system, and get MSVC-compatible debug info (PDB) for free - Chrome does this for its Windows builds. This interoperability isn't some inherent LLVM property, though - it's `clang-cl` deliberately imitating the MSVC ABI and runtime. Point Clang at the MinGW-w64 target instead and it has exactly the same interop limitations as GCC does. Even in `clang-cl` mode, compatibility isn't perfect - there are still occasional edge cases around obscure MSVC extensions or ABI corners - but it's been production-quality for years.

## Architecture difference

The single biggest structural difference is what "the compiler" actually is.

GCC is, at its core, a single large program per language/target combination, built around GENERIC and GIMPLE intermediate representations that aren't meant to be a stable, external interface - they can and do change between GCC versions, and there's no supported way to hook into the middle of the pipeline from outside the project. Plugin support exists but is limited and GPL-encumbered.

LLVM was built the other way round: the LLVM IR is a documented, relatively stable, textual and binary format that's a first-class citizen of the project. A frontend's only job is to translate source code into LLVM IR; everything downstream - optimisation passes, code generation, linking - operates purely on that IR and doesn't need to know anything about the original source language. That's what lets Rust, Swift or Julia reuse the same optimiser and code generator as Clang, and it's why you can pass around `.ll` or `.bc` files as an intermediate build artefact.

In practice this means:

| | GCC | LLVM/Clang |
|---|---|---|
| Structure | Monolithic compiler per frontend | Set of reusable libraries |
| Intermediate representation | GENERIC/GIMPLE, internal, unstable | LLVM IR, documented, relatively stable |
| Licence | GPLv3 (with runtime library exception) | Apache 2.0 with LLVM exceptions |
| Embeddable as a library | Difficult | Designed for it (JITs, custom tools) |
| Target architecture coverage | Very broad, including many legacy/embedded targets | Broad, focused on mainstream targets |
| Typical strength | Raw optimisation on some workloads, breadth of targets | Diagnostics, tooling, modularity |

Neither is strictly "better" - modern GCC and Clang produce code of broadly comparable quality, and each wins specific benchmarks depending on the workload and target. The practical difference that actually matters day to day is architectural: whether you need to reuse compiler internals as a library, whether you're targeting an architecture only one of them supports, and which licence fits your project.

## Relationship with Go

Go is a bit of an outlier here: its main compiler, `gc` (also called `cmd/compile`), is neither GCC nor LLVM-based. It's a self-hosted toolchain with its own backend, tracing its lineage back to the Plan 9 C compiler that some of Go's original authors (Ken Thompson, Rob Pike) had worked on. It doesn't emit GENERIC/GIMPLE or LLVM IR, and doesn't share code with either project.

That said, both suites do have a foothold:

- **gccgo** is a Go frontend for GCC, included in the GNU toolchain since GCC 4.6 (2011). It's an official, actively maintained alternative to `gc`, useful mainly on architectures or platforms `gc` doesn't support, or where being part of a standard GCC toolchain matters more than having the very latest language features - `gccgo` tends to lag a release or two behind.
- **gollvm**, an experimental LLVM-based backend for Go led by Google, reused the standard Go frontend but generated LLVM IR instead of `gc`'s native code, mainly to get LLVM's optimisations, wider architecture support and sanitizer integration. It never became a production default and development has largely stalled in favour of improving `gc` directly.

So unlike C/C++, where the choice is genuinely between GCC and Clang for most day-to-day purposes, Go's default toolchain sidesteps both - GCC and LLVM are secondary options rather than the main event.

## More info

- https://gcc.gnu.org/wiki/History
- https://llvm.org/pubs/2008-10-04-ACAT-LLVM-Intro.html
- https://www.aosabook.org/en/llvm.html
