---
title: "LLVM vs GNU GCC - compiler suites on Linux"
date: 2026-07-24T20:00:00
draft: true
tags: ["linux", "compilers"]
---

Every Linux distribution needs a C compiler - the kernel and most of userspace are written in C. For many years that meant GCC, no alternatives. Today, most distributions ship both GCC and LLVM/Clang, and quite a few packages - including the Linux kernel in some cases - are compiled with Clang instead. Both projects grew to cover many other programming languages. So why do we have two compiler suites doing the same job?

## A brief history of GCC

GCC (originally the GNU C Compiler, now the GNU Compiler Collection) was started by Richard Stallman in 1987, as part of the GNU project. The goal was explicitly political as well as technical: a free operating system needed a free compiler. GCC is released under the GPL licence, which means all derivative works need to be disributed with the same license.

The GNU project ended up releasing many important components for a free Unix-like system, starting with development tools. It didn't, however, release a usable kernel[^1], and Linux, created a few years later, was the missing piece. GCC quickly became the standard compiler on Linux, but also other Unix-like systems. It was a common choice on BSD systems too, since it was free, portable and reasonably fast-moving (FreeBSD later moved to Clang).

Initially a C compiler, it later added more languages, beginning with C++ soon after the initial release.

Development wasn't always smooth. Richard Stallman is known to be hard to work with. In the late 1990s a group of developers, frustrated with the slow pace of development and the difficulty of getting patches accepted, forked the project as EGCS (Experimental/Enhanced GNU Compiler System). EGCS turned out to be technically superior, and in 1999 the Free Software Foundation made it the official GCC, resolving the fork. GCC 2.95, released shortly afterwards, was GCC by another name. GCC has been developed under FSF and GNU project governance ever since, currently on a roughly yearly major release cadence.

## A brief history of LLVM/Clang

LLVM started in 2000 as a research project at the University of Illinois, led by Chris Lattner (initially as part of his studies, later continuing at Apple). The name originally stood for "Low Level Virtual Machine", although the project long outgrew that description and LLVM is now just a name, not an acronym.

Apple hired Lattner in 2005 and became the main driver behind LLVM for years - largely because Apple wanted a compiler and toolchain that were both modern and non-GPL licensed. GCC didn't want to allow non-GPL plugins and Apple didn't want to open all of their code. Clang, the C/C++/Objective-C frontend for LLVM, was announced in 2007 and reached production quality around 2010-2011, when it started shipping as the default compiler on macOS.

Since then LLVM has grown well beyond a single frontend and backend pair - it's now an umbrella project covering a shared set of reusable compiler and toolchain components (Clang for C-family languages, LLD as a linker, lldb as a debugger, compiler-rt for runtime support, and more), released under the Apache 2.0 license with LLVM exceptions. It's governed by the LLVM Foundation, with contributions from Apple, Google, Sony, ARM, Intel and many others.

## Languages and architectures

Both projects have grown to support far more than what they started with.

GCC's native frontends cover C, C++, Objective-C/C++, Fortran, Ada, Cobol, Go, D, Algol-68 and Modula-2. A notable omission is Rust (there's an experimental support, but so far it can't compile non-trivial code); some old versions could also compile Java to native machine code (instead of JVM) but that was dropped in 2016. GCC supports an extremely wide range of target architectures - dozens of CPUs, some I haven't even heard about, in addition to the usual ones such as x86, ARM, RISC-V. If you need to compile code for a 1980s CPU, GCC is usually the only choice.

LLVM's main frontend is Clang, covering C, C++, Objective-C/C++, plus GPU languages (OpenCL, CUDA and HIP). But because LLVM is designed as a set of reusable libraries rather than a single monolithic program, it has become the backend of choice for many other languages: Fortran is official (though with another frontend called Flang) while Rust, Swift, Julia, Zig and many more are 3rd party. Numerous JIT compilers and language runtimes embed LLVM directly, something that's much harder to do with GCC. LLVM's architecture support covers everything mainstream: x86, ARM/AArch64, RISC-V, POWER, WebAssembly and a few more, but not the whole range of retro or niche targets.

## Why do we need two compiler suites?

- **Licensing.** GCC is GPL, which means anything statically linking against its internals (as opposed to just invoking the `gcc` binary) inherits GPL obligations. LLVM's permissive licence lets companies embed it in proprietary tools without those obligations. This is the main reason why Apple chose it.
- **Architecture.** GCC was designed as a compiler, and only later grew library-like interfaces. LLVM is a much newer design and it was modular from day one. The components have stable APIs and are meant to be reused piece by piece. This makes LLVM much easier to build tooling on top of - static analysers, JITs, sanitizers, IDE language servers, you name it.
- **Diagnostics and tooling.** Clang built its reputation partly on much better error messages and diagnostics than GCC had at the time (GCC has since improved a lot). The sanitizers (ASan, UBSan, TSan) originated in the LLVM/Clang world.
- **Competition is healthy.** Having two independent, actively developed compilers catches different classes of bugs (miscompilations, undefined behaviour, security issues) and keeps both projects moving. The Linux kernel, for instance, is regularly built with both to catch bugs that only one compiler's warnings or optimisations would surface.
- **Nobody wants a single point of failure.** For something as fundamental as compiling your entire operating system, relying on a single project (and by extension, its maintainers and their goals) is risky.

In summary:

| | GCC | LLVM/Clang |
|---|---|---|
| Structure | Monolithic compiler per frontend | Set of reusable libraries |
| Intermediate representation | GENERIC/GIMPLE, internal, unstable | LLVM IR, documented, relatively stable |
| Licence | GPLv3 (with runtime library exception) | Apache 2.0 with LLVM exceptions |
| Embeddable as a library | Difficult | Designed for it (JITs, custom tools) |
| Programming languages | about 12 | about 7 native + several dozen 3rd party |
| Target architecture coverage | Very broad, including many legacy targets | Focused on mainstream targets |
| Typical strength | Raw optimisation on some workloads, breadth of targets | Diagnostics, tooling, modularity |

Neither is strictly "better". If we're talking about C and C++ on Linux, both produce code of comparable quality. You can find a benchmark where one or the other wins, but neither is always better in optimisation. The difference is about licence most of the time, and sometimes it's about specific applications.

## Platform differences

### Linux kernel

Historically GCC-only - for a long time the kernel wouldn't compile with anything else. That slowly changed - it started around 2012 with LLVMLinux project, later moved into ClangBuiltLinux project and finally the mainstream kernel. Linux has supported `CC=clang` for most architectures for a while now. GCC remains the default for most distributions' kernel packages, but Clang builds are a first-class, regularly-tested option, not an experiment. Rust support in the kernel (merged in 6.1) only exists via `rustc`, which is itself LLVM-based, so LLVM toolchain is unavoidable if you want a kernel with Rust code enabled (for the Rust part, that is - C part can be compiled with GCC, though mixing compiler withing one build is not recommended).

### Android

Android has fully moved to Clang/LLVM for native code (including the kernel). The NDK dropped GCC entirely years ago (it was deprecated, then removed), so any C/C++ code in an app or in the platform's native libraries (bionic libc, native services, and so on) is built with Clang. Android's Java/Kotlin side is unrelated to either.

### Apple

Apple is the biggest reason LLVM exists as a serious GCC alternative. Apple's last shipped GCC was version 4.2.1, the final release before GCC moved to GPLv3 - a licence Apple didn't want to distribute under. Which meant Apple could either be stuck forever on an ageing compiler or needed to migrate to another project - and they chose CLANG. Xcode switched its default compiler to Clang in 2011, and GCC support was dropped from Xcode entirely not long after (the `gcc` command on macOS today is just a Clang alias). Swift, Apple's other language, is built on LLVM as well, sharing much of its optimiser and code generator with Clang.

### Windows

Microsoft's own compiler is MSVC (Microsoft Visual C++ - contrary to the name, it can be used as a CLI tool outside Visual Studio). It's free to use for individuals, for companies - some restrictions apply. Both open source suites also work on Windows:

- **GCC on Windows** usually means MinGW-w64, which produces native Windows executables without any POSIX emulation layer (there's also Cygwin's GCC, which links against a POSIX compatibility DLL). It's a common choice for cross-compiling Windows binaries from Linux, and for standalone, self-contained builds it works fine. Where it gets awkward is interoperating with the native MSVC ecosystem: MinGW uses a different C++ ABI and runtime than MSVC, so mixing MinGW- and MSVC-built code (especially C++ objects across a DLL boundary) is unsupported or fragile.
- **Clang on Windows** can act as a drop-in replacement for MSVC via `clang-cl`. This lets projects that build with Visual Studio or MSBuild swap in Clang's frontend and diagnostics without changing their build system, and get MSVC-compatible debug info (PDB) for free - Chrome does this for its Windows builds. This interoperability isn't some inherent LLVM property, though - it's `clang-cl` deliberately imitating the MSVC ABI and runtime. Point Clang at the MinGW-w64 target instead and it has exactly the same interop limitations as GCC does. Even in `clang-cl` mode, compatibility isn't perfect - there are still occasional edge cases around obscure MSVC extensions or ABI corners - but it's been production-quality for years.

## More info

- [A Brief History of GCC](https://gcc.gnu.org/wiki/History)
- [Chris Lattner - Introduction to the LLVM Compiler System (2008)](https://llvm.org/pubs/2008-10-04-ACAT-LLVM-Intro.html)
- [The Architecture of Open Source Applications - LLVM](https://www.aosabook.org/en/llvm.html)
- [GCC Front Ends](https://gcc.gnu.org/frontends.html)
- [GCJ - GCC Wiki](https://gcc.gnu.org/wiki/GCJ)
- [LWN - gccrs, the GCC Rust front end](https://lwn.net/Articles/991199/)
- [Flang - the LLVM Fortran compiler](https://flang.llvm.org/)
- [Clang documentation - MSVC compatibility](https://clang.llvm.org/docs/MSVCCompatibility.html)
- [The LLVMLinux Project - announcement on llvm-dev (2012)](https://lists.llvm.org/pipermail/llvm-dev/2012-September/053147.html)
- [ClangBuiltLinux](https://clangbuiltlinux.github.io/)
- [Linux kernel documentation - Rust Quick Start](https://docs.kernel.org/rust/quick-start.html)
- [LWN - building the kernel with Rust and GCC](https://lwn.net/Articles/1024202/)

[^1]: Technically, GNU Hurd was started a year before Linux and it used a promising modern architecture (microkernel) instead of a traditional monolithic design of Linux. But in a few years Linux was used in production servers, backed by several companies, developed by hundreds of contributors and ported to several architectures; GNU Hurd has never reached maturity.