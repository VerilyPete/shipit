+++
date = '2025-10-28T00:07:00-05:00'
draft = false
title = 'Containers'
+++
# Apple’s Native Containers in macOS 26

_Author’s note: Apple released an update to their container runtime overnight, putting it at a_ [_v0.6.0 release_](https://github.com/apple/container/releases/tag/0.6.0)_. (_[_Changelog_](https://github.com/apple/container/compare/0.5.0...0.6.0) _…oooh, support for subnets!) I’ll put that through the same tests and report back here._

With all the buzz around Liquid Glass in Tahoe, it’s easy to miss that Apple quietly introduced its own container runtime in the latest macOS release.

### Containers?

If you’re reading this, you’re probably somewhat familiar with containers. Just in case you aren’t, I’ll give a condensed explanation.

Containerization is the practice of packaging an application together with all of its underlying dependencies into a tidy bundle that can be distributed and run on any platform with supported architecture. Namespaces and cgroups (both features of the Linux kernel) confine what the container can see and keep its resource usage within specified limits. The result is an application running in an isolated environment without the associated bloat of virtualizing the entire operating system.

Containers need a Linux kernel since they’re using Linux kernel features. In the past, that meant that in Docker (the most popular containerization platform) on macOS you were running a single Linux VM and all of your containers lived within it.

!\[A Docker VM with three containers inside\](dockervm.jpg)

When I say “Docker” for the purpose of the post, I’ll be referring to [Docker CLI](https://www.docker.com/products/cli/) running as the management layer, backed by [Colima](https://github.com/abiosoft/colima) handling the underlying virtualization.  
  
Colima leans on the Apple native [Virtualization.framework](https://developer.apple.com/documentation/virtualization) that premiered in macOS 11 to run the Linux VM that hosts the containers for Docker. At launch, you specify how much memory to allocate to Colima’s VM container host and that memory is immediately allocated, although you’ll only be using a fraction of it through the magic of Virtualization.framework’s [memory balloon device](https://developer.apple.com/documentation/virtualization/vzvirtiotraditionalmemoryballoondevice).

* * *

### What makes Apple’s container runtime different?

_A note on nomenclature: Apple named their API and CLI tools for working with Linux containers a bit generically._ [_The framework for working with containers is called containerization;_](https://github.com/apple/containerization) _the actual tool itself is called_ [_container_](https://github.com/apple/container)_. Apple’s container runtime requires macOS 15+ and Apple Silicon to operate, with macOS 26 required for full functionality._

With the CLI tool container, Apple is blurring the lines between traditional containerization and virtualization. Instead of hosting containers in a shared VM, Apple has chosen to give each container its own virtual machine - complete with its own ext4-based block storage, IP address, and user-definable CPU & memory limits. On M3 or newer hardware, you can even [nest virtualization inside of a container.](https://github.com/apple/container/blob/main/docs/how-to.md#expose-virtualization-capabilities-to-a-container) (We’re all thinking the same infinite recursion joke, right?)

!\[Apple container with 3 VMs isolated\](containervm.jpg)

The first time you launch the container system via ‘container system start’, you’ll notice something: there’s zero delay. With no host VM to launch, you press enter and you’re immediately back at a prompt. It’s a change compared to the ~15 second wait of starting Colima and had me wondering if something had gone wrong the first time I ran it.

  
Much like Colima on Apple Silicon, Apple uses their Virtualization.framework to host these per-container virtual machines. Apple containers don’t share resources, so you can assign resources per-container independently instead of accounting for total consumption at a shared VM layer or risking OOMKills. The containerization framework also uses [memory balloon devices](https://developer.apple.com/documentation/virtualization/vzvirtualmachineconfiguration/memoryballoondevices) in its implementation of Virtualization.framework and manages the allocated real memory more aggressively than Colima. See below: those are two instances of the same container, one hosted by Colima in Virtualization.framework, the other by Apple’s native container runtime.

!\[Activity Monitor showing two VM processes\](limavscontainermem.jpg)

Apple’s container runtime follows the [Open Containers Initiative](https://opencontainers.org) image & runtime spec. This means your current Docker/Podman/Kubernetes images & registries are compatible. Apple’s Rosetta 2 translation layer is also supported, allowing you to run amd64 images on Apple Silicon through the use of the —platform flag.

When the container runtime launches for the first time, it will fetch the kernel that it uses when running container images. In version 0.50 (current as of this writing), it uses a [v6.12.28 aarch64 Linux Kernel](https://cdn.kernel.org/pub/linux/kernel/v6.x/ChangeLog-6.12.28) from the [Kata Containers](https://katacontainers.io) v3.17.0 release in May of 2025. Kata Containers is a project that aims to create container runtimes that deliver “The speed of containers, the security of VMs” to quote their page. No surprise that Apple is leaning on their work to deliver their goal of sub-second container launches.

* * *

### Using the container runtime

If you’re familiar with Docker’s CLI syntax, you’ll feel right at home. The team behind container has kept the syntax almost entirely unchanged. I’ve read accounts of people swapping Docker for Podman and aliasing “docker” to “podman” and going on with their lives without encountering any issues. That would _almost_ work here too. (“container run —help | wc -l” returns a count of 70 lines. The same command on Docker returns 112 lines, so you may find some options missing.)

The one notable difference is that when launching the container, you’re configuring the CPU and memory count of the VM. If you aren’t happy with a given container’s baked-in defaults, you need to [add those flags.](https://github.com/apple/container/blob/main/docs/command-reference.md)

I mentioned that Apple’s goal is sub-second launch times on these containers, and I’m seeing that in my own testing. I tossed together a simple script that runs an Alpine Linux container and then immediately opens a shell and exits that container.

I’m seeing 1.2 second cold starts and 0.8 second warm starts when I run that script.

* * *

### Some Benchmarks

I don’t intend for these to be authoritative benchmarks; I wanted to do some rough performance comparisons between the Apple container runtime and Docker. I’ll also provide all the details of the hardware, tools, and environment that I used along with a Gist showing the exact syntax and text output in case anyone wants to double-check my work.

Hardware: M1 Pro with 32GB of RAM running macOS 26.0.1 on the internal SSD.

Versions: Apple container runtime v0.5.0, Docker CLI 28.5.1 (client), 28.4.0 (server) and Colima 0.9.1

Containers used for benchmarking: stress-ng, fio, 7zip

Colima’s VM was started with 4 CPUs & 3GB of RAM to allow 2GB for containers with overhead for the OS. I specified 4 CPUs and 3GB of RAM for container run. An interesting thing: stress-ng reported 2.2GB of available memory running under Docker, but 2.76GB of available memory when run under container.

[Link to container/Docker run syntax used and benchmark output as text.](https://gist.github.com/VerilyPete/bf2f09070c9539e87a44646f8374109e)

[Link to Gist with Python script comparing Docker and container startup and lifecycle.](https://gist.github.com/VerilyPete/98a2281bdb5f8f336f732c69f3e338a9)

We’ll start off with a small suite of simple benchmarks that I created in Cursor to test the following: pulling a container, container cold start, warm start, lifecycle (create, run, rm), starting 5 containers in parallel, and creating 1000 temp files. You can find the script in the Gist above.

!\[Docker had the advantage\](startup\_performance.jpg)

My takeaways from this chart:

- Docker pulls containers much faster.
- Docker’s much slower at creating containers, which leads to the Lifecycle result.
- Every other test shows the advantage of already having that VM running when you start the container. Apple’s native container runtime reliably returned sub-second starts as advertised.

* * *

Moving on to the stress-ng, fio & 7zip benchmarks. You can find the exact syntax used to launch these benchmarks and the output in the first Gist above.

[stress-ng](https://github.com/ColinIanKing/stress-ng) is a Linux stress-testing tool for system burn-in and benchmarking. It has hundreds of built-in methods of abusing a machine. I chose to use a couple of tests that exercise I/O, CPU utilization, and memory read/write speed.

We’ll start with stress-ng running its “iomix” I/O suite on 4 workers simultaneously for 30 seconds.

!\[stress-ng's iomix results\](iomix\_4\_workers.png)

For this test, the native container runtime made better use of its CPU and trounced Docker. It used 73% more of its CPU to reliably return 31x more I/O Mix operations.

The next test has stress-ng running 4 CPU stressors and 2 memory stressors in parallel for 30 seconds.

!\[stress-ng memory & cpu test\](stress\_ng\_memory\_cpu.png)

- CPU performance was evenly matched, while memory performance strongly favored the native implementation.
- The native CPU results seemed silly. If you check the text output, it’s actually 0.02% system time. The CPU operations of this test run almost entirely in userspace, so the low kernel utilization makes sense.

* * *

[Fio](https://github.com/axboe/fio) is the Flexible I/O tester. Its raison d’être is running reproducible I/O load testing in any number of shapes. I criminally underutilized it with the pair of tests that I ran, starting with this test that performed 10 seconds of 4k reads on a single worker.

!\[fio randomized 4k read\](fio\_randread\_comparison.png)

I can’t explain an interesting shift that we see here. Fio shows the opposite of the previous results, with Docker performing substantially better than the native container runtime.

That result holds up as we move on to a test that performs a mix of 70% read and 30% write on 8k blocks across four simultaneous workers.

!\[Mixed read/write\](mixed\_rw\_fio\_comparison.png)

* * *

The final test I ran was 7zip’s “b” benchmark, where we hammered on the CPU as it compressed and then decompressed data with the LZMA algorithm.

!\[7zip test\](7zip\_benchmark\_comparison.png)

This final test supports the earlier CPU-based benchmarks with very similar results. In every category, we’re near (or under) 1% difference.

* * *

### Benchmarking Takeaways

It’s interesting that stress-ng and Fio reliably offered such different results. I’ll continue digging into what might be responsible for the performance between the two suites.

CPU performance was very evenly matched between the two runtimes. Memory-based benchmarks favored Apple’s implementation and I/O based operations strongly favored Docker.  
  
Even with that disparity in performance, I’m confident that Apple’s new container runtime is suitable as a drop-in replacement for most simple local development tasks where you’d normally reach for Docker. The lack of mature tooling like Buildx, Docker Compose, and Kubernetes means that Docker still has a secure place in the ecosystem it created, but Apple will continue to narrow that gap with new releases.

* * *

### Final thoughts

Overall, Apple’s native runtime feels promising. It offers levels of isolation previously only seen in much bulkier VMs, solid CPU and memory performance, somewhat inconsistent I/O performance, and delightfully low memory utilization with running containers - and near zero utilization when those containers are stopped.

Not bad for v0.5.0.

I’m watching the GitHub repo and planning to adopt the native container runtime for use with the dev containers plugin in Cursor/VSCode just as soon as they resolve the few remaining [issues.](https://github.com/microsoft/vscode-remote-release/issues/11012)