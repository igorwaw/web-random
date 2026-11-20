---
title: "Reproducible builds - can you trust your distro's binaries?"
date: 2026-11-20T19:00:00
draft: false
tags:
  - linux
  - security
  - compilers
---

Open source means you can read the code but almost nobody does. And even if you do, how can you be sure that the code you're running is the same? Compiling the whole operating system with libraries and applications directly from source is possible - that was the whole point, after all - and some people do it for education. But it's not a practical way of keeping a fleet of servers updated. We need a way to prove the provenance of the package.

Package signatures are one step - and an important one - but they only prove the binary came from the distribution. They don't prove it was built from the source it claims to be built from. That's the gap reproducible builds are trying to close.

A build is reproducible if anyone who compiles the same source, with the same tools and parameters, gets a **bit-for-bit identical** result. Then anyone can verify the build server: rebuild the package independently and compare the checksums. If they match, the binary really comes from that source.

By default, the builds are not reproducible. Compilers and build systems embed timestamps, paths, usernames, file ordering and random keys into their output. Debian, Arch and Fedora now build the vast majority of their packages reproducibly. That took a lot of effort over the years.

## How it began: Trusting Trust

In 1983 Ken Thompson gave his Turing Award lecture, "Reflections on Trusting Trust". He described a compiler modified to do two things: insert a backdoor when compiling the `login` program, and insert its own modification when compiling the compiler itself. Once you have such a compiler binary, you can remove the malicious code from the compiler source so there's no trace - but every compiler built from clean source with the infected binary is infected too.

That's an extreme example of a supply chain attack. Is it possible in the real world? Ken Thompson created a proof of concept, which worked (ironically, the backdoor made the build non-reproducible in an unusual way: the compiler grew by one byte each time it compiled itself). There were similar cases in the wild: Induc virus infected Delphi compilers in 2009, and XCodeGhost in 2015 targeted, you guessed it, Xcode.

Moving away from compilers: build servers, used (among others) by Linux distros, are a very attractive target. If you compromise one machine, your malware will be shipped to millions of users, carrying a valid signature and coming from a trusted source.

## Why aren't builds reproducible by default?

### Timestamps

Let's try a simple example:

```c
#include <stdio.h>
int main(void) {
    printf("Hello, built on %s %s\n", __DATE__, __TIME__);
    return 0;
}
```

Compile it twice, a few seconds apart:

```bash
$ gcc -O2 -o hello1 hello.c
$ gcc -O2 -o hello2 hello.c
$ sha256sum hello1 hello2
e095fb1f0e1f44220468876b93075a4985c21f0a9f32c17c917a75f3a824246f  hello1
0b3715369f9bcd163e85e0ff55b746301893f3ed264ec54d5b6df97b7cab0f33  hello2
```

Different. Obviously, that's because the program embeds the build time. Do you think I cheated, no real software does that? Quite the contrary, many popular apps have it in their version string. I bet you have at least two of those:

```bash
igor@ood:/$ vim --version | head -n 1
VIM - Vi IMproved 9.2 (2026 Feb 14, compiled Jul 27 2026 10:11:19)
igor@ood:/$ uname -v
#1 SMP PREEMPT_DYNAMIC Debian 7.2.15-1 (2026-11-01)
igor@ood:/$ python3 -VV
Python 3.14.10 (main, Oct 26 2026, 15:12:11) [GCC 16.2.0]
```

And the timestamps aren't only there if the source code explicitly references them. A real app is not just a single C file that turns into a single binary. A deb or rpm package contains files, and they all have modification time. Worse, those files might be archives (e.g. jar) that contain more files.

### Other usual suspects

- **Build paths.** Debug information records the directory the code was compiled in. Build the same thing in `/home/alice/src` and `/build/pkg-1234` and you get different binaries.
- **File ordering.** If a build script does the equivalent of `for f in *.o` without sorting, the link order (and therefore the binary) can be different in every case.
- **Environment.** Locale, timezone, umask, hostname, username.
- **Randomness.** Hash table seeds, temporary file names that end up in the output, and the big one: **randomly generated signing keys**. If you followed my [kernel compiling post](../compiling-linux-kernel/), the build generated a fresh module signing key, so every build of the same kernel is different. Not a bug, that's the whole point of a throwaway key - but it does break reproducibility.
- **Parallelism.** Parallel builds, e.g. `make -j16`, can finish things in a different order, and it can affect the binary.

## How it's fixed

The Reproducible Builds project started in Debian around 2013 (Bitcoin had been doing deterministic builds from 2011 and Tor Browser started a bit earlier in 2013) and grew into a cross-distribution effort: Arch and a few others distros were next. Fedora joined later and is catching up.

### .buildinfo, .BUILDINFO and `koji buildinfo`

To reproduce the build, you need to use the same versions of the compiler, linker, documentation generators and other tools. And then you need to have the same path and environment variables. So the first step is writing down what the build environment actually was. Debian's standard is the `.buildinfo` file. It records all these plus the checksums of the resulting binaries and it's published alongside the packages. Arch uses a `.BUILDINFO` file inside every package. Despite the similar name, it's a different format.

Fedora has a different approach - its build tool `koji` can query the build server to read the environment. If you rebuild a package with `fedora-repro-build`, it will happen automatically, so the end result will have the same checksum.

Some distros have rebuild servers in addition to the usual build servers. They continuously rebuild packages and would alert the developers if the checksum differed. But you can also get the environment information and do the same process yourself. It's important for creating trust.

### Build time

Timestamps are the most important sources of non-reproducibility. The fix is the **`SOURCE_DATE_EPOCH`** environment variable, containing a Unix timestamp, that tells all build tools "pretend this is the current time". GCC and Clang use it for `__DATE__` and `__TIME__`. Which means that if an app chooses to embed a build time in its version string, it will work - the time won't be real, but who cares anyway? Debian and Fedora set the SOURCE_DATE_EPOCH to the date of the last changelog entry, so it's fixed for a given source package. A long list of other tools (tar, zip, Sphinx, gzip wrappers, documentation generators) were patched to respect it too.

Let's test it on the same C file.

```bash
$ export SOURCE_DATE_EPOCH=1700000000
$ gcc -O2 -o hello3 hello.c
$ gcc -O2 -o hello4 hello.c
$ sha256sum hello3 hello4
a8ad13ca37c3242c42466b3ac39da35e5f66423ba2dc9145d8cbfcac639bf30e  hello3
a8ad13ca37c3242c42466b3ac39da35e5f66423ba2dc9145d8cbfcac639bf30e  hello4
$ ./hello3
Hello, built on Nov 14 2023 22:13:20
```

### Build path

Both GCC and Clang have a parameter **`-ffile-prefix-map`** to map directory paths. It was added specifically to make builds reproducible regardless of the current path. Let's do the experiment again:

```bash
$ (cd a && gcc -g -O2 -o ../h1 $PWD/hello.c)
$ (cd b && gcc -g -O2 -o ../h2 $PWD/hello.c)
$ sha256sum h1 h2
35564b676c8a1e0ea85d78b2a298912c3db87abf98847be1a6701acd7827769e  h1
3300ea4ea1dcfba4ad1c7661895b0fff6a6f3aa117dbb3bea3d2cb6f213f114d  h2
$ (cd a && gcc -g -O2 -ffile-prefix-map=$PWD=. -o ../h1 $PWD/hello.c)
$ (cd b && gcc -g -O2 -ffile-prefix-map=$PWD=. -o ../h2 $PWD/hello.c)
$ sha256sum h1 h2
632b8ebaaa01e74390f11f2d6277de330bd343caa8e7d8eced1512ce5f4c070d  h1
632b8ebaaa01e74390f11f2d6277de330bd343caa8e7d8eced1512ce5f4c070d  h2
```

### Special case: the Linux kernel

The Linux kernel embeds a few things about the environment: build time, username and hostname. It was a rational thing to do in the 1990s when people often compiled custom kernels: easier to tell which one you're running if you had several to choose from, and if something didn't work, you knew which admin to blame. Older Unix systems that Linux imitated did the same and for the same reasons. Today, when almost everyone uses a distribution's kernel, it's not needed anymore and of course it would totally break reproducibility. But all of these can be configured with environment variables: `KBUILD_BUILD_TIMESTAMP`, `KBUILD_BUILD_USER` and `KBUILD_BUILD_HOST`.

Another problem is module signing. For the reproducible build, you need to provide a pre-generated key instead of letting the build create a random one - and then keep it secret, because anyone with the key can produce the same signed binary. That's an interesting conflict: a signature is by definition something only one party can produce, so signed artefacts are verified by stripping or ignoring the signature and comparing the rest.

### Everything else

For other problems there's `strip-nondeterminism` (Debian) or `add-determinism` (Fedora): post-processing tools that normalise timestamps and ordering in archives after the build. And if that fails - patching of build scripts. Hundreds of upstream projects received such patches.

### Tools for checking

- **diffoscope** - "diff on steroids". Give it two files (rpm/deb packages, ISOs, archives, binaries) and it recursively unpacks them and shows you *where* they differ. Even if it means finding a specific file in a zip inside a jar inside a deb. Even if you don't care about reproducible builds, it's a great tool for other uses.
- **reprotest** - builds the same thing twice while deliberately varying the environment (time, path, locale, user, timezone, file order, umask...) to check for hidden dependencies.
- **debrebuild** (Debian) and **makerepropkg** (Arch) - rebuild an official package locally with exactly the same build environment as the original one, using the .buildinfo/.BUILDINFO file
- **koji** - a client for Fedora build system
- **rebuilderd** - an rebuilder software used by some distros, which continuously rebuilds packages and publishes the results.

## Where are we now?

Debian's continuous testing at tests.reproducible-builds.org shows about 96% of packages in testing building reproducibly - and that's while deliberately varying the environment between the two builds, which is harsher than a real-world rebuild. As is often the case, achieving the first 90% was easy, while the rest was the struggle. Arch Linux is behind with 74% (though the core repository is 93%, extras are the problem - and many of those are not non-reproducible, just not tested yet).

Fedora reached about 90% reproducible package builds mostly with infrastructure changes. They initially aimed to achieve 99% in Fedora 43, but later moved the target to Fedora 46. RHEL is built from Fedora, so it benefits from the same work, but Red Hat doesn't publish any reproducibility numbers for it. Ubuntu doesn't publish any either: packages synced unchanged from Debian benefit from Debian's fixes, but nobody continuously rebuilds Ubuntu to check, and Launchpad only started publishing the `.buildinfo` files needed for independent rebuilds in 2024.

NixOS and Guix get it largely for free from their design, since their builds already run in isolated, pinned environments; NixOS announced as early as 2021 that its minimal installation ISO was reproducible. Guix took it a step further with **bootstrappable builds**: the whole distribution can be built starting from a tiny binary seed of a few hundred bytes, small enough to audit by hand (you don't even need to trust a pre-built GCC). That's the real answer to Trusting Trust.

## The xz backdoor and the important next step

In March 2024 Andres Freund noticed that SSH logins on his Debian sid machine were taking half a second longer than they should and found a backdoor in xz/liblzma (CVE-2024-3094). After years of patient social engineering, the attacker became a maintainer and slipped the payload in through "binary test files" plus a modified build script.

Here's the catch: the modified build script (`build-to-host.m4`) wasn't in the git repository. It was only in the **release tarball**. Build servers commonly get the source from release files, while individual users often get it directly from git. Reproducible builds verify that the binary matches the source - and it did. The source that was used was the problem.

What would have helped is comparing the tarball with the git tag, which is now a hot topic - some distributions are moving towards building from git instead of from tarballs, and reproducible builds people are working on making release tarballs themselves reproducible from the repository.

## What it doesn't protect from

Having reproducible builds doesn't automatically mean that it's impossible to add backdoors to Linux packages. Only that it's more likely to be quickly detected. And that's when coupled with other things, such as package signatures and code reviews.

Most supply chain attacks these days are aimed at other targets, such as Python or Node packages, not the Linux distros. But that doesn't mean the effort was wasted - it means the distros are now a really tough target.

## Should you care?

As a user: not much you need to do, except appreciate that someone is doing the boring work that few people would even understand.

As someone who builds software: set `SOURCE_DATE_EPOCH`, don't embed build timestamps and hostnames if you don't have to, sort your file lists, use `-ffile-prefix-map`. It costs almost nothing, and makes your releases verifiable. Or make your app so popular that it gets into Debian, then the package maintainer will do that for you.

## Sources

- Ken Thompson, [Reflections on Trusting Trust](https://doi.org/10.1145/358198.358210), Communications of the ACM, 1984
- Russ Cox, [Running the "Reflections on Trusting Trust" Compiler](https://research.swtch.com/nih)
- [XcodeGhost](https://en.wikipedia.org/wiki/XcodeGhost) on Wikipedia
- [Reproducible Builds project](https://reproducible-builds.org/)
- [`SOURCE_DATE_EPOCH` specification](https://reproducible-builds.org/specs/source-date-epoch/)
- [diffoscope](https://diffoscope.org/)
- [Debian reproducibility status](https://tests.reproducible-builds.org/debian/reproducible.html)
- [Arch Linux reproducibility status](https://reproducible.archlinux.org/)
- [Fedora change aims for 99% package reproducibility](https://lwn.net/Articles/1014979/) (LWN, 2025) and the [Fedora change proposal](https://fedoraproject.org/wiki/Changes/Package_builds_are_expected_to_be_reproducible)
- [buildinfo files are now available from Launchpad](https://lists.snapcraft.io/archives/kernel-team/2024-January/148400.html) (Ubuntu kernel team list, 2024)
- [NixOS minimal ISO is 100% reproducible](https://discourse.nixos.org/t/nixos-unstable-s-iso-minimal-x86-64-linux-is-100-reproducible/13723) (NixOS Discourse, 2021)
- [The Full-Source Bootstrap](https://guix.gnu.org/en/blog/2023/the-full-source-bootstrap-building-from-source-all-the-way-down/) (Guix blog, 2023) and [bootstrappable.org](https://bootstrappable.org/)
- David A. Wheeler, [Countering Trusting Trust through Diverse Double-Compiling](https://dwheeler.com/trusting-trust/)
- Andres Freund, [backdoor in upstream xz/liblzma](https://www.openwall.com/lists/oss-security/2024/03/29/4) (oss-security, 2024)
- [CVE-2024-3094](https://nvd.nist.gov/vuln/detail/CVE-2024-3094) at NVD
