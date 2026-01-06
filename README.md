# Sched_ext Schedulers and Tools

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/sched-ext/scx-c-examples)

**IMPORTANT**: This repository includes example schedulers used for illustration purposes, or as a jumping-off point for 
prototyping. The definitive versions of all chedulers in this repository are in the upstream kernel under the `tools/sched_ext`
directory. Please do not submit PRs for new features for the schedulers here. If you find any issues or bugs, please
submit a patchset in the upstream sched\_ext mailing list.

[`sched_ext`](https://github.com/sched-ext/scx) is a Linux kernel feature
which enables implementing kernel thread schedulers in BPF and dynamically
loading them. This repository contains various scheduler implementations and
support utilities.

`sched_ext` enables safe and rapid iterations of scheduler implementations, thus
radically widening the scope of scheduling strategies that can be experimented
with and deployed; even in massive and complex production environments.

You can find more information, links to blog posts and recordings, in the [wiki](https://github.com/sched-ext/scx/wiki).
The following are a few highlights of this repository.

- The [`scx_layered` case
  study](https://github.com/sched-ext/scx/blob/case-studies/case-studies/scx_layered.md)
  concretely demonstrates the power and benefits of `sched_ext`.
- For a high-level but thorough overview of the `sched_ext` (especially its
  motivation), please refer to the [overview document](OVERVIEW.md).
- For a description of the schedulers shipped with this tree, please refer to
  the [schedulers document](scheds/README.md).
- The following video is the [`scx_rustland`](https://github.com/sched-ext/scx/tree/main/scheds/rust/scx_rustland)
  scheduler which makes most scheduling decisions in userspace `Rust` code showing
  better FPS in terraria while kernel is being compiled. This doesn't mean that
  `scx_rustland` is a better scheduler but does demonstrate how safe and easy it is to
  implement a scheduler which is generally usable and can outperform the default
  scheduler in certain scenarios.

[scx_rustland-terraria](https://github.com/sched-ext/scx/assets/1051723/42ec3bf2-9f1f-4403-80ab-bf5d66b7c2d5)

`sched_ext` is supported by the upstream kernel starting from version 6.12. Both
Meta and Google are fully committed to `sched_ext` and Meta is in the process of
mass production deployment. See [`#kernel-feature-status`](#kernel-feature-status) for more details.

In all example shell commands, `$SCX` refers to the root of this repository.

## Getting Started

All that's necessary for running `sched_ext` schedulers is a kernel with
`sched_ext` support and the scheduler binaries along with the libraries they
depend on. Switching to a `sched_ext` scheduler is as simple as running a
`sched_ext` binary:

```bash
root@test ~# cat /sys/kernel/sched_ext/state /sys/kernel/sched_ext/*/ops 2>/dev/null
disabled
root@test ~# scx_simple
local=1 global=0
local=74 global=15
local=78 global=32
local=82 global=42
local=86 global=54
^Zfish: Job 1, 'scx_simple' has stopped
root@test ~# cat /sys/kernel/sched_ext/state /sys/kernel/sched_ext/*/ops 2>/dev/null
enabled
simple
root@test ~# fg
Send job 1 (scx_simple) to foreground
local=635 global=179
local=696 global=192
^CEXIT: BPF scheduler unregistered
```

[`scx_simple`](https://github.com/sched-ext/scx/blob/main/scheds/c/scx_simple.bpf.c)
is a very simple global vtime scheduler which can behave acceptably on CPUs
with a simple topology (single socket and single L3 cache domain).

Above, we switch the whole system to use `scx_simple` by running the binary,
suspend it with `ctrl-z` to confirm that it's loaded, and then switch back
to the kernel default scheduler by terminating the process with `ctrl-c`.
For `scx_simple`, suspending the scheduler process doesn't affect scheduling
behavior because all that the userspace component does is print statistics.
This doesn't hold for all schedulers.

In addition to terminating the program, there are two more ways to disable a
`sched_ext` scheduler - `sysrq-S` and the watchdog timer. Ignoring kernel
bugs, the worst damage a `sched_ext` scheduler can do to a system is starving
some threads until the watchdog timer triggers.

As illustrated, once the kernel and binaries are in place, using `sched_ext`
schedulers is straightforward and safe. While developing and building
schedulers in this repository isn't complicated either, `sched_ext` makes use
of many new BPF features, some of which require build tools which are newer
than what many distros are currently shipping. This should become less of an
issue in the future. For the time being, the following custom repositories
are provided for select distros.

## Install Instructions by Distro

- [Ubuntu](INSTALL.md#ubuntu)
- [Arch Linux](INSTALL.md#arch-linux)
- [Gentoo Linux](INSTALL.md#gentoo-linux)
- [Fedora](INSTALL.md#fedora)
- [Nix](INSTALL.md#nix)
- [openSUSE Tumbleweed](INSTALL.md#opensuse-tumbleweed)

## Repository Structure

```
scx
|-- scheds               : Sched_ext scheduler implementations
|   |-- include          : Shared BPF and user C include files
|   |-- vmlinux          : vmlinux.h header
|   |-- c                : Example schedulers - userspace code written C
```

## Build & Install

Use `make` to build all the schedulers in this repo.

**Dependencies:**

- `clang`: >=16 required, >=17 recommended
- `libbpf`: >=1.2.2 required, >=1.3 recommended
- `bpftool`: Usually available in `linux-tools-common` or similar packages
- `libelf`, `libz`, `libzstd`: For linking against libbpf
- `pkg-config`: For finding system libraries

The kernel has to be built with the following configuration:

- `CONFIG_BPF=y`
- `CONFIG_BPF_SYSCALL=y`
- `CONFIG_BPF_JIT=y`
- `CONFIG_DEBUG_INFO_BTF=y`
- `CONFIG_BPF_JIT_ALWAYS_ON=y`
- `CONFIG_BPF_JIT_DEFAULT_ON=y`
- `CONFIG_SCHED_CLASS_EXT=y`

The [`scx/kernel.config`](./kernel.config) file includes all required and other recommended options for using `sched_ext`.
You can append its contents to your kernel `.config` file to enable the necessary features.

### Building and Installing

```shell
$ cd $SCX
$ make all                          # Build all C schedulers
$ make install INSTALL_DIR=~/bin    # Install to custom directory
```

Binary location is `build/scheds/c/scx_simple`

### Environment Variables

Both `make` and `cargo` support these environment variables for BPF compilation:

- `BPF_CLANG`: The clang command to use. (Default: `clang`)
- `BPFTOOL`: The bpftool command to use. (Default: `bpftool`)
- `CC`: The C compiler to use. (Default: `cc`)

**Examples:**

```shell
# Use specific clang version for C schedulers
$ BPF_CLANG=clang-17 make all

# Use specific clang version for Rust schedulers
$ BPF_CLANG=clang-17 cargo build --release

# Use clang for C compilation and system bpftool
$ CC=clang BPFTOOL=/usr/bin/bpftool make all
```

## Checking scx_stats

With the implementation of `scx_stats`, schedulers no longer display statistics by default. To display the statistics from the currently running scheduler, a manual user action is required.
Below are examples of how to do this.

- To check the scheduler statistics, use the

```shell
$ scx_SCHEDNAME --monitor $INTERVAL
```

for example `0.5` - this will print the output every half a second

```shell
$ scx_bpfland --monitor 0.5
```

Some schedulers may implement different or multiple monitoring options. Refer to `--help` of each scheduler for details.
Most schedulers also accept `--stats $INTERVAL` to print the statistics directly from the scheduling instance.

## [Developer Guide](./DEVELOPER_GUIDE.md)

Want to learn how to develop a scheduler or find some useful tools for working
with schedulers? See the developer guide for more details.


## Getting in Touch

We aim to build a friendly and approachable community around `sched_ext`. You
can reach us through the following channels:

- `GitHub`: https://github.com/sched-ext/scx
- `Discord`: https://discord.gg/b2J8DrWa7t
- `Mailing List`: sched-ext@lists.linux.dev (for kernel development)

We also hold weekly office hours every Tuesday. Please see the `#office-hours`
channel on `Discord` for details.

## Additional Resources

There are articles and videos about `sched_ext`, which helps you to explore
`sched_ext` in various ways. Following are some examples:

- [`Sched_ext` YT playlist](https://youtube.com/playlist?list=PLLLT4NxU7U1TnhgFH6k57iKjRu6CXJ3yB&si=DETiqpfwMoj8Anvl)
- [LWN: The extensible scheduler class (February, 2023)](https://lwn.net/Articles/922405/)
- [arighi's blog: Implement your own kernel CPU scheduler in Ubuntu with `sched_ext` (July, 2023)](https://arighi.blogspot.com/2023/07/implement-your-own-cpu-scheduler-in.html)
- [David Vernet's talk : Kernel Recipes 2023 - `sched_ext`: pluggable scheduling in the Linux kernel (September, 2023)](https://youtu.be/8kAcnNVSAdI)
- [Changwoo's blog: `sched_ext`: a BPF-extensible scheduler class (Part 1) (December, 2023)](https://blogs.igalia.com/changwoo/sched-ext-a-bpf-extensible-scheduler-class-part-1/)
- [arighi's blog: Getting started with `sched_ext` development (April, 2024)](https://arighi.blogspot.com/2024/04/getting-started-with-sched-ext.html)
- [Changwoo's blog: `sched_ext`: scheduler architecture and interfaces (Part 2) (June, 2024)](https://blogs.igalia.com/changwoo/sched-ext-scheduler-architecture-and-interfaces-part-2/)
- [arighi's YT channel: `scx_bpfland` Linux scheduler demo: topology awareness (August, 2024)](https://youtu.be/R-FEZOveG-I)
- [David Vernet's talk: Kernel Recipes 2024 - Scheduling with superpowers: Using `sched_ext` to get big perf gains (September, 2024)](https://youtu.be/Cy7-oqdcUCs)
- [arighi's talk: Kernel Recipes 2025 - Schedule Recipes (September, 2025)](https://youtu.be/NEwCs7EqAbU)
