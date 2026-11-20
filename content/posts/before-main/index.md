---
title: "What happens before main() - from ELF to a running process"
date: 2026-11-19T00:00:00
draft: true
tags:
  - linux
  - compilers
---

Every C tutorial starts with `main()`, as if that's where a program begins. It isn't. By the time `main()` runs, the kernel and the dynamic loader have done quite a lot of work: mapped the file into memory, found and loaded the libraries, fixed up hundreds of addresses, and run code you never wrote. Let's follow it step by step.

Our test subject:

```c
#include <stdio.h>
int main(void) {
    puts("Hello");
    return 0;
}
```

```bash
$ gcc -O2 -o hello hello.c
$ file hello
hello: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=9c53ab93ce948601478bc0f99efda993c42c283e, for GNU/Linux 3.2.0, not stripped
```

That one line already contains most of the story: ELF, PIE, dynamically linked, interpreter.

## ELF - the file format

ELF (Executable and Linkable Format) is the format of executables, shared libraries, object files and core dumps on Linux and most other Unix-like systems. It starts with a header:

```bash
$ readelf -h hello
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  ...
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Entry point address:               0x1070
  ...
```

The magic number is `0x7f` followed by `ELF` in ASCII. Note the **entry point**: `0x1070`. That's not `main()` - we'll get to what it is.

An ELF file has two views of the same content. **Sections** (`.text` for code, `.data`, `.rodata`, `.bss` and many more) are for the linker and debugger. **Segments** (program headers) are for the kernel - they say what to load where, and with what permissions:

```bash
$ readelf -lW hello | grep -E 'LOAD|GNU_RELRO|GNU_STACK'
  LOAD           0x000000 0x0000000000000000 0x0000000000000000 0x000608 0x000608 R   0x1000
  LOAD           0x001000 0x0000000000001000 0x0000000000001000 0x000165 0x000165 R E 0x1000
  LOAD           0x002000 0x0000000000002000 0x0000000000002000 0x000120 0x000120 R   0x1000
  LOAD           0x002dd0 0x0000000000003dd0 0x0000000000003dd0 0x000248 0x000250 RW  0x1000
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x002dd0 0x0000000000003dd0 0x0000000000003dd0 0x000230 0x000230 R   0x1
```

Four chunks to map: read-only headers, executable code (`R E`), read-only data and writable data. Nothing is both writable and executable, and the stack (`GNU_STACK`) isn't executable either - basic exploit mitigations, now the default.

## The kernel's part: execve()

When the shell runs `./hello`, it calls `fork()` and then `execve()`. The kernel:

1. Reads the first bytes and recognises the ELF magic (the same mechanism handles `#!` scripts and, via `binfmt_misc`, things like Windows `.exe` files through Wine).
2. Maps the `LOAD` segments into a fresh address space. Since this is a PIE (Position-Independent Executable), it picks a random base address - that's ASLR.
3. Finds the `INTERP` segment, which names the **dynamic loader**: `/lib64/ld-linux-x86-64.so.2`. The kernel maps that too.
4. Sets up the stack: `argc`, `argv`, the environment variables, and the **auxiliary vector** - extra information from the kernel, such as where the program headers are, the page size, and the address of the **vDSO**.
5. Jumps to the entry point - of the **loader**, not the program.

The kernel doesn't know anything about shared libraries. That's entirely the loader's job. In fact, the loader is a program on its own - you can run it directly:

```bash
$ /lib64/ld-linux-x86-64.so.2 ./hello
Hello
```

(This works even if `hello` doesn't have the executable bit, which is a classic way of bypassing a `noexec` mount.)

## The loader's part

The dynamic loader (`ld.so`, part of glibc) reads the program's `DYNAMIC` section to find out what it needs:

```bash
$ readelf -d hello | grep NEEDED
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
$ ldd hello
	linux-vdso.so.1 (0x00007f75da908000)
	libc.so.6 => /usr/lib/x86_64-linux-gnu/libc.so.6 (0x00007f75da6e7000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f75da90a000)
```

It searches for each library (in `RPATH`/`RUNPATH`, `LD_LIBRARY_PATH`, the `/etc/ld.so.cache` and default directories), maps it, and repeats for that library's own dependencies. `linux-vdso.so.1` is special: it's not a file at all, but a tiny library the kernel maps into every process, so that calls like `gettimeofday()` don't need a real system call.

Then come **relocations**. The program and libraries are loaded at random addresses, so every place where code refers to an absolute address, or to a symbol in another library, needs to be fixed. The loader has a built-in statistics option:

```bash
$ LD_DEBUG=statistics ./hello
    runtime linker statistics:
      total startup time in dynamic loader: 34708 cycles
            time needed for relocation: 680 cycles (1.9%)
                 number of relocations: 96
      ...
```

96 relocations for a hello world. For a big C++ application with dozens of libraries it's hundreds of thousands, which is why startup time of large desktop apps was a real problem in the past (and why tricks like prelinking existed).

## PLT and GOT - calling a library function

How does `puts()` in our program find `puts()` in libc, if libc's address is random? Through two tables:

- **GOT** (Global Offset Table) - a table of addresses in the writable data segment, one entry per external symbol.
- **PLT** (Procedure Linkage Table) - small stubs in the code. Calling `puts` in our program actually calls `puts@plt`, which jumps to whatever address is in the GOT entry for `puts`.

```bash
$ objdump -R hello | grep puts
0000000000004000 R_X86_64_JUMP_SLOT  puts@GLIBC_2.2.5
```

By default, these entries are resolved **lazily**. Initially the GOT entry points back to a resolver in the loader. On the first call to `puts`, the resolver looks up the real address, writes it into the GOT and jumps there. The next call goes straight to libc. You can watch it happen:

```bash
$ LD_DEBUG=bindings ./hello
    240482:	binding file ./hello [0] to /usr/lib/x86_64-linux-gnu/libc.so.6 [0]: normal symbol `puts' [GLIBC_2.2.5]
Hello
```

Lazy binding makes startup faster, but a writable table of function pointers is a gift for attackers - overwrite a GOT entry and the next call to that function goes wherever you want. Hence **RELRO**: with "full RELRO" (`-Wl,-z,relro,-z,now`), all symbols are resolved at startup and then the GOT is made read-only. Some distributions build everything this way. Our binary uses partial RELRO (no `BIND_NOW` flag), which is GCC's default on Debian.

By the way, even if the source code used `printf("Hello\n")`, the binary would still call `puts` - GCC replaces it, since there's nothing to format. A fun detail when you're wondering why your `printf` breakpoint never triggers.

## LD_PRELOAD - cutting in line

Since the loader decides which library provides each symbol, you can tell it to look in your library first. Here's a library that replaces `puts` with a version that shouts:

```c
#define _GNU_SOURCE
#include <ctype.h>
#include <dlfcn.h>
#include <string.h>

int puts(const char *s) {
    static int (*real_puts)(const char *);
    if (!real_puts)
        real_puts = dlsym(RTLD_NEXT, "puts");
    char buf[256];
    size_t i;
    for (i = 0; s[i] && i < sizeof(buf) - 1; i++)
        buf[i] = toupper((unsigned char)s[i]);
    buf[i] = '\0';
    return real_puts(buf);
}
```

```bash
$ gcc -shared -fPIC -o shout.so shout.c
$ ./hello
Hello
$ LD_PRELOAD=./shout.so ./hello
HELLO
```

`RTLD_NEXT` finds the next definition of `puts` - the real one in libc - so the wrapper can call it. This is how many useful tools work: `faketime` (fake the system time for one program), `libeatmydata` (make `fsync()` a no-op), `jemalloc` or `tcmalloc` replacing `malloc`, and various debugging and profiling tools. It's also a well-known trick used by malware to hide files and processes (via `/etc/ld.so.preload`, which applies to every program on the system).

For obvious reasons, the loader ignores `LD_PRELOAD` (and `LD_LIBRARY_PATH`) for setuid programs - otherwise any user could inject code into `sudo`.

## Finally, main()

After loading and relocating, the loader runs the **initialisers** of all libraries (C++ global constructors, functions marked `__attribute__((constructor))`) and jumps to the program's entry point, `_start`. That's the `0x1070` from the ELF header - a few instructions of assembly from glibc's `crt1.o`, linked into every program. `_start` calls `__libc_start_main()`, which sets up a few more things (thread-local storage, the `atexit` handlers, stack protector canary) and only then calls `main(argc, argv, envp)`.

When `main()` returns, its return value goes to `exit()`, which runs `atexit` handlers and destructors, flushes the `stdio` buffers, and calls the `exit_group` system call.

## Static binaries are back

All of this can be skipped. A statically linked binary contains everything it needs, has no `INTERP` segment, and the kernel jumps straight to its `_start`:

```bash
$ gcc -O2 -static -o hello-static hello.c
$ file hello-static
hello-static: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, ...
$ LD_PRELOAD=./shout.so ./hello-static
Hello
```

No loader, no `LD_PRELOAD`. The price is size: 16 KB for the dynamic version, 747 KB for the static one - most of it glibc, which was never really designed for static linking (functions like `getpwnam()` or DNS lookups through NSS still try to load shared libraries at runtime).

Static linking used to be considered old-fashioned: every program carries its own copy of every library, and a security fix in a library means rebuilding everything. But it's back in fashion. Go produces static binaries by default (as long as you avoid cgo), Rust can easily target musl, and Alpine-based containers made musl libc popular. A single file that runs on any Linux distribution, with no dependency on the installed library versions, turned out to be very convenient for containers and command-line tools. The distributions are less happy about it - but that's a topic for another post.

## Tools to explore further

- `readelf`, `objdump`, `nm` - inspect ELF files.
- `ldd` - list libraries (but don't run it on untrusted binaries, it may execute them).
- `LD_DEBUG=help ./anything` - all the debug options of the loader.
- `strace ./hello` - see the system calls, including the loader's `openat` and `mmap` calls for each library.
- `cat /proc/<pid>/maps` - the memory map of a running process.
