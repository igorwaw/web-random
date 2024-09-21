Title: Compiling Linux kernel for fun and profit
Date: 2024-05-17 19:00
Status: draft
Category: random
Tags: testing, linux
Slug: compiling-linux-kernel

Since almost the beginning of my Linux journey and up to around the year 2005, I used a custom kernel, both on my own computer and on the servers I managed. Why did I do that and why did I stop? 


| Why?      | Why not? |
| ----------- | ----------- |
| You can optimize your kernel for the specific CPU it will be running on.     |  In the early 2000s it had a noticable impact, not so much on today's CPUs. Plus, it only makes kernel code faster - how much your CPU spends on kernel code?      |
| You can make your kernel much smaller by disabling unneeded features, half of the standard size is not unusual. Kernel is always loaded into RAM, it can't be swapped out.   | When PCs typically had 4-16MB of RAM, the kernel was a sizable portion of it. Today, wasting a few MB is not a big deal. Plus, most things like hardware drivers or filesystems are built as modules, only loaded as needed, so you have to disable whole classes of hardware to see any impact.       |
| You can add extra patches. Typically, I used to add security patches such as openwall or grsecurity, sometimes also hardware drivers. | The security patches I used to use were mostly absorbed by the mainline Linux kernel. And I haven't seen a driver distributed as a kernel source patch for years.  |
| Disabling some kernel features reduces an attack surface and the amount of (possibly buggy) code running. | The more unneeded features you disable, the more likely you are to discover that they were, in fact, needed. You might think you don't need a certain feature, but the software you use might rely on it. You probably never explicitly used multicast or namespaces, but I bet your system uses them.   |
| You can run the newest kernel with shiny features instead of an ancient one provided by your distribution. |  Like any software, Linux kernel has some compatibility requirements. Too new kernel will not run or compile with an old distro. You either need to check the documentation or risk discovering it the hard way.  |
|   |  Customizing and compiling kernel takes at least half an hour of active work, plus more time (up to few hours on a slow computer) for compilation, and you need to repeat it every time a new kernel is released. |

## Why is kernel compiling still relevant today?

One reason is education. You can really learn a lot about the inner working of your system just by looking at the kernel configuration options (not even the source code). In fact, I occasionaly do just the configuration step without actually compiling and using the custom kernel, which gives me 99% of education with 1% of the problems.

Another is to help with debugging. Unless you're actively working on kernel development, discovering a bug that shows up in your specific setup is unlikely. But not impossible. In my 25 years of working with Linux, it only happened once. When it happens and you decide to contact relevant developers, they might ask you to test with different kernel versions or do some quick modifications to see how they impact your problem.

But the most likely reason is embedded hardware. All the arguments about CPU speed and RAM size are relevant again. On a router or a similar device that deals with IP stack and network drivers all the time, CPU spends most time running the kernel code, so any speedup of it is welcome. Plus, there's even no such thing as a univeral ARM kernel anyway. In a PC world, the same kernel can run on a pocket sized Intel Atom board and on a 512-core NUMA supercomputer node, but embedded hardware is even more varied than that.

## How hard is that?

Easier than it seems. You don't need to know anything about C programming and build systems. Some knowledge about computer hardware, network, filesystems etc. is needed if you want to customize kernel features.

One caveat is Secure Boot. Most PCs have this feature enabled, meaning they will only allow to boot systems considered secure. The whole path needs to be trusted:

- UEFI, the system firmware, loads shim, first-stage bootloader (signed by Microsoft),
- shim loads grub (signed by Canonical, Debian, RedHat or whoever created your distro),
- grub loads kernel,
- kernel loads modules.

And you probably guessed: kernel and modules need to be signed as well, those provided by distribion already are and grub is configured to trust their signing key.

The "proper" way to deal with it is to generate a keypair, cryptographically sign your custom kernel and modules and configure grub to trust your key. Possible, but increases the difficulty of the whole process tenfold. The easy way, especially if you're just doing a quick test, is to disable Secure Boot.

For the embedded device things are a little bit different. You need to setup a cross-compiling environment, which used to be a pain, but became much easier with projects such as Yocto that aim to automate the build process. Still, it's a different thing, so I'll focus on compiling for a PC this time.

## Getting the kernel source

There are several ways to get the source code. Your distribution might provide the package, but if you're going through the hurdle of compiling the kernel, you probably want to get the newest one. You can download a tar.xz package from [kernel.org](https://kernel.org) and unpack it with `tar -xvJf`. If you only need one version, either to run on your embedded board or just to have a look at configuration options, that's the way to go.

However, if you're debugging a problem, developers will likely ask you to try several versions. Or to apply a patch and then revert it. Linux kernel is stored with git (in fact, git was written for this purpose) and you can clone it just like any other repo: `git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git`. But you probably don't need full history since the invention of git. To limit download for example to last 6 months, use:

```bash
git clone --no-checkout --depth 1 -b master  https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
git fetch --shallow-since='6 months' origin
git checkout --force --detach origin/master
```

This will first do a shallow clone (latest version only), then add 6 months worth of latest changes and finally checkout the latest version. If later you want to switch to a specific version, do `git checkout --force --detach v6.9-rc3` (careful, it will overwrite your local changes) - use `git tag` to see a list of available versions.

Shallow clones are not recommended for long term work. If you try to add more versions, you'll probably see an error that something can't be unshallowed. Even commands like `git log` or `git blame` might fail or give strange results. If you're not a kernel developer and it's just a one-off task, it's probably OK and you just saved gigabytes of download. And if you are a kernel developer, you know it already.

Whether you choose a tar.xz file or a git clone, download it your home directory, or any other place writable by your standard account. If you're as old as me, you might remember that Linux distros placed kernel sources in /usr/src/linux, but it's no longer recommended and hasn't been for years. The reason is you shouldn't use a root account for the compilation.

## Installing the tools

First approximation: you only need GNU make and the C compiler. But if you look at it more closely:

- I prefer to use Debian or Ubuntu, meaning I'm going to build a deb package, which requires at least dpkg-dev and realistically a few more tools. (Sidenote: if you're reading another guide and it mentions **kernel-package** or **make-kpkg** command, be warned that it's a bit outdated. Might be still useful though, just skip that part and read below about the current way of building packages)
- You might need to get the code from git or install a patch.
- Very likely you'll need compression libriaries.
- If you want to configure kernel with `make menuconfig` (my preferred way), you need header files for the NCurses library.
- Depending on the exact configuration, some more tools or libraries might be needed.
- Note that I didn't include any Rust tools here. Moving to Rust is a slow process, currently it's at an experimental stage, if you're new to kernel compilation, you shouldn't touch it.

A good starting point for Debian or Ubuntu is: `sudo apt install build-essential debhelper dkms libncurses-dev git fakeroot libssl-dev libelf-dev pahole bison flex liblzma-dev libbz2-dev`

Package build-essential depends on make, C and C++ compilers and some essential header files; debhelper will fetch dpkg development tools; fakeroot is also needed for building deb package; git is obvious; libncurses-dev is needed for *make menuconfig*; the rest of tools and headers might not be needed if some kernel features are not enabled, but honestly it's just easier to install a bit too much. It might look like the list is missing many packages, but it will actually pull a lot of dependencies. You might be still missing something required for a specific kernel feature, but it should be easy to find.

## Patching the kernel source

If you downloaded a .tar.xz file, there's no need to redownload the whole package when the new kernel is released. You can get a much smaller patch file. Sometimes you might also need a patch for some extra functionality or driver. Or a kernel developer might prepare a patch for you to try.

Regardless of why you need it, the procedure is similar. Copy the file to your kernel source directory and type: `patch -p1 < name-of-patch-file`.

The parameter means: remove the first directory from the file path. The convention is to prepare the patches in such way that the first directory is 'linux' or 'linux-5.9.3' or something like that - meaning it's one level above the kernel sources dir, which might be named differently on your computer than on the developer's system. If in doubt, you can try `patch --dry-run -p1 < name-of-patch-file`, if you see errors, adjust the number.

## Configuring the kernel

Here's where the paths diverge. Go to your kernel source directory and:

- First, edit/create file *localversion*, add something like '-my1', this string will be appended to the kernel version. Not required, but makes it easy to differentiate between the kerneles, especially if you recompile the same version with a different config. Plus, if you're compiling the same kernel version as you're currently running, you want your custom kernel to have a different version string so you don't replace the running kernel by mistake (note, some people would test a kernel in a VM, I'm assuming you're running it directly on your main machine because YOLO).
- If you want to use your current running kernel as a starting point, do `cp /boot/config-$(uname -r) .config`, if you want to use the defaults from kernel.org as a starting point, skip this step. Note that if you're new kernel is close to the kernel you're already using, it's more likely to work properly.
- If you copied the running kernel config but are now compiling a different (newer) version, you can now run `make oldconfig`, this will ask you how to configure new features but skip all those already present in the config file - which usually means just a few questions.
- Another possible option is `make localmodconfig`. It will only enable the modules that are currently loaded, greatly decreasing the kernel package size and the build time. But it will also mean you might be missing a driver for a device you sometimes connect, just not at the point when you ran this command; or a network feature that weren't used since the last reboot but will be needed tomorrow. Dangerous if you're really want to use this kernel, probably OK for a quick test.
- Finally, if you want to see all configuration options - either because you want to properly customize your kernel, or you just want to have look - run `make menuconfig` for ncurses (text-mode GUI) interface. There are other interfaces (xconfig for qt, gconfig for gtk or config for the simple text-mode questions) but I find this one the most comfortable.

Note that in every configuration interface you will have the possibility to read help. It provides useful information, which often ends with "If unsure, say Y". You should be careful with the recommendations, especially if they recommend to say N, as they are often very outdated. For example, it recommends to disable Symmetric Multi-Processing, while vast majority of CPUs today are multicore; it says you probably don't need Control Groups, but they are heavily used by systemd.

## Compiling and installing the kernel

In the past, you would use several **make** commands to compile kernel and modules, then copy files to the relevant places (/boot and /lib/modules). That worked, but it completely bypassed distributions's package manager. Another option was to a script prepared by the distribution maintainers, eg. make-kgpg on Debian, to create a package. But for a long time, an option to create deb and rpm packages is built into kernel's Makefile.

On Debian or Ubuntu, simply run `make bindeb-pkg` (or, to make use of multi-core, `make -j $(nproc --all) bindeb-pkg`). This will create several deb files in the parent directory, containing the kernel itself, header files, optionally firmware and debug information. Similarly, on rpm-based distribution, use `make -j $(nproc --all) binrpm-pkg`. Then simply install packages with your usual way, eg. `dpkg -i package-name.deb`.

## Dealing with errors

If the compilation process results in an error, see if you can determine the cause. Often you're just missing a development tool or a header file, in such case simply install what's missing.

Linux kernel is a complex piece of software, therefore it requires a lot of tools to build - and it requires a lot from those tools. The special point of incompatibility is the C compiler. If you ever learned C, you heard of "undefined behaviour" - in some cases the language specification doesn't decree what's the result of a specific language construct, different compilers or hardware architectures give different results. It's usually not a big issue, until you try to compile a huge pile of C files that deal directly with hardware. Which is why kernel source comes with a list of recommended software versions in `Documentation/process/changes.rst`. The file says it's a list of minimal versions, but - especially in case of gcc - too new version may also result in compilation error.

By default, all compilation warnings are treated as errors (they break the build). It's a policy that kernel build should not cause any compiler warnings. But that's only true if you use a supported version of compilers and tools and don't enable any experimental or obsolete drivers - obviously it's impossible to test every combination. If you believe your error is actually a harmless warning, you might run your kernel configuration again, go to "General Setup" and near the top, disable option "Compile the kernel with warnings as errors".

In any case, simply running the build process again should quickly go through the parts that were compiled before and pick up from where it broke before. But if you believe you made a mess, run `make clean` before the build command to delete all compiled files.

## More info

- https://debian-handbook.info/browse/stable/sect.kernel-compilation.html
- https://docs.kernel.org/admin-guide/quickly-build-trimmed-linux.html
