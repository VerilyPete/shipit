+++
date = '2025-11-04T13:42:47-06:00'
draft = false
title = 'Container Follow Up'
+++

[cover]
image = 'ian-taylor-jOqJbvo1P9g-unsplash.jpg'
alt = 'Definitely not an Apple VM.'
relative = true

# Follow up on Apple’s container runtime.
![Clearly a non-apple VM](ian-taylor-jOqJbvo1P9g-unsplash.jpg)
I wrote a post about Apple’s container runtime that’s new in macOS 26. Afterward, I saw several questions pop up multiple times, so I thought I’d answer them.

_“How does Apple make these VMs boot in under a second?”_

There’s a [video from WWDC](https://www.youtube.com/watch?v=JvQtvbhtXmo) that explains it nicely, but here’s the condensed version. The VM that hosts each container has been stripped of _anything_ that isn’t required to support a container. No libraries (including libc), no utilities, and no support for legacy hardware. Also, the container runtime is macOS-only, so it expects to lean on many of the OS features to support networking, service management, interprocess communication, logging, and more.

The first time you run the container system, it fetches a precompiled kernel from Kata Containers and puts it into:  
  
`~/Library/Application Support/com.apple.container/kernels`

This results in a 14.8M vmlinux kernel image that Virtualization.framework loads.

After compilation, the container runtime creates a bare ext4 filesystem. Then, it cross-compiles vminitd via the Swift Static Linux SDK with musl libc linked inside. This gives vminitd the system call wrappers and libc functions needed to talk to a Linux kernel from a Swift binary.

It’s just a kernel, a bare filesystem, and a single-file init (vminitd as PID 1). That minimalism delivers sub-second boots and a much smaller attack surface. Since that can look VM-like, it’s fair to ask: is it really a container?

* * *

“_Each container has its own kernel, filesystem & network. That sounds like a VM. Is it really a container_?”

In the [previous post](https://shipit.peterhollmer.com/posts/containers/), I mentioned that when we talk about containers, we’re talking about using features of the Linux kernel to isolate processes and control resource consumption. There’s always going to be a VM hosting that kernel on macOS. The real difference is whether we’re using a stripped-down environment per-container or a larger virtualized instance of an OS to host one or more containers.

Additionally, the VM used by the container runtime is optimized to _do the minimum necessary to support a container -_ and not much else_._ This VM is built to do one thing only, and it’s useless unless it’s hosting a container. That’s the tradeoff for increased security via minimal attack surface area and sub-second boot times.

This design choice has implications for hardware-access features such as GPU passthrough.

* * *

_“Does this support GPU passthrough?”_

The default kernel is a Kata kernel, and Kata supports GPU passthrough for certain Intel and Nvidia GPUs. It is natural to hope that Apple’s container runtime would support GPU passthrough to allow containers to utilize MLX and unleash the M-series GPU and memory bandwidth.

There’s a discussion on the [Apple container GitHub repo](https://github.com/apple/container/discussions/62) that’s been going for several months. Users are explaining their use cases and [engineers from other projects](https://github.com/apple/container/discussions/62#discussioncomment-13657155) are explaining the inherent difficulty. From my reading of the thread, it feels like GPU passthrough is not under serious consideration for the Apple container runtime just yet.

A reply on LinkedIn from Landon Clipp might supply an additional reason why Apple’s not in a hurry to add GPU passthrough. Landon has extensive experience using Kata Containers with GPU passthrough and he called out: “_Kata VMs suffer with really poor boot times with GPU passthrough on DGX/HGX platforms because of (what I suspect) is the MMIO mapping step between guest physical addresses and host physical addresses for these large BAR GPUs taking a long time._” If sub-second boot times are one of the project’s goals, GPU passthrough could be a setback in that regard.

For now, Apple’s container runtime is a fast, secure way to run Linux containers on macOS with minimal overhead. If your workload depends on GPU passthrough, you’ll likely need to keep that piece on Linux or use native macOS frameworks. I’ll keep watching the project and share an update if anything changes.