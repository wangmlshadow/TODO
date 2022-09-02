> https://github.com/spdk/spdk

# What is SPDK

> [SPDK: What is SPDK](https://spdk.io/doc/about.html)

The Storage Performance Development Kit (SPDK) provides a set of tools and libraries for writing high performance, scalable, user-mode storage applications. It achieves high performance through the use of a number of key techniques:

Moving all of the necessary drivers into userspace, which avoids syscalls and enables zero-copy access from the application.

Polling hardware for completions instead of relying on interrupts, which lowers both total latency and latency variance.

Avoiding all locks in the I/O path, instead relying on message passing.

The bedrock of SPDK is a user space, polled-mode, asynchronous, lockless [NVMe](http://www.nvmexpress.org/) driver. This provides zero-copy, highly parallel access directly to an SSD from a user space application. The driver is written as a C library with a single public header. See [NVMe Driver](https://spdk.io/doc/nvme.html) for more details.

SPDK further provides a full block stack as a user space library that performs many of the same operations as a block stack in an operating system. This includes unifying the interface between disparate storage devices, queueing to handle conditions such as out of memory or I/O hangs, and logical volume management. See [Block Device User Guide](https://spdk.io/doc/bdev.html) for more information.

Finally, SPDK provides [NVMe-oF](http://www.nvmexpress.org/nvm-express-over-fabrics-specification-released), [iSCSI](https://en.wikipedia.org/wiki/ISCSI), and [vhost](http://blog.vmsplice.net/2011/09/qemu-internals-vhost-architecture.html) servers built on top of these components that are capable of serving disks over the network or to other processes. The standard Linux kernel initiators for NVMe-oF and iSCSI interoperate with these targets, as well as QEMU with vhost. These servers can be up to an order of magnitude more CPU efficient than other implementations. These targets can be used as examples of how to implement a high performance storage target, or used as the basis for production deployments.

> 存储性能开发套件 （SPDK） 提供了一组工具和库，用于编写高性能、可扩展的用户模式存储应用程序。它通过使用一些关键技术实现高性能：
>
> * 将所有必要的驱动程序移动到用户空间，从而避免 syscalls，并允许从应用程序中访问零拷贝。
> * 对硬件进行完成轮询，而不是依赖中断，这降低了总延迟和延迟方差。
> * 避免 I/O 路径中的所有锁，而是依靠消息传递。
>
> ‎SPDK 的基石是用户空间、轮询模式、异步、无锁 ‎[‎NVMe‎](http://www.nvmexpress.org/)‎ 驱动程序。这提供了从用户空间应用程序直接对 SSD 进行零拷贝、高度并行访问。驱动程序编写为具有单个公共标头的 C 库。有关更多详细信息‎[‎，请参阅 NVMe 驱动程序‎](https://spdk.io/doc/nvme.html)‎。‎
>
> ‎SPDK还提供了一个完整的块堆栈作为用户空间库，该库执行许多与操作系统中的块堆栈相同的操作。这包括统一不同存储设备之间的接口、排队处理内存不足或 I/O 挂起等情况，以及逻辑卷管理。有关详细信息‎[‎，请参阅块设备用户指南‎](https://spdk.io/doc/bdev.html)‎。‎
>
> ‎最后，SPDK 提供了基于这些组件构建的 ‎‎NVMe-oF‎‎、‎‎iSCSI‎‎ 和 ‎‎vhost‎‎ 服务器，这些组件能够通过网络或其他进程提供磁盘。NVMe-oF 和 iSCSI 的标准 Linux 内核发起程序可与这些目标进行互操作，QEMU 与 vhost 也可互操作。这些服务器的 CPU 效率可以比其他实现高出一个数量级。这些目标可以用作如何实现高性能存储目标的示例，也可以用作生产部署的基础。‎

# Concepts

* [User Space Drivers](https://spdk.io/doc/userspace.html)
* [Direct Memory Access (DMA) From User Space](https://spdk.io/doc/memory.html)
* [Message Passing and Concurrency](https://spdk.io/doc/concurrency.html)
* [NAND Flash SSD Internals](https://spdk.io/doc/ssd_internals.html)
* [Submitting I/O to an NVMe Device](https://spdk.io/doc/nvme_spec.html)
* [Virtualized I/O with Vhost-user](https://spdk.io/doc/vhost_processing.html)
* [SPDK Structural Overview](https://spdk.io/doc/overview.html)
* [SPDK Porting Guide](https://spdk.io/doc/porting.html)

# User Space Drivers

# Controlling Hardware From User Space

Much of the documentation for SPDK talks about  *user space drivers* , so it's important to understand what that means at a technical level. First and foremost, a *driver* is software that directly controls a particular device attached to a computer. Second, operating systems segregate the system's virtual memory into two categories of addresses based on privilege level - [kernel space and user space](https://en.wikipedia.org/wiki/User_space). This separation is aided by features on the CPU itself that enforce memory separation called [protection rings](https://en.wikipedia.org/wiki/Protection_ring). Typically, drivers run in kernel space (i.e. ring 0 on x86). SPDK contains drivers that instead are designed to run in user space, but they still interface directly with the hardware device that they are controlling.

In order for SPDK to take control of a device, it must first instruct the operating system to relinquish control. This is often referred to as unbinding the kernel driver from the device and on Linux is done by [writing to a file in sysfs](https://lwn.net/Articles/143397/). SPDK then rebinds the driver to one of two special device drivers that come bundled with Linux - [uio](https://www.kernel.org/doc/html/latest/driver-api/uio-howto.html) or [vfio](https://www.kernel.org/doc/Documentation/vfio.txt). These two drivers are "dummy" drivers in the sense that they mostly indicate to the operating system that the device has a driver bound to it so it won't automatically try to re-bind the default driver. They don't actually initialize the hardware in any way, nor do they even understand what type of device it is. The primary difference between uio and vfio is that vfio is capable of programming the platform's [IOMMU](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit), which is a critical piece of hardware for ensuring memory safety in user space drivers. See [Direct Memory Access (DMA) From User Space](https://spdk.io/doc/memory.html) for full details.

Once the device is unbound from the operating system kernel, the operating system can't use it anymore. For example, if you unbind an NVMe device on Linux, the devices corresponding to it such as /dev/nvme0n1 will disappear. It further means that filesystems mounted on the device will also be removed and kernel filesystems can no longer interact with the device. In fact, the entire kernel block storage stack is no longer involved. Instead, SPDK provides re-imagined implementations of most of the layers in a typical operating system storage stack all as C libraries that can be directly embedded into your application. This includes a [block device abstraction layer](https://spdk.io/doc/bdev.html) primarily, but also [block allocators](https://spdk.io/doc/blob.html) and [filesystem-like components](https://spdk.io/doc/blobfs.html).

User space drivers utilize features in uio or vfio to map the [PCI BAR](https://en.wikipedia.org/wiki/PCI_configuration_space) for the device into the current process, which allows the driver to perform [MMIO](https://en.wikipedia.org/wiki/Memory-mapped_I/O) directly. The SPDK [NVMe Driver](https://spdk.io/doc/nvme.html), for instance, maps the BAR for the NVMe device and then follows along with the [NVMe Specification](http://nvmexpress.org/wp-content/uploads/NVM_Express_Revision_1.3.pdf) to initialize the device, create queue pairs, and ultimately send I/O.

> SPDK 的大部分文档都涉及‎*‎用户空间驱动程序‎*‎，因此在技术级别了解这意味着什么非常重要。首先，‎*‎驱动程序‎*‎是直接控制连接到计算机的特定设备的软件。其次，操作系统根据权限级别将系统的虚拟内存分为两类地址 - ‎[‎内核空间和用户空间‎](https://en.wikipedia.org/wiki/User_space)‎。CPU 本身的功能有助于实现这种分离，这些功能强制执行内存分离（称为‎[‎保护环‎](https://en.wikipedia.org/wiki/Protection_ring)‎）。通常，驱动程序在内核空间中运行（即 x86 上的 ring 0）。SPDK 包含的驱动程序被设计为在用户空间中运行，但它们仍直接与它们所控制的硬件设备交互。‎
>
> ‎为了使SPDK能够控制设备，它必须首先指示操作系统放弃控制。这通常称为从设备中解绑内核驱动程序，在 Linux 上是通过‎[‎写入 sysfs 中的文件来完成的‎](https://lwn.net/Articles/143397/)‎。然后，SPDK 将驱动程序重新绑定到与 Linux 捆绑在一起的两个特殊设备驱动程序之一 - ‎[‎uio‎](https://www.kernel.org/doc/html/latest/driver-api/uio-howto.html)‎ 或 ‎[‎vfio‎](https://www.kernel.org/doc/Documentation/vfio.txt)‎。这两个驱动程序是“虚拟”驱动程序，因为它们主要向操作系统指示设备绑定了驱动程序，因此它不会自动尝试重新绑定默认驱动程序。他们实际上并没有以任何方式初始化硬件，甚至不了解它是什么类型的设备。uio和vfio之间的主要区别在于vfio能够对平台的‎[‎IOMMU进行编程，IOMMU‎](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit)‎是确保用户空间驱动程序内存安全的关键硬件。有关完整的详细信息‎[‎，请参阅从用户空间进行直接内存访问 （DMA）。‎](https://spdk.io/doc/memory.html)
>
> ‎一旦设备从操作系统内核中解绑，操作系统就不能再使用它了。例如，如果在 Linux 上取消绑定 NVMe 设备，则与之对应的设备（如 /dev/nvme0n1）将消失。这进一步意味着挂载在设备上的文件系统也将被删除，内核文件系统不能再与设备交互。实际上，整个内核块存储堆栈不再涉及。相反，SPDK 提供了典型操作系统存储堆栈中大多数层的重新构想的实现，所有这些实现都作为 C 库，可以直接嵌入到应用程序中。这主要包括‎[‎块设备抽象层‎](https://spdk.io/doc/bdev.html)‎，但也包括‎[‎块分配器‎](https://spdk.io/doc/blob.html)‎和‎[‎类似文件系统的组件‎](https://spdk.io/doc/blobfs.html)‎。‎
>
> ‎用户空间驱动程序利用 uio 或 vfio 中的功能将设备的 ‎[‎PCI BAR‎](https://en.wikipedia.org/wiki/PCI_configuration_space)‎ 映射到当前进程，从而允许驱动程序直接执行 ‎[‎MMIO‎](https://en.wikipedia.org/wiki/Memory-mapped_I/O)‎。例如，SPDK ‎[‎NVMe 驱动程序‎](https://spdk.io/doc/nvme.html)‎映射 NVMe 设备的 BAR，然后按照 ‎[‎NVMe 规范‎](http://nvmexpress.org/wp-content/uploads/NVM_Express_Revision_1.3.pdf)‎初始化设备、创建队列对并最终发送 I/O。‎

# Interrupts

SPDK polls devices for completions instead of waiting for interrupts. There are a number of reasons for doing this: 1) practically speaking, routing an interrupt to a handler in a user space process just isn't feasible for most hardware designs, 2) interrupts introduce software jitter and have significant overhead due to forced context switches. Operations in SPDK are almost universally asynchronous and allow the user to provide a callback on completion. The callback is called in response to the user calling a function to poll for completions. Polling an NVMe device is fast because only host memory needs to be read (no MMIO) to check a queue pair for a bit flip and technologies such as Intel's [DDIO](https://www.intel.com/content/www/us/en/io/data-direct-i-o-technology.html) will ensure that the host memory being checked is present in the CPU cache after an update by the device.

> ‎SPDK 轮询设备以查找完成情况，而不是等待中断。这样做的原因有很多：1）实际上，在用户空间进程中将中断路由到处理程序对于大多数硬件设计是不可行的，2）中断会引入软件抖动，并且由于强制上下文切换而产生显着的开销。SPDK 中的操作几乎都是异步的，允许用户在完成时提供回调。调用回调是为了响应用户调用函数以轮询完成情况。轮询 NVMe 设备的速度很快，因为只需要读取主机内存（无需 MMIO）即可检查队列对是否有位翻转，而英特尔的 ‎[‎DDIO‎](https://www.intel.com/content/www/us/en/io/data-direct-i-o-technology.html)‎ 等技术将确保在设备更新后，正在检查的主机内存存在于 CPU 缓存中。‎

# Threading

NVMe devices expose multiple queues for submitting requests to the hardware. Separate queues can be accessed without coordination, so software can send requests to the device from multiple threads of execution in parallel without locks. Unfortunately, kernel drivers must be designed to handle I/O coming from lots of different places either in the operating system or in various processes on the system, and the thread topology of those processes changes over time. Most kernel drivers elect to map hardware queues to cores (as close to 1:1 as possible), and then when a request is submitted they look up the correct hardware queue for whatever core the current thread happens to be running on. Often, they'll need to either acquire a lock around the queue or temporarily disable interrupts to guard against preemption from threads running on the same core, which can be expensive. This is a large improvement from older hardware interfaces that only had a single queue or no queue at all, but still isn't always optimal.

A user space driver, on the other hand, is embedded into a single application. This application knows exactly how many threads (or processes) exist because the application created them. Therefore, the SPDK drivers choose to expose the hardware queues directly to the application with the requirement that a hardware queue is only ever accessed from one thread at a time. In practice, applications assign one hardware queue to each thread (as opposed to one hardware queue per core in kernel drivers). This guarantees that the thread can submit requests without having to perform any sort of coordination (i.e. locking) with the other threads in the system.

> ‎NVMe 设备公开多个队列以向硬件提交请求。无需协调即可访问单独的队列，因此软件可以从多个执行线程并行向设备发送请求，而无需锁定。不幸的是，内核驱动程序必须设计为处理来自操作系统或系统上各种进程中许多不同位置的 I/O，并且这些进程的线程拓扑会随着时间的推移而变化。大多数内核驱动程序选择将硬件队列映射到内核（尽可能接近 1：1），然后在提交请求时，它们会查找当前线程碰巧在其上运行的任何内核的正确硬件队列。通常，他们需要在队列周围获取一个锁，或者暂时禁用中断，以防止在同一内核上运行的线程抢占，这可能很昂贵。与只有单个队列或根本没有队列的旧硬件接口相比，这是一个很大的改进，但仍然并不总是最佳的。‎
>
> ‎另一方面，用户空间驱动程序嵌入到单个应用程序中。此应用程序确切地知道存在多少个线程（或进程），因为应用程序创建了它们。因此，SPDK 驱动程序选择直接向应用程序公开硬件队列，并要求一次只能从一个线程访问硬件队列。实际上，应用程序为每个线程分配一个硬件队列（而不是内核驱动程序中每个内核一个硬件队列）。这保证了线程可以提交请求，而不必与系统中的其他线程执行任何类型的协调（即锁定）。‎

# Direct Memory Access (DMA) From User Space

[]()The following is an attempt to explain why all data buffers passed to SPDK must be allocated using [spdk_dma_malloc()](https://spdk.io/doc/env_8h.html#a0874731c44ac31e4b14d91c6844a87d1 "Allocate a pinned memory buffer with the given size and alignment.") or its siblings, and why SPDK relies on DPDK's proven base functionality to implement memory management.

Computing platforms generally carve physical memory up into 4KiB segments called pages. They number the pages from 0 to N starting from the beginning of addressable memory. Operating systems then overlay 4KiB virtual memory pages on top of these physical pages using arbitrarily complex mappings. See [Virtual Memory](https://en.wikipedia.org/wiki/Virtual_memory) for an overview.

Physical memory is attached on channels, where each memory channel provides some fixed amount of bandwidth. To optimize total memory bandwidth, the physical addressing is often set up to automatically interleave between channels. For instance, page 0 may be located on channel 0, page 1 on channel 1, page 2 on channel 2, etc. This is so that writing to memory sequentially automatically utilizes all available channels. In practice, interleaving is done at a much more granular level than a full page.

Modern computing platforms support hardware acceleration for virtual to physical translation inside of their Memory Management Unit (MMU). The MMU often supports multiple different page sizes. On recent x86_64 systems, 4KiB, 2MiB, and 1GiB pages are supported. Typically, operating systems use 4KiB pages by default.

NVMe devices transfer data to and from system memory using Direct Memory Access (DMA). Specifically, they send messages across the PCI bus requesting data transfers. In the absence of an IOMMU, these messages contain *physical* memory addresses. These data transfers happen without involving the CPU, and the MMU is responsible for making access to memory coherent.

NVMe devices also may place additional requirements on the physical layout of memory for these transfers. The NVMe 1.0 specification requires all physical memory to be describable by what is called a  *PRP list* . To be described by a PRP list, memory must have the following properties:

* The memory is broken into physical 4KiB pages, which we'll call device pages.
* The first device page can be a partial page starting at any 4-byte aligned address. It may extend up to the end of the current physical page, but not beyond.
* If there is more than one device page, the first device page must end on a physical 4KiB page boundary.
* The last device page begins on a physical 4KiB page boundary, but is not required to end on a physical 4KiB page boundary.

The specification allows for device pages to be other sizes than 4KiB, but all known devices as of this writing use 4KiB.

The NVMe 1.1 specification added support for fully flexible scatter gather lists, but the feature is optional and most devices available today do not support it.

User space drivers run in the context of a regular process and so have access to virtual memory. In order to correctly program the device with physical addresses, some method for address translation must be implemented.

The simplest way to do this on Linux is to inspect `/proc/self/pagemap` from within a process. This file contains the virtual address to physical address mappings. As of Linux 4.0, accessing these mappings requires root privileges. However, operating systems make absolutely no guarantee that the mapping of virtual to physical pages is static. The operating system has no visibility into whether a PCI device is directly transferring data to a set of physical addresses, so great care must be taken to coordinate DMA requests with page movement. When an operating system flags a page such that the virtual to physical address mapping cannot be modified, this is called **pinning** the page.

There are several reasons why the virtual to physical mappings may change, too. By far the most common reason is due to page swapping to disk. However, the operating system also moves pages during a process called compaction, which collapses identical virtual pages onto the same physical page to save memory. Some operating systems are also capable of doing transparent memory compression. It is also increasingly possible to hot-add additional memory, which may trigger a physical address rebalance to optimize interleaving.

POSIX provides the `mlock` call that forces a virtual page of memory to always be backed by a physical page. In effect, this is disabling swapping. This does *not* guarantee, however, that the virtual to physical address mapping is static. The `mlock` call should not be confused with a **pin** call, and it turns out that POSIX does not define an API for pinning memory. Therefore, the mechanism to allocate pinned memory is operating system specific.

SPDK relies on DPDK to allocate pinned memory. On Linux, DPDK does this by allocating `hugepages` (by default, 2MiB). The Linux kernel treats hugepages differently than regular 4KiB pages. Specifically, the operating system will never change their physical location. This is not by intent, and so things could change in future versions, but it is true today and has been for a number of years (see the later section on the IOMMU for a future-proof solution).

With this explanation, hopefully it is now clear why all data buffers passed to SPDK must be allocated using [spdk_dma_malloc()](https://spdk.io/doc/env_8h.html#a0874731c44ac31e4b14d91c6844a87d1 "Allocate a pinned memory buffer with the given size and alignment.") or its siblings. The buffers must be allocated specifically so that they are pinned and so that physical addresses are known.

> ‎下面尝试解释为什么传递给 SPDK 的所有数据缓冲区都必须使用 ‎‎spdk_dma_malloc（）‎‎ 或其同级进行分配，以及为什么 SPDK 依赖于 DPDK 经过验证的基本功能来实现内存管理。‎
>
> ‎计算平台通常将物理内存划分为4KiB段，称为页面。它们从可寻址内存的开头开始对页进行从 0 到 N 的编号。然后，操作系统使用任意复杂的映射在这些物理页面的顶部覆盖4KiB虚拟内存页面。有关概述‎[‎，请参阅虚拟内存‎](https://en.wikipedia.org/wiki/Virtual_memory)‎。‎
>
> ‎物理内存附加在通道上，其中每个内存通道提供一些固定的带宽量。为了优化总内存带宽，通常将物理寻址设置为在通道之间自动交错。例如，第 0 页可能位于通道 0 上，第 1 页位于通道 1 上，第 2 页位于通道 2 上，依此类推。这样，按顺序写入内存即可自动利用所有可用通道。在实践中，交错是在比整页更精细的级别上完成的。‎
>
> ‎现代计算平台支持硬件加速，以便在其内存管理单元 （MMU） 内进行虚拟到物理转换。MMU 通常支持多种不同的页面大小。在最近的x86_64系统上，支持 4KiB、2MiB 和 1GiB 页。通常，操作系统默认使用 4KiB 页。‎
>
> ‎NVMe 设备使用直接内存访问 （DMA） 在系统内存之间来回传输数据。具体来说，它们通过PCI总线发送消息，请求数据传输。在没有 IOMMU 的情况下，这些消息包含‎*‎物理‎*‎内存地址。这些数据传输在不涉及CPU的情况下进行，MMU负责使对内存的访问保持一致。‎
>
> ‎NVMe 设备还可能对这些传输的内存的物理布局提出其他要求。NVMe 1.0 规范要求所有物理内存都可以通过所谓的 ‎*‎PRP 列表‎*‎进行描述。要通过 PRP 列表进行描述，内存必须具有以下属性：‎
>
> * ‎内存被分解为物理 4KiB 页，我们称之为设备页。‎
> * ‎第一个设备页面可以是从任何 4 字节对齐地址开始的部分页面。它可以一直延伸到当前物理页面的末尾，但不能延伸到更远的地方。‎
> * ‎如果有多个设备页，则第一个设备页必须在物理 4KiB 页边界上结束。‎
> * ‎最后一个设备页从物理 4KiB 页边界开始，但不需要在物理 4KiB 页边界上结束。‎
>
> ‎该规范允许设备页面的大小不超过 4KiB，但在撰写本文时，所有已知设备都使用 4KiB。‎
>
> ‎NVMe 1.1 规范增加了对完全灵活的分散收集列表的支持，但该功能是可选的，目前可用的大多数设备都不支持它。‎
>
> ‎用户空间驱动程序在常规进程的上下文中运行，因此可以访问虚拟内存。为了使用物理地址对设备进行正确编程，必须实现一些地址转换方法。‎
>
> ‎在 Linux 上执行此操作的最简单方法是从进程内进行检查‎`/proc/self/pagemap`。此文件包含虚拟地址到物理地址的映射。从 Linux 4.0 开始，访问这些映射需要 root 权限。但是，操作系统绝对不能保证虚拟页面到物理页面的映射是静态的。操作系统无法了解 PCI 设备是否将数据直接传输到一组物理地址，因此必须非常小心地协调 DMA 请求与页面移动。当操作系统标记页面，使得无法修改虚拟到物理地址的映射时，这称为‎‎固定‎‎页面。
>
> ‎虚拟到物理映射也可能发生变化的原因也有几个。到目前为止，最常见的原因是由于页面交换到磁盘。但是，操作系统还会在称为压缩的过程中移动页面，这会将相同的虚拟页面折叠到同一物理页面上以节省内存。某些操作系统还能够执行透明内存压缩。热添加额外内存的可能性也越来越大，这可能会触发物理地址重新平衡以优化交错。‎
>
> ‎POSIX 提供了 `mlock`强制虚拟内存页始终由物理页支持的调用。实际上，这是禁用交换。但是，这‎‎并不能保证‎‎虚拟到物理地址的映射是静态的。不应将 `mlock`调用与 ‎‎pin 调用‎‎混淆，事实证明 POSIX 没有定义用于固定内存的 API。因此，分配固定内存的机制是特定于操作系统的。‎
>
> ‎SPDK 依靠 DPDK 来分配固定内存。在 Linux 上，DPDK 通过分配 `hugepages`（默认为 2MiB）来实现此目的。Linux内核处理大页面的方式与常规的4KiB页面不同。具体来说，操作系统永远不会更改其物理位置。这不是故意的，所以事情可能会在未来的版本中发生变化，但今天确实如此，并且已经存在了很多年（请参阅IOMMU的后面部分以获取面向未来的解决方案）。‎
>
> ‎有了这个解释，希望现在很清楚为什么传递给SPDK的所有数据缓冲区都必须使用‎[‎spdk_dma_malloc（）‎](https://spdk.io/doc/env_8h.html#a0874731c44ac31e4b14d91c6844a87d1 "Allocate a pinned memory buffer with the given size and alignment.")‎或其同级分配。必须专门分配缓冲区，以便固定缓冲区并使物理地址已知。‎

# IOMMU Support

Many platforms contain an extra piece of hardware called an I/O Memory Management Unit (IOMMU). An IOMMU is much like a regular MMU, except it provides virtualized address spaces to peripheral devices (i.e. on the PCI bus). The MMU knows about virtual to physical mappings per process on the system, so the IOMMU associates a particular device with one of these mappings and then allows the user to assign arbitrary *bus addresses* to virtual addresses in their process. All DMA operations between the PCI device and system memory are then translated through the IOMMU by converting the bus address to a virtual address and then the virtual address to the physical address. This allows the operating system to freely modify the virtual to physical address mapping without breaking ongoing DMA operations. Linux provides a device driver, `vfio-pci`, that allows a user to configure the IOMMU with their current process.

This is a future-proof, hardware-accelerated solution for performing DMA operations into and out of a user space process and forms the long-term foundation for SPDK and DPDK's memory management strategy. We highly recommend that applications are deployed using vfio and the IOMMU enabled, which is fully supported today.

> ‎许多平台包含一个额外的硬件，称为 I/O 内存管理单元 （IOMMU）。IOMMU很像普通的MMU，除了它为外围设备（即PCI总线）提供虚拟化的地址空间。MMU 了解系统上每个进程的虚拟到物理映射，因此 IOMMU 将特定设备与这些映射之一相关联，然后允许用户在其进程中将任意‎‎总线地址‎‎分配给虚拟地址。然后，PCI 设备和系统内存之间的所有 DMA 操作都通过 IOMMU 进行转换，方法是将总线地址转换为虚拟地址，然后将虚拟地址转换为物理地址。这允许操作系统自由修改虚拟到物理地址的映射，而不会中断正在进行的 DMA 操作。Linux 提供了一个设备驱动程序 `vfio-pci` ，允许用户使用其当前进程配置 IOMMU。‎
>
> ‎这是一个面向未来的硬件加速解决方案，用于在用户空间进程中执行 DMA 操作，并为 SPDK 和 DPDK 的内存管理策略奠定了长期基础。我们强烈建议使用 vfio 部署应用程序并启用 IOMMU，目前已完全支持此功能。

# Message Passing and Concurrency

# Theory

One of the primary aims of SPDK is to scale linearly with the addition of hardware. This can mean many things in practice. For instance, moving from one SSD to two should double the number of I/O's per second. Or doubling the number of CPU cores should double the amount of computation possible. Or even doubling the number of NICs should double the network throughput. To achieve this, the software's threads of execution must be independent from one another as much as possible. In practice, that means avoiding software locks and even atomic instructions.

Traditionally, software achieves concurrency by placing some shared data onto the heap, protecting it with a lock, and then having all threads of execution acquire the lock only when accessing the data. This model has many great properties:

* It's easy to convert single-threaded programs to multi-threaded programs because you don't have to change the data model from the single-threaded version. You add a lock around the data.
* You can write your program as a synchronous, imperative list of statements that you read from top to bottom.
* The scheduler can interrupt threads, allowing for efficient time-sharing of CPU resources.

Unfortunately, as the number of threads scales up, contention on the lock around the shared data does too. More granular locking helps, but then also increases the complexity of the program. Even then, beyond a certain number of contended locks, threads will spend most of their time attempting to acquire the locks and the program will not benefit from more CPU cores.

SPDK takes a different approach altogether. Instead of placing shared data in a global location that all threads access after acquiring a lock, SPDK will often assign that data to a single thread. When other threads want to access the data, they pass a message to the owning thread to perform the operation on their behalf. This strategy, of course, is not at all new. For instance, it is one of the core design principles of [Erlang](http://erlang.org/download/armstrong_thesis_2003.pdf) and is the main concurrency mechanism in [Go](https://tour.golang.org/concurrency/2). A message in SPDK consists of a function pointer and a pointer to some context. Messages are passed between threads using a [lockless ring](http://dpdk.org/doc/guides/prog_guide/ring_lib.html). Message passing is often much faster than most software developer's intuition leads them to believe due to caching effects. If a single core is accessing the same data (on behalf of all of the other cores), then that data is far more likely to be in a cache closer to that core. It's often most efficient to have each core work on a small set of data sitting in its local cache and then hand off a small message to the next core when done.

In more extreme cases where even message passing may be too costly, each thread may make a local copy of the data. The thread will then only reference its local copy. To mutate the data, threads will send a message to each other thread telling them to perform the update on their local copy. This is great when the data isn't mutated very often, but is read very frequently, and is often employed in the I/O path. This of course trades memory size for computational efficiency, so it is used in only the most critical code paths.

> SPDK 的主要目标之一是通过添加硬件进行线性扩展。这在实践中可能意味着很多事情。例如，从一个 SSD 移动到两个 SSD 应该使每秒的 I/O 数量增加一倍。或者，将 CPU 内核数量增加一倍，应使可能的计算量增加一倍。甚至将 NIC 的数量增加一倍，网络吞吐量也会翻倍。为了实现这一点，软件的执行线程必须尽可能地彼此独立。在实践中，这意味着避免软件锁甚至原子指令。‎
>
> ‎传统上，软件通过将一些共享数据放在堆上，使用锁保护它，然后让所有执行线程仅在访问数据时获取锁来实现并发性。此模型具有许多出色的属性：‎
>
> * ‎将单线程程序转换为多线程程序很容易，因为您不必从单线程版本更改数据模型。在数据周围添加一个锁。‎
> * ‎您可以将程序编写为从上到下读取的语句的同步命令性列表。‎
> * ‎调度程序可以中断线程，从而实现 CPU 资源的高效分时。‎
>
> ‎不幸的是，随着线程数量的增加，围绕共享数据的锁上的争用也是如此。更精细的锁定会有所帮助，但也会增加程序的复杂性。即使这样，超过一定数量的争用锁，线程也会花费大部分时间尝试获取锁，并且程序不会从更多的CPU内核中受益。‎
>
> ‎SPDK采取了完全不同的方法。SPDK 通常不会将共享数据放在所有线程在获取锁后都访问的全局位置，而是将该数据分配给单个线程。当其他线程想要访问数据时，它们会将消息传递给拥有线程以代表它们执行操作。当然，这种策略并不是什么新鲜事。例如，它是‎[‎Erlang‎](http://erlang.org/download/armstrong_thesis_2003.pdf)‎的核心设计原则之一，也是‎[‎Go‎](https://tour.golang.org/concurrency/2)‎中的主要并发机制。SPDK 中的消息由函数指针和指向某个上下文的指针组成。消息使用‎[‎无锁环‎](http://dpdk.org/doc/guides/prog_guide/ring_lib.html)‎在线程之间传递。由于缓存效应，消息传递通常比大多数软件开发人员的直觉导致他们相信的要快得多。如果单个内核正在访问相同的数据（代表所有其他内核），则该数据更有可能位于更接近该内核的缓存中。通常最有效的方法是让每个核心都对一小组数据进行本地缓存，然后在完成后向下一个核心传递一条小消息。‎
>
> ‎在更极端的情况下，即使消息传递也可能太昂贵，每个线程都可能创建数据的本地副本。然后，线程将仅引用其本地副本。为了改变数据，线程将向彼此线程发送一条消息，告诉它们对其本地副本执行更新。当数据不经常发生突变，但读取频率非常高，并且经常在 I/O 路径中使用时，这非常有用。这当然会牺牲内存大小来提高计算效率，因此它仅用于最关键的代码路径。‎

# Message Passing Infrastructure

SPDK provides several layers of message passing infrastructure. The most fundamental libraries in SPDK, for instance, don't do any message passing on their own and instead enumerate rules about when functions may be called in their documentation (e.g. [NVMe Driver](https://spdk.io/doc/nvme.html)). Most libraries, however, depend on SPDK's [thread](http://www.spdk.io/doc/thread_8h.html) abstraction, located in `libspdk_thread.a`. The thread abstraction provides a basic message passing framework and defines a few key primitives.

First, `spdk_thread` is an abstraction for a lightweight, stackless thread of execution. A lower level framework can execute an `spdk_thread` for a single timeslice by calling `spdk_thread_poll()`. A lower level framework is allowed to move an `spdk_thread` between system threads at any time, as long as there is only a single system thread executing `spdk_thread_poll()`on that `spdk_thread` at any given time. New lightweight threads may be created at any time by calling `spdk_thread_create()` and destroyed by calling `spdk_thread_destroy()`. The lightweight thread is the foundational abstraction for threading in SPDK.

There are then a few additional abstractions layered on top of the `spdk_thread`. One is the `spdk_poller`, which is an abstraction for a function that should be repeatedly called on the given thread. Another is an `spdk_msg_fn`, which is a function pointer and a context pointer, that can be sent to a thread for execution via `spdk_thread_send_msg()`.

The library also defines two additional abstractions: `spdk_io_device` and `spdk_io_channel`. In the course of implementing SPDK we noticed the same pattern emerging in a number of different libraries. In order to implement a message passing strategy, the code would describe some object with global state and also some per-thread context associated with that object that was accessed in the I/O path to avoid locking on the global state. The pattern was clearest in the lowest layers where I/O was being submitted to block devices. These devices often expose multiple queues that can be assigned to threads and then accessed without a lock to submit I/O. To abstract that, we generalized the device to `spdk_io_device` and the thread-specific queue to `spdk_io_channel`. Over time, however, the pattern has appeared in a huge number of places that don't fit quite so nicely with the names we originally chose. In today's code `spdk_io_device` is any pointer, whose uniqueness is predicated only on its memory address, and `spdk_io_channel` is the per-thread context associated with a particular `spdk_io_device`.

The threading abstraction provides functions to send a message to any other thread, to send a message to all threads one by one, and to send a message to all threads for which there is an io_channel for a given io_device.

Most critically, the thread abstraction does not actually spawn any system level threads of its own. Instead, it relies on the existence of some lower level framework that spawns system threads and sets up event loops. Inside those event loops, the threading abstraction simply requires the lower level framework to repeatedly call `<a class="el text-primary" href="https://spdk.io/doc/thread_8h.html#ad9e3693e8e9e6c9063ea36414294ae91" title="Perform one iteration worth of processing on the thread.">spdk_thread_poll()</a>` on each `spdk_thread()` that exists. This makes SPDK very portable to a wide variety of asynchronous, event-based frameworks such as [Seastar](https://www.seastar.io/) or [libuv](https://libuv.org/).

> ‎SPDK 提供了多层消息传递基础结构。例如，SPDK中最基本的库不会自己执行任何消息传递，而是枚举有关何时可以在其文档中调用函数的规则（例如‎‎NVMe驱动程序‎‎）。但是，大多数库都依赖于 SPDK 的‎‎线程‎‎抽象，位于‎`libspdk_thread.a` .线程抽象提供了一个基本的消息传递框架，并定义了几个关键基元。
>
> ‎首先，`spdk_thread`是轻量级、无堆栈的执行线程的抽象。较低级别的框架可以通过调用 `spdk_thread_poll` 来执行单个时间片的 。允许较低级别的框架随时在系统线程之间移动，只要在任何给定时间只有一个系统线程在其上执行即可。可以通过调用 `spdk_thread_create`随时创建新的轻量级线程，也可以通过调用 `spdk_thread_destroy`来销毁。轻量级线程是 SPDK 中线程处理的基础抽象。‎
>
> ‎然后，在 `spdk_thread` .一个是 `spdk_poller` ，它是应该在给定线程上重复调用的函数的抽象。另一个是 `spdk_msg_fn`，它是函数指针和上下文指针，可以通过 `spdk_thread_send_msg` 发送到线程以执行。
>
> ‎该库还定义了两个附加抽象：`spdk_io_channel`和 `spdk_io_device`。在实现SPDK的过程中，我们注意到许多不同的库中出现了相同的模式。为了实现消息传递策略，代码将描述一些具有全局状态的对象，以及一些与在 I/O 路径中访问的对象关联的每线程上下文，以避免锁定全局状态。在将 I/O 提交到块设备的最低层中，该模式最为清晰。这些设备通常会公开多个队列，这些队列可以分配给线程，然后在没有锁定的情况下进行访问以提交 I/O。为了抽象化这一点，我们将设备推广到 `spdk_io_device` ，并将特定于线程的队列推广到 `spdk_io_channel` 。然而，随着时间的流逝，这种模式已经出现在大量与我们最初选择的名称不太契合的地方。在今天的代码中，任何指针都是指针，其唯一性仅基于其内存地址，并且 `spdk_io_channel`是与特定 `spdk_io_device`.‎
>
> ‎线程抽象提供了一些函数，用于将消息发送到任何其他线程，逐个向所有线程发送消息，以及将消息发送到给定io_device存在io_channel的所有线程。‎
>
> ‎最关键的是，线程抽象实际上并没有生成任何自己的系统级线程。相反，它依赖于一些较低级别的框架的存在，该框架生成系统线程并设置事件循环。在这些事件循环中，线程抽象只需要较低级别的框架重复调用每个存在的事件循环。这使得SPDK非常可移植到各种基于事件的异步框架，如‎‎Seastar‎‎或‎‎libuv‎‎。

# The event Framework

The SPDK project didn't want to officially pick an asynchronous, event-based framework for all of the example applications it shipped with, in the interest of supporting the widest variety of frameworks possible. But the applications do of course require something that implements an asynchronous event loop in order to run, so enter the `event` framework located in `lib/event`. This framework includes things like polling and scheduling the lightweight threads, installing signal handlers to cleanly shutdown, and basic command line option parsing. Only established applications should consider directly integrating the lower level libraries.

> SPDK 项目不想正式为其附带的所有示例应用程序选择一个异步的、基于事件的框架，以支持尽可能广泛的框架。但是应用程序当然需要实现异步事件循环的东西才能运行，因此请输入位于 `event`中的框架。该框架包括轮询和调度轻量级线程，安装信号处理程序以干净关闭以及基本命令行选项解析等内容。只有已建立的应用程序才应考虑直接集成较低级别的库。

# Limitations of the C Language

Message passing is efficient, but it results in asynchronous code. Unfortunately, asynchronous code is a challenge in C. It's often implemented by passing function pointers that are called when an operation completes. This chops up the code so that it isn't easy to follow, especially through logic branches. The best solution is to use a language with support for [futures and promises](https://en.wikipedia.org/wiki/Futures_and_promises), such as C++, Rust, Go, or almost any other higher level language. However, SPDK is a low level library and requires very wide compatibility and portability, so we've elected to stay with plain old C.

We do have a few recommendations to share, though. For *simple* callback chains, it's easiest if you write the functions from bottom to top. By that we mean if function `foo` performs some asynchronous operation and when that completes function `bar` is called, then function `bar` performs some operation that calls function `baz` on completion, a good way to write it is as such:

> ‎消息传递是有效的，但它会产生异步代码。不幸的是，异步代码在C中是一个挑战。它通常通过传递在操作完成时调用的函数指针来实现。这会切断代码，使其不容易遵循，特别是通过逻辑分支。最好的解决方案是使用一种支持[futures and promises](https://en.wikipedia.org/wiki/Futures_and_promises)的语言[‎](https://en.wikipedia.org/wiki/Futures_and_promises)‎，例如C++，Rust，Go或几乎任何其他更高级的语言。但是，SPDK是一个低级库，需要非常广泛的兼容性和可移植性，因此我们选择使用普通的旧C。‎
>
> 不过，我们确实有一些建议可以分享。对于‎*‎简单的‎*‎回调链，最简单的方法是从下到上编写函数。我们的意思是，如果函数执行一些异步操作，并且当调用该完成函数时，则函数执行一些在完成时调用函数的操作，那么编写它的好方法是这样的：

```c
void baz(void *ctx) {

    ...

}

void bar(void *ctx) {

    async_op(baz, ctx);

}

void foo(void *ctx) {

    async_op(bar, ctx);

}

```

Don't split these functions up - keep them as a nice unit that can be read from bottom to top.

For more complex callback chains, especially ones that have logical branches or loops, it's best to write out a state machine. It turns out that higher level languages that support futures and promises are just generating state machines at compile time, so even though we don't have the ability to generate them in C we can still write them out by hand. As an example, here's a callback chain that performs `foo` 5 times and then calls `bar` - effectively an asynchronous for loop.

> ‎不要拆分这些函数 - 将它们保留为一个可以从下到上读取的漂亮单元。‎
>
> ‎对于更复杂的回调链，特别是具有逻辑分支或循环的回调链，最好写出状态机。事实证明，支持未来和承诺的更高级语言只是在编译时生成状态机，所以即使我们没有能力在C中生成它们，我们仍然可以手动写出来。例如，下面是一个回调链，它执行 5 次，然后调用 - 实际上是一个异步 for 循环。

```c
enum states {

    FOO_START = 0,

    FOO_END,

    BAR_START,

    BAR_END

};

struct state_machine {

    enum states state;

    int count;

};

static void foo_complete(void *ctx)
{

    struct state_machine *sm = ctx;

    sm->state = FOO_END;

    run_state_machine(sm);

}

static void foo(struct state_machine *sm)
{

    do_async_op(foo_complete, sm);

}

static void bar_complete(void *ctx)
{

    struct state_machine *sm = ctx;

    sm->state = BAR_END;

    run_state_machine(sm);

}

static void bar(struct state_machine *sm)
{

    do_async_op(bar_complete, sm);

}

static void run_state_machine(struct state_machine *sm)
{

    enum states prev_state;

    do {

    prev_state = sm->state;

    switch (sm->state) {

    case FOO_START:

    foo(sm);

    break;

    case FOO_END:

    **/* This is the loop condition */**

    if (sm->count++ < 5) {

    sm->state = FOO_START;

    }else {

    sm->state = BAR_START;

    }

    break;

    case BAR_START:

    bar(sm);

    break;

    case BAR_END:

    break;

    }

    }while (prev_state != sm->state);

}

void do_async_for(void)
{

    **struct **state_machine *sm;

    sm = malloc(sizeof(*sm));

    sm->state = FOO_START;

    sm->count = 0;

    run_state_machine(sm);

}
```

This is complex, of course, but the `run_state_machine` function can be read from top to bottom to get a clear overview of what's happening in the code without having to chase through each of the callbacks.

> 当然，这很复杂，但可以从上到下读取该函数 `run_state_machine`，以便清楚地了解代码中发生的情况，而无需逐个回调。

# NAND Flash SSD Internals

Solid State Devices (SSD) are complex devices and their performance depends on how they're used. The following description is intended to help software developers understand what is occurring inside the SSD, so that they can come up with better software designs. It should not be thought of as a strictly accurate guide to how SSD hardware really works.

As of this writing, SSDs are generally implemented on top of [NAND Flash](https://en.wikipedia.org/wiki/Flash_memory) memory. At a very high level, this media has a few important properties:

* The media is grouped onto chips called NAND dies and each die can operate in parallel.
* Flipping a bit is a highly asymmetric process. Flipping it one way is easy, but flipping it back is quite hard.

NAND Flash media is grouped into large units often referred to as  **erase blocks** . The size of an erase block is highly implementation specific, but can be thought of as somewhere between 1MiB and 8MiB. For each erase block, each bit may be written to (i.e. have its bit flipped from 0 to 1) with bit-granularity once. In order to write to the erase block a second time, the entire block must be erased (i.e. all bits in the block are flipped back to 0). This is the asymmetry part from above. Erasing a block causes a measurable amount of wear and each block may only be erased a limited number of times.

SSDs expose an interface to the host system that makes it appear as if the drive is composed of a set of fixed size **logical blocks** which are usually 512B or 4KiB in size. These blocks are entirely logical constructs of the device firmware and they do not statically map to a location on the backing media. Instead, upon each write to a logical block, a new location on the NAND Flash is selected and written and the mapping of the logical block to its physical location is updated. The algorithm for choosing this location is a key part of overall SSD performance and is often called the **flash translation layer** or FTL. This algorithm must correctly distribute the blocks to account for wear (called  **wear-leveling** ) and spread them across NAND dies to improve total available performance. The simplest model is to group all of the physical media on each die together using an algorithm similar to RAID and then write to that set sequentially. Real SSDs are far more complicated, but this is an excellent simple model for software developers - imagine they are simply logging to a RAID volume and updating an in-memory hash-table.

One consequence of the flash translation layer is that logical blocks do not necessarily correspond to physical locations on the NAND at all times. In fact, there is a command that clears the translation for a block. In NVMe, this command is called deallocate, in SCSI it is called unmap, and in SATA it is called trim. When a user attempts to read a block that doesn't have a mapping to a physical location, drives will do one of two things:

1. Immediately complete the read request successfully, without performing any data transfer. This is acceptable because the data the drive would return is no more valid than the data already in the user's data buffer.
2. Return all 0's as the data.

Choice #1 is much more common and performing reads to a fully deallocated device will often show performance far beyond what the drive claims to be capable of precisely because it is not actually transferring any data. Write to all blocks prior to reading them when benchmarking!

As SSDs are written to, the internal log will eventually consume all of the available erase blocks. In order to continue writing, the SSD must free some of them. This process is often called  **garbage collection** . All SSDs reserve some number of erase blocks so that they can guarantee there are free erase blocks available for garbage collection. Garbage collection generally proceeds by:

1. Selecting a target erase block (a good mental model is that it picks the least recently used erase block)
2. Walking through each entry in the erase block and determining if it is still a valid logical block.
3. Moving valid logical blocks by reading them and writing them to a different erase block (i.e. the current head of the log)
4. Erasing the entire erase block and marking it available for use.

Garbage collection is clearly far more efficient when step #3 can be skipped because the erase block is already empty. There are two ways to make it much more likely that step #3 can be skipped. The first is that SSDs reserve additional erase blocks beyond their reported capacity (called  **over-provisioning** ), so that statistically its much more likely that an erase block will not contain valid data. The second is software can write to the blocks on the device in sequential order in a circular pattern, throwing away old data when it is no longer needed. In this case, the software guarantees that the least recently used erase blocks will not contain any valid data that must be moved.

The amount of over-provisioning a device has can dramatically impact the performance on random read and write workloads, if the workload is filling up the entire device. However, the same effect can typically be obtained by simply reserving a given amount of space on the device in software. This understanding is critical to producing consistent benchmarks. In particular, if background garbage collection cannot keep up and the drive must switch to on-demand garbage collection, the latency of writes will increase dramatically. Therefore the internal state of the device must be forced into some known state prior to running benchmarks for consistency. This is usually accomplished by writing to the device sequentially two times, from start to finish. For a highly detailed description of exactly how to force an SSD into a known state for benchmarking see this [SNIA Article](http://www.snia.org/sites/default/files/SSS_PTS_Enterprise_v1.1.pdf).

> ‎固态设备 （SSD） 是复杂的设备，其性能取决于它们的使用方式。以下描述旨在帮助软件开发人员了解SSD内部发生的情况，以便他们能够提出更好的软件设计。它不应该被认为是SSD硬件如何真正工作的严格准确的指南。‎
>
> ‎在撰写本文时，SSD通常在‎[‎NAND闪存‎](https://en.wikipedia.org/wiki/Flash_memory)‎之上实现。在非常高的层次上，这种介质具有一些重要的特性：‎
>
> * ‎介质被分组到称为NAND芯片的芯片上，每个芯片可以并行工作。‎
> * ‎翻转一点是一个高度不对称的过程。以一种方式翻转它很容易，但将其翻转回来是相当困难的。‎
>
> ‎NAND 闪存介质被分组为大型单元，通常称为‎**‎擦除块‎**‎。擦除块的大小高度特定于实现，但可以认为介于 1MiB 和 8MiB 之间。对于每个擦除块，每个位都可以写入（即将其位从0翻转为1）一次位粒度。为了再次写入擦除块，必须擦除整个块（即块中的所有位都翻转回0）。这是上面的不对称部分。擦除块会导致可测量的磨损量，并且每个块只能被擦除有限的次数。‎
>
> ‎SSD 向主机系统公开一个接口，使其看起来好像驱动器由一组固定大小的逻辑块组成，这些‎**‎逻辑块‎**‎的大小通常为 512B 或 4KiB。这些块完全是设备固件的逻辑构造，它们不会静态映射到支持介质上的位置。相反，每次写入逻辑块时，都会选择并写入 NAND 闪存上的新位置，并更新逻辑块与其物理位置的映射。选择此位置的算法是整体 SSD 性能的关键部分，通常称为‎**‎闪存转换层‎**‎或 FTL。该算法必须正确分配模块以考虑磨损（称为‎**‎磨损均衡），‎**‎并将它们分布在NAND芯片上，以提高总可用性能。最简单的模型是使用类似于RAID的算法将每个芯片上的所有物理介质组合在一起，然后按顺序写入该集。真正的SSD要复杂得多，但对于软件开发人员来说，这是一个极好的简单模型 - 想象一下，他们只是登录到RAID卷并更新内存中的哈希表。‎
>
> ‎闪存转换层的一个结果是，逻辑块不一定始终对应于NAND上的物理位置。实际上，有一个命令可以清除块的转换。在 NVMe 中，此命令称为解除分配，在 SCSI 中称为取消映射，在 SATA 中称为 trim。当用户尝试读取没有映射到物理位置的块时，驱动器将执行以下两项操作之一：‎
>
> 1. ‎立即成功完成读取请求，无需执行任何数据传输。这是可以接受的，因为驱动器将返回的数据并不比用户数据缓冲区中已有的数据更有效。‎
> 2. ‎返回所有 0 作为数据。‎
>
> ‎选择#1更为常见，对完全解除分配的设备执行读取通常会显示远远超出驱动器声称的能力的性能，因为它实际上并没有传输任何数据。在基准测试时读取所有块之前，请先写入它们！‎
>
> ‎写入 SSD 时，内部日志最终将占用所有可用的擦除块。为了继续写入，SSD必须释放其中一些。此过程通常称为‎**‎垃圾回收‎**‎。所有SSD都保留一定数量的擦除块，以便它们可以保证有可用的可用擦除块用于垃圾回收。垃圾回收通常通过以下方式进行：‎
>
> 1. ‎选择目标擦除块（一个好的心智模型是它选择最近最少使用的擦除块）‎
> 2. ‎遍历擦除块中的每个条目，并确定它是否仍是有效的逻辑块。‎
> 3. ‎通过读取有效的逻辑块并将其写入不同的擦除块（即日志的当前头）来移动它们‎
> 4. ‎擦除整个擦除块并将其标记为可用。‎
>
> ‎当可以跳过步骤#3时，垃圾回收显然要有效得多，因为擦除块已经是空的。有两种方法可以使跳过步骤#3的可能性更大。首先，SSD 会保留超出其报告容量的额外擦除块（称为‎**‎过度预配‎**‎），因此从统计上讲，擦除块更有可能不包含有效数据。第二种是软件可以按循环模式按顺序写入设备上的块，在不再需要时丢弃旧数据。在这种情况下，软件保证最近最少使用的擦除块将不包含任何必须移动的有效数据。‎
>
> ‎如果设备过度预配的数量会极大地影响随机读取和写入工作负载的性能，如果工作负载填满了整个设备。但是，通常可以通过简单地在软件中保留设备上的给定空间量来获得相同的效果。这种理解对于产生一致的基准至关重要。特别是，如果后台垃圾回收无法跟上，并且驱动器必须切换到按需垃圾回收，则写入的延迟将大大增加。因此，在运行基准测试以保持一致性之前，必须强制设备的内部状态进入某个已知状态。这通常是通过从头到尾按顺序写入设备两次来完成的。有关如何强制SSD进入已知状态以进行基准测试的非常详细的描述，请参阅此‎‎SNIA文章‎‎。‎

# Submitting I/O to an NVMe Device

# The NVMe Specification

The NVMe specification describes a hardware interface for interacting with storage devices. The specification includes network transport definitions for remote storage as well as a hardware register layout for local PCIe devices. What follows here is an overview of how an I/O is submitted to a local PCIe device through SPDK.

NVMe devices allow host software (in our case, the SPDK NVMe driver) to allocate queue pairs in host memory. The term "host" is used a lot, so to clarify that's the system that the NVMe SSD is plugged into. A queue pair consists of two queues - a submission queue and a completion queue. These queues are more accurately described as circular rings of fixed size entries. The submission queue is an array of 64 byte command structures, plus 2 integers (head and tail indices). The completion queue is similarly an array of 16 byte completion structures, plus 2 integers (head and tail indices). There are also two 32-bit registers involved that are called doorbells.

An I/O is submitted to an NVMe device by constructing a 64 byte command, placing it into the submission queue at the current location of the submission queue tail index, and then writing the new index of the submission queue tail to the submission queue tail doorbell register. It's actually valid to copy a whole set of commands into open slots in the ring and then write the doorbell just one time to submit the whole batch.

There is a very detailed description of the command submission and completion process in the NVMe specification, which is conveniently available from the main page over at [NVM Express](https://nvmexpress.org/).

Most importantly, the command itself describes the operation and also, if necessary, a location in host memory containing a descriptor for host memory associated with the command. This host memory is the data to be written on a write command, or the location to place the data on a read command. Data is transferred to or from this location using a DMA engine on the NVMe device.

The completion queue works similarly, but the device is instead the one writing entries into the ring. Each entry contains a "phase" bit that toggles between 0 and 1 on each loop through the entire ring. When a queue pair is set up to generate interrupts, the interrupt contains the index of the completion queue head. However, SPDK doesn't enable interrupts and instead polls on the phase bit to detect completions. Interrupts are very heavy operations, so polling this phase bit is often far more efficient.

> NVMe 规范描述了用于与存储设备交互的硬件接口。该规范包括远程存储的网络传输定义以及本地 PCIe 设备的硬件寄存器布局。下面概述了如何通过 SPDK 将 I/O 提交到本地 PCIe 设备。‎
>
> ‎NVMe 设备允许主机软件（在本例中为 SPDK NVMe 驱动程序）在主机内存中分配队列对。术语“主机”被大量使用，因此要澄清这是NVMe SSD插入的系统。队列对由两个队列组成 - 提交队列和完成队列。这些队列更准确地描述为固定大小条目的圆形环。提交队列是一个包含 64 字节命令结构的数组，外加 2 个整数（头索引和尾索引）。完成队列同样是一个包含 16 个字节完成结构的数组，外加 2 个整数（头索引和尾索引）。还涉及两个称为门铃的32位寄存器。‎
>
> ‎通过构造 64 字节命令，将其放入提交队列尾部索引的当前位置的提交队列中，然后将提交队列尾部的新索引写入提交队列尾部寄存器，将 I/O 提交到 NVMe 设备。实际上，将整组命令复制到环中的空位中，然后只写一次门铃以提交整个批次是有效的。‎
>
> ‎NVMe规范中有对命令提交和完成过程的非常详细的描述，可以从‎[‎NVM Express‎](https://nvmexpress.org/)‎的主页上方便地获得。‎
>
> ‎最重要的是，该命令本身描述了操作，并在必要时描述了主机内存中的一个位置，其中包含与该命令关联的主机内存的描述符。此主机内存是要写入写入命令的数据，或将数据放在读取命令上的位置。使用 NVMe 设备上的 DMA 引擎将数据传输到此位置或从此位置传输数据。‎
>
> ‎完成队列的工作方式类似，但设备是将条目写入环的那个。每个条目都包含一个“相位”位，该位在整个环中的每个循环上切换为 0 到 1。当队列对设置为生成中断时，中断将包含完成队列头的索引。但是，SPDK 不启用中断，而是轮询相位以检测完成情况。中断是非常繁重的操作，因此轮询此相位通常效率要高得多。‎

# The SPDK NVMe Driver I/O Path

Now that we know how the ring structures work, let's cover how the SPDK NVMe driver uses them. The user is going to construct a queue pair at some early time in the life cycle of the program, so that's not part of the "hot" path. Then, they'll call functions like [spdk_nvme_ns_cmd_read()](https://spdk.io/doc/nvme_8h.html#a084c6ecb53bd810fbb5051100b79bec5 "Submits a read I/O to the specified NVMe namespace.") to perform an I/O operation. The user supplies a data buffer, the target LBA, and the length, as well as other information like which NVMe namespace the command is targeted at and which NVMe queue pair to use. Finally, the user provides a callback function and context pointer that will be called when a completion for the resulting command is discovered during a later call to [spdk_nvme_qpair_process_completions()](https://spdk.io/doc/nvme_8h.html#aa331d140870e977722bfbb6826524782 "Process any outstanding completions for I/O submitted on a queue pair.").

The first stage in the driver is allocating a request object to track the operation. The operations are asynchronous, so it can't simply track the state of the request on the call stack. Allocating a new request object on the heap would be far too slow, so SPDK keeps a pre-allocated set of request objects inside of the NVMe queue pair object - `struct spdk_nvme_qpair`. The number of requests allocated to the queue pair is larger than the actual queue depth of the NVMe submission queue because SPDK supports a couple of key convenience features. The first is software queueing - SPDK will allow the user to submit more requests than the hardware queue can actually hold and SPDK will automatically queue in software. The second is splitting. SPDK will split a request for many reasons, some of which are outlined next. The number of request objects is configurable at queue pair creation time and if not specified, SPDK will pick a sensible number based on the hardware queue depth.

The second stage is building the 64 byte NVMe command itself. The command is built into memory embedded into the request object - not directly into an NVMe submission queue slot. Once the command has been constructed, SPDK attempts to obtain an open slot in the NVMe submission queue. For each element in the submission queue an object called a tracker is allocated. The trackers are allocated in an array, so they can be quickly looked up by an index. The tracker itself contains a pointer to the request currently occupying that slot. When a particular tracker is obtained, the command's CID value is updated with the index of the tracker. The NVMe specification provides that CID value in the completion, so the request can be recovered by looking up the tracker via the CID value and then following the pointer.

Once a tracker (slot) is obtained, the data buffer associated with it is processed to build a PRP list. That's essentially an NVMe scatter gather list, although it is a bit more restricted. The user provides SPDK with the virtual address of the buffer, so SPDK has to go do a page table look up to find the physical address (pa) or I/O virtual addresses (iova) backing that virtual memory. A virtually contiguous memory region may not be physically contiguous, so this may result in a PRP list with multiple elements. Sometimes this may result in a set of physical addresses that can't actually be expressed as a single PRP list, so SPDK will automatically split the user operation into two separate requests transparently. For more information on how memory is managed, see [Direct Memory Access (DMA) From User Space](https://spdk.io/doc/memory.html).

The reason the PRP list is not built until a tracker is obtained is because the PRP list description must be allocated in DMA-able memory and can be quite large. Since SPDK typically allocates a large number of requests, we didn't want to allocate enough space to pre-build the worst case scenario PRP list, especially given that the common case does not require a separate PRP list at all.

Each NVMe command has two PRP list elements embedded into it, so a separate PRP list isn't required if the request is 4KiB (or if it is 8KiB and aligned perfectly). Profiling shows that this section of the code is not a major contributor to the overall CPU use.

With a tracker filled out, SPDK copies the 64 byte command into the actual NVMe submission queue slot and then rings the submission queue tail doorbell to tell the device to go process it. SPDK then returns back to the user, without waiting for a completion.

The user can periodically call `spdk_nvme_qpair_process_completions()` to tell SPDK to examine the completion queue. Specifically, it reads the phase bit of the next expected completion slot and when it flips, looks at the CID value to find the tracker, which points at the request object. The request object contains a function pointer that the user provided initially, which is then called to complete the command.

The `spdk_nvme_qpair_process_completions()` function will keep advancing to the next completion slot until it runs out of completions, at which point it will write the completion queue head doorbell to let the device know that it can use the completion queue slots for new completions and return.

> 现在我们知道了环形结构的工作原理，让我们来了解一下 SPDK NVMe 驱动程序如何使用它们。用户将在程序生命周期的早期构建队列对，因此这不是“热”路径的一部分。然后，他们将调用‎[‎spdk_nvme_ns_cmd_read（）‎](https://spdk.io/doc/nvme_8h.html#a084c6ecb53bd810fbb5051100b79bec5 "Submits a read I/O to the specified NVMe namespace.")‎等函数来执行 I/O 操作。用户提供数据缓冲区、目标 LBA 和长度以及其他信息，例如命令针对哪个 NVMe 命名空间以及要使用的 NVMe 队列对。最后，用户提供了一个回调函数和上下文指针，当在以后调用‎[‎spdk_nvme_qpair_process_completions（）‎](https://spdk.io/doc/nvme_8h.html#aa331d140870e977722bfbb6826524782 "Process any outstanding completions for I/O submitted on a queue pair.")‎期间发现结果命令的完成时，将调用该函数和上下文指针。‎
>
> ‎驱动程序中的第一阶段是分配一个请求对象来跟踪操作。这些操作是异步的，因此它不能简单地跟踪调用堆栈上请求的状态。在堆上分配新的请求对象会太慢，因此 SPDK 在 NVMe 队列对对象 - 中保留了一组预先分配的请求对象。分配给队列对的请求数大于 NVMe 提交队列的实际队列深度，因为 SPDK 支持几个关键的便利功能。首先是软件排队 - SPDK将允许用户提交比硬件队列实际可以容纳的更多的请求，SPDK将自动在软件中排队。第二个是分裂。SPDK 会出于多种原因拆分请求，其中一些原因将在后面概述。请求对象的数量在队列对创建时是可配置的，如果未指定，SPDK 将根据硬件队列深度选择合理的数量。‎`struct spdk_nvme_qpair`
>
> ‎第二阶段是构建 64 字节 NVMe 命令本身。该命令内置于嵌入到请求对象的内存中，而不是直接内置到 NVMe 提交队列插槽中。构造命令后，SPDK 将尝试在 NVMe 提交队列中获取一个空位。对于提交队列中的每个元素，都会分配一个称为跟踪器的对象。跟踪器在数组中分配，因此可以通过索引快速查找它们。跟踪器本身包含一个指向当前占用该插槽的请求的指针。获得特定跟踪器后，命令的 CID 值将使用跟踪器的索引进行更新。NVMe规范在完成中提供了CID值，因此可以通过CID值查找跟踪器，然后按照指针来恢复请求。‎
>
> ‎一旦获得跟踪器（插槽），就会处理与其关联的数据缓冲区以构建PRP列表。这本质上是一个NVMe分散收集列表，尽管它受到更多的限制。用户向 SPDK 提供缓冲区的虚拟地址，因此 SPDK 必须执行页表查找以查找支持该虚拟内存的物理地址 （pa） 或 I/O 虚拟地址 （iova）。几乎连续的内存区域在物理上可能不连续，因此这可能会导致 PRP 列表包含多个元素。有时，这可能会导致一组物理地址实际上无法表示为单个 PRP 列表，因此 SPDK 会自动将用户操作透明地拆分为两个单独的请求。有关如何管理内存的详细信息，请参阅‎[‎从用户空间直接访问内存 （DMA）。‎](https://spdk.io/doc/memory.html)
>
> ‎在获得跟踪器之前不会构建 PRP 列表的原因是，PRP 列表描述必须分配在 DMA 内存中，并且可能非常大。由于 SPDK 通常会分配大量请求，因此我们不希望分配足够的空间来预先构建最坏情况的 PRP 列表，特别是考虑到常见情况根本不需要单独的 PRP 列表。‎
>
> ‎每个 NVMe 命令都嵌入了两个 PRP 列表元素，因此，如果请求为 4KiB（或者请求为 8KiB 且完全对齐），则不需要单独的 PRP 列表。分析表明，此部分代码不是总体 CPU 使用率的主要贡献者。‎
>
> ‎填写跟踪器后，SPDK 将 64 字节命令复制到实际的 NVMe 提交队列插槽中，然后按响提交队列尾门铃以告知设备进行处理。然后，SPDK 返回给用户，而无需等待完成。‎
>
> ‎用户可以定期调用 `spdk_nvme_qpair_process_completions`以告知 SPDK 检查完成队列。具体来说，它读取下一个预期完成槽的相位位，当它翻转时，查看CID值以找到指向请求对象的跟踪器。请求对象包含用户最初提供的函数指针，然后调用该指针以完成命令。
>
> ‎该函数 `spdk_nvme_qpair_process_completions`将继续前进到下一个完成槽，直到它用完完成，此时它将写入完成队列头门铃，让设备知道它可以使用完成队列槽进行新的完成并返回。‎

# Virtualized I/O with Vhost-user

# Table of Contents

* [Introduction](https://spdk.io/doc/vhost_processing.html#vhost_processing_intro)
* [QEMU](https://spdk.io/doc/vhost_processing.html#vhost_processing_qemu)
* [Device initialization](https://spdk.io/doc/vhost_processing.html#vhost_processing_init)
* [I/O path](https://spdk.io/doc/vhost_processing.html#vhost_processing_io_path)
* [SPDK optimizations](https://spdk.io/doc/vhost_processing.html#vhost_spdk_optimizations)

# Introduction

This document is intended to provide an overview of how Vhost works behind the scenes. Code snippets used in this document might have been simplified for the sake of readability and should not be used as an API or implementation reference.

Reading from the [Virtio specification](http://docs.oasis-open.org/virtio/virtio/v1.0/virtio-v1.0.html):

> The purpose of virtio and [virtio] specification is that virtual environments and guests should have a straightforward, efficient, standard and extensible mechanism for virtual devices, rather than boutique per-environment or per-OS mechanisms.

Virtio devices use virtqueues to transport data efficiently. Virtqueue is a set of three different single-producer, single-consumer ring structures designed to store generic scatter-gatter I/O. Virtio is most commonly used in QEMU VMs, where the QEMU itself exposes a virtual PCI device and the guest OS communicates with it using a specific Virtio PCI driver. With only Virtio involved, it's always the QEMU process that handles all I/O traffic.

Vhost is a protocol for devices accessible via inter-process communication. It uses the same virtqueue layout as Virtio to allow Vhost devices to be mapped directly to Virtio devices. This allows a Vhost device, exposed by an SPDK application, to be accessed directly by a guest OS inside a QEMU process with an existing Virtio (PCI) driver. Only the configuration, I/O submission notification, and I/O completion interruption are piped through QEMU. See also [SPDK optimizations](https://spdk.io/doc/vhost_processing.html#vhost_spdk_optimizations)

The initial vhost implementation is a part of the Linux kernel and uses ioctl interface to communicate with userspace applications. What makes it possible for SPDK to expose a vhost device is Vhost-user protocol.

The [Vhost-user specification](https://qemu-project.gitlab.io/qemu/interop/vhost-user.html) describes the protocol as follows:

> [Vhost-user protocol] is aiming to complement the ioctl interface used to control the vhost implementation in the Linux kernel. It implements the control plane needed to establish virtqueue sharing with a user space process on the same host. It uses communication over a Unix domain socket to share file descriptors in the ancillary data of the message.
>
> The protocol defines 2 sides of the communication, front-end and back-end. The front-end is the application that shares its virtqueues, in our case QEMU. The back-end is the consumer of the virtqueues.
>
> In the current implementation QEMU is the front-end, and the back-end is the external process consuming the virtio queues, for example a software Ethernet switch running in user space, such as Snabbswitch, or a block device back-end processing read and write to a virtual disk.
>
> The front-end and back-end can be either a client (i.e. connecting) or server (listening) in the socket communication.

SPDK vhost is a Vhost-user back-end server. It exposes Unix domain sockets and allows external applications to connect.

> ‎本文档旨在概述 Vhost 如何在幕后工作。为了便于阅读，本文档中使用的代码段可能已进行了简化，不应用作 API 或实现参考。‎
>
> ‎从‎[‎Virtio规范‎](http://docs.oasis-open.org/virtio/virtio/v1.0/virtio-v1.0.html)‎中解读：‎
>
>> ‎virtio 和 [virtio] 规范的目的是虚拟环境和gust应该为虚拟设备提供一个简单、高效、标准和可扩展的机制，而不是针对每个环境或每个操作系统的精品机制。‎
>>
>
> ‎Virtio设备使用virtqueues来有效地传输数据。Virtqueue 是一组三种不同的单生产者、单使用者环结构，旨在存储通用的分散式 I/O。Virtio 最常用于 QEMU VM，其中 QEMU 本身公开虚拟 PCI 设备，来宾操作系统使用特定的 Virtio PCI 驱动程序与其通信。由于只涉及 Virtio，因此始终是 QEMU 进程来处理所有 I/O 流量。‎
>
> ‎Vhost是可通过进程间通信访问的设备的协议。它使用与Virtio相同的virtqueue布局，以允许Vhost设备直接映射到Virtio设备。这允许由 SPDK 应用程序公开的 Vhost 设备由具有现有 Virtio （PCI） 驱动程序的 QEMU 进程内的来宾操作系统直接访问。只有配置、I/O 提交通知和 I/O 完成中断通过 QEMU 进行管道传输。另请参阅 ‎[‎SPDK 优化‎](https://spdk.io/doc/vhost_processing.html#vhost_spdk_optimizations)
>
> ‎初始 vhost 实现是 Linux 内核的一部分，使用 ioctl 接口与用户空间应用程序进行通信。使 SPDK 能够公开 vhost 设备的是 Vhost 用户协议。‎
>
> [‎Vhost 用户规范‎](https://qemu-project.gitlab.io/qemu/interop/vhost-user.html)‎对协议的描述如下：‎
>
>> ‎[Vhost-user协议]旨在补充用于控制Linux内核中vhost实现的ioctl接口。它实现了与同一主机上的用户空间进程建立 virtqueue 共享所需的控制平面。它使用通过 Unix 域套接字进行通信来共享消息辅助数据中的文件描述符。‎
>>
>> ‎该协议定义了通信的两端，前端和后端。前端是共享其 virtqueues 的应用程序，在我们的例子中是 QEMU。后端是 virtqueues 的使用者。‎
>>
>> ‎在当前的实现中，QEMU是前端，后端是消耗virtio队列的外部进程，例如在用户空间中运行的软件以太网交换机，例如Snabbswitch，或块设备后端处理对虚拟磁盘的读取和写入。‎
>>
>> ‎前端和后端可以是客户端（即连接）或服务器（侦听）中的套接字通信。‎
>>
>
> ‎SPDK vhost 是 Vhost 用户后端服务器。它公开 Unix 域套接字并允许外部应用程序进行连接。‎

# QEMU

One of major Vhost-user use cases is networking (DPDK) or storage (SPDK) offload in QEMU. The following diagram presents how QEMU-based VM communicates with SPDK Vhost-SCSI device.

> ‎Vhost用户的主要用例之一是QEMU中的网络（DPDK）或存储（SPDK）卸载。下图显示了基于 QEMU 的虚拟机如何与 SPDK Vhost-SCSI 设备进行通信。‎

![](https://spdk.io/doc/qemu_vhost_data_flow.svg "QEMU/SPDK vhost data flow")

# Device initialization

All initialization and management information is exchanged using Vhost-user messages. The connection always starts with the feature negotiation. Both the front-end and the back-end expose a list of their implemented features and upon negotiation they choose a common set of those. Most of these features are implementation-related, but also regard e.g. multiqueue support or live migration.

After the negotiation, the Vhost-user driver shares its memory, so that the vhost device (SPDK) can access it directly. The memory can be fragmented into multiple physically-discontiguous regions and Vhost-user specification puts a limit on their number - currently 8. The driver sends a single message for each region with the following data:

* file descriptor - for mmap
* user address - for memory translations in Vhost-user messages (e.g. translating vring addresses)
* guest address - for buffers addresses translations in vrings (for QEMU this is a physical address inside the guest)
* user offset - positive offset for the mmap
* size

The front-end will send new memory regions after each memory change - usually hotplug/hotremove. The previous mappings will be removed.

Drivers may also request a device config, consisting of e.g. disk geometry. Vhost-SCSI drivers, however, don't need to implement this functionality as they use common SCSI I/O to inquiry the underlying disk(s).

Afterwards, the driver requests the number of maximum supported queues and starts sending virtqueue data, which consists of:

* unique virtqueue id
* index of the last processed vring descriptor
* vring addresses (from user address space)
* call descriptor (for interrupting the driver after I/O completions)
* kick descriptor (to listen for I/O requests - unused by SPDK)

If multiqueue feature has been negotiated, the driver has to send a specific *ENABLE* message for each extra queue it wants to be polled. Other queues are polled as soon as they're initialized.

> ‎所有初始化和管理信息都使用 Vhost 用户消息进行交换。连接始终从功能协商开始。前端和后端都公开了其实现功能的列表，并在协商时选择一组通用的功能。这些功能中的大多数都与实现相关，但也涉及例如多队列支持或实时迁移。‎
>
> ‎协商后，Vhost 用户驱动程序将共享其内存，以便 vhost 设备 （SPDK） 可以直接访问它。内存可以分段到多个物理上不连续的区域，Vhost用户规范对其数量进行了限制 - 目前为8个。驱动程序为每个区域发送一条消息，其中包含以下数据：‎
>
> * ‎文件描述符 - 用于 mmap‎
> * ‎用户地址 - 用于 Vhost 用户消息中的内存转换（例如，转换 vring 地址）‎
> * ‎gost地址 - 对于缓冲区地址转换在 vrings 中（对于 QEMU，这是来宾内部的物理地址）‎
> * ‎用户偏移量 - mmap 的正偏移量‎
> * ‎大小‎
>
> ‎前端将在每次内存更改后发送新的内存区域 - 通常是热插拔/热删除。以前的映射将被删除。‎
>
> ‎驱动程序还可以请求设备配置，例如磁盘几何图形。但是，Vhost-SCSI 驱动程序不需要实现此功能，因为它们使用通用的 SCSI I/O 来查询底层磁盘。‎
>
> ‎之后，驱动程序请求最大支持队列数，并开始发送 virtqueue 数据，其中包括：‎
>
> * unique virtqueue id
> * ‎上次处理的 vring 描述符的索引‎
> * ‎vring 地址（来自用户地址空间）‎
> * ‎调用描述符（用于在 I/O 完成后中断驱动程序）‎
> * ‎踢描述符（侦听 I/O 请求 - SPDK 未使用）‎
>
> ‎如果已协商多队列功能，则驱动程序必须为要轮询的每个额外队列发送特定的 ‎*‎ENABLE‎*‎ 消息。其他队列在初始化后立即进行轮询。‎

# I/O path

The front-end sends I/O by allocating proper buffers in shared memory, filling the request data, and putting guest addresses of those buffers into virtqueues.

> ‎前端通过在共享内存中分配适当的缓冲区、填充请求数据以及将这些缓冲区的来宾地址放入 virtqueues 来发送 I/O。‎

A Virtio-Block request looks as follows.

```c
struct virtio_blk_req {

    uint32_t type;// READ, WRITE, FLUSH (read-only)

    uint64_t offset;// offset in the disk (read-only)

    struct iovec buffers[];// scatter-gatter list (read/write)

    uint8_t status;// I/O completion status (write-only)

};

```

And a Virtio-SCSI request as follows.

```c
struct virtio_scsi_req_cmd {

  struct virtio_scsi_cmd_req *req; // request data (read-only)

  struct iovec read_only_buffers[]; // scatter-gatter list for WRITE I/Os

  struct virtio_scsi_cmd_resp *resp; // response data (write-only)

  struct iovec write_only_buffers[]; // scatter-gatter list for READ I/Os

}
```

Virtqueue generally consists of an array of descriptors and each I/O needs to be converted into a chain of such descriptors. A single descriptor can be either readable or writable, so each I/O request consists of at least two (request + response).

> Virtqueue通常由一个描述符数组组成，每个I / O都需要转换为此类描述符链。单个描述符可以是可读的，也可以是可写的，因此每个 I/O 请求至少包含两个（请求 + 响应）。‎

```c
struct virtq_desc {

    / Address (guest-physical). /

    le64 addr;

    / Length. /

    le32 len;

/ This marks a buffer as continuing via the next field. /

#define VIRTQ_DESC_F_NEXT   1

/ This marks a buffer as device write-only (otherwise device read-only). /

#define VIRTQ_DESC_F_WRITE     2

    / The flags as indicated above. /

    le16 flags;

    / Next field if flags & NEXT /

    le16 next;

};
```

Legacy Virtio implementations used the name vring alongside virtqueue, and the name vring is still used in virtio data structures inside the code. Instead of `struct virtq_desc`, the `struct vring_desc` is much more likely to be found.

The device after polling this descriptor chain needs to translate and transform it back into the original request struct. It needs to know the request layout up-front, so each device backend (Vhost-Block/SCSI) has its own implementation for polling virtqueues. For each descriptor, the device performs a lookup in the Vhost-user memory region table and goes through a gpa_to_vva translation (guest physical address to vhost virtual address). SPDK enforces the request and response data to be contained within a single memory region. I/O buffers do not have such limitations and SPDK may automatically perform additional iovec splitting and gpa_to_vva translations if required. After forming the request structs, SPDK forwards such I/O to the underlying drive and polls for the completion. Once I/O completes, SPDK vhost fills the response buffer with proper data and interrupts the guest by doing an eventfd_write on the call descriptor for proper virtqueue. There are multiple interrupt coalescing features involved, but they are not be discussed in this document.

> ‎Legacy Virtio 实现将名称 vring 与 virtqueue 一起使用，并且名称 vring 仍然用于代码中的 virtio 数据结构中。而不是‎`struct virtq_desc` ，`struct vring_desc`更有可能被发现。
>
> ‎轮询此描述符链后，设备需要将其转换并转换回原始请求结构。它需要预先知道请求布局，因此每个设备后端（Vhost-Block/SCSI）都有自己的用于轮询 virtqueues 的实现。对于每个描述符，设备在 Vhost 用户内存区域表中执行查找，并执行gpa_to_vva转换（来宾物理地址到 vhost 虚拟地址）。SPDK 强制将请求和响应数据包含在单个内存区域中。I/O 缓冲区没有此类限制，如果需要，SPDK 可以自动执行额外的 iovec 拆分和gpa_to_vva转换。形成请求结构后，SPDK 将此类 I/O 转发到基础驱动器并轮询完成。I/O 完成后，SPDK vhost 会用适当的数据填充响应缓冲区，并通过在调用描述符上执行eventfd_write以正确排队来中断来宾。涉及多个中断合并功能，但本文档不讨论这些功能。‎

## SPDK optimizations

Due to its poll-mode nature, SPDK vhost removes the requirement for I/O submission notifications, drastically increasing the vhost server throughput and decreasing the guest overhead of submitting an I/O. A couple of different solutions exist to mitigate the I/O completion interrupt overhead (irqfd, vDPA), but those won't be discussed in this document. For the highest performance, a poll-mode [Virtio driver](https://spdk.io/doc/virtio.html) can be used, as it suppresses all I/O completion interrupts, making the I/O path to fully bypass the QEMU/KVM overhead.

> ‎由于其轮询模式特性，SPDK vhost 消除了对 I/O 提交通知的要求，从而大大提高了 vhost 服务器的吞吐量并减少了提交 I/O 的来宾开销。有几种不同的解决方案可以减轻 I/O 完成中断开销（irqfd、vDPA），但本文档不会讨论这些解决方案。为了获得最高性能，可以使用轮询模式‎[‎Virtio驱动程序‎](https://spdk.io/doc/virtio.html)‎，因为它可以抑制所有I / O完成中断，从而使I / O路径完全绕过QEMU / KVM开销。‎

# SPDK Structural Overview

# Overview

SPDK is composed of a set of C libraries residing in `lib` with public interface header files in `include/spdk`, plus a set of applications built out of those libraries in `app`. Users can use the C libraries in their software or deploy the full SPDK applications.

SPDK is designed around message passing instead of locking, and most of the SPDK libraries make several assumptions about the underlying threading model of the application they are embedded into. However, SPDK goes to great lengths to remain agnostic to the specific message passing, event, co-routine, or light-weight threading framework actually in use. To accomplish this, all SPDK libraries interact with an abstraction library in `lib/thread` (public interface at `<a class="el text-primary" href="https://spdk.io/doc/thread_8h.html" title="Thread.">include/spdk/thread.h</a>`). Any framework can initialize the threading abstraction and provide callbacks to implement the functionality that the SPDK libraries need. For more information on this abstraction, see [Message Passing and Concurrency](https://spdk.io/doc/concurrency.html).

SPDK is built on top of POSIX for most operations. To make porting to non-POSIX environments easier, all POSIX headers are isolated into `<a class="el text-primary" href="https://spdk.io/doc/stdinc_8h.html" title="Standard C headers.">include/spdk/stdinc.h</a>`. However, SPDK requires a number of operations that POSIX does not provide, such as enumerating the PCI devices on the system or allocating memory that is safe for DMA. These additional operations are all abstracted in a library called `env` whose public header is at `<a class="el text-primary" href="https://spdk.io/doc/env_8h.html" title="Encapsulated third-party dependencies.">include/spdk/env.h</a>`. By default, SPDK implements the `env` interface using a library based on DPDK. However, that implementation can be swapped out. See [SPDK Porting Guide](https://spdk.io/doc/porting.html) for additional information.

> ‎SPDK 由一组 C 库组成，这些库驻留在 中，其中有公共接口头文件 `nclude/spdk`，以及 一组由 这些库构建的应用程序。用户可以在其软件中使用 C 库或部署完整的 SPDK 应用程序。‎
>
> ‎SPDK 是围绕消息传递而不是锁定而设计的，大多数 SPDK 库对它们嵌入到的应用程序的基础线程模型做出了一些假设。但是，SPDK 不遗余力地保持与实际使用的特定消息传递、事件、共例程或轻量级线程框架无关。为了实现这一点，所有SPDK库都与（公共接口位于）中的抽象库进行交互。任何框架都可以初始化线程抽象并提供回调来实现 SPDK 库所需的功能。有关此抽象的更多信息，请参见‎‎消息传递和并发‎‎。‎
>
> ‎SPDK建立在POSIX之上，用于大多数操作。为了更轻松地移植到非 POSIX 环境，所有 POSIX 标头都隔离到 .但是，SPDK 需要许多 POSIX 不提供的操作，例如枚举系统上的 PCI 设备或分配对 DMA 安全的内存。这些附加操作都抽象在一个名为的库中，该库的公共标头位于 。默认情况下，SPDK 使用基于 DPDK 的库实现接口。但是，该实现可以换掉。有关其他信息‎‎，请参阅 SPDK 移植指南‎‎

# Applications

The `app` top-level directory contains full-fledged applications, built out of the SPDK components. For a full overview, see [An Overview of SPDK Applications](https://spdk.io/doc/app_overview.html).

SPDK applications can typically be started with a small number of configuration options. Full configuration of the applications is then performed using JSON-RPC. See [JSON-RPC](https://spdk.io/doc/jsonrpc.html) for additional information.

> ‎顶级目录包含由 SPDK 组件构建的完整应用程序。有关完整概述，请参阅 ‎‎SPDK 应用程序概述‎‎。
>
> ‎SPDK 应用程序通常可以使用少量配置选项启动。然后使用 JSON-RPC 执行应用程序的完整配置。有关其他信息‎[‎，请参阅 JSON-RPC‎](https://spdk.io/doc/jsonrpc.html)‎。‎

# Libraries

The `lib` directory contains the real heart of SPDK. Each component is a C library with its own directory under `lib`. Some of the key libraries are:

* [Block Device User Guide](https://spdk.io/doc/bdev.html)
* [NVMe Driver](https://spdk.io/doc/nvme.html)

# Documentation

The `doc` top-level directory contains all of SPDK's documentation. API Documentation is created using Doxygen directly from the code, but more general articles and longer explanations reside in this directory, as well as the Doxygen config file.

To build the documentation, just type `make` within the doc directory.

# Examples

The `examples` top-level directory contains a set of examples intended to be used for reference. These are different than the applications, which are doing a "real" task that could reasonably be deployed. The examples are instead either heavily contrived to demonstrate some facet of SPDK, or aren't considered complete enough to warrant tagging them as a full blown SPDK application.

This is a great place to learn about how SPDK works. In particular, check out `examples/nvme/hello_world`.

# Include

The `include` directory is where all of the header files are located. The public API is all placed in the `spdk` subdirectory of `include` and we highly recommend that applications set their include path to the top level `include` directory and include the headers by prefixing `spdk/` like this:

**#include "**[spdk/nvme.h](https://spdk.io/doc/nvme_8h.html)"

Most of the headers here correspond with a library in the `lib` directory. There are a few headers that stand alone, however. They are:

* `<a class="el text-primary" href="https://spdk.io/doc/assert_8h.html" title="Runtime and compile-time assert macros.">assert.h</a>`
* `<a class="el text-primary" href="https://spdk.io/doc/barrier_8h.html" title="Memory barriers.">barrier.h</a>`
* `<a class="el text-primary" href="https://spdk.io/doc/endian_8h.html" title="Endian conversion functions.">endian.h</a>`
* `<a class="el text-primary" href="https://spdk.io/doc/fd_8h.html" title="OS filesystem utility functions.">fd.h</a>`
* `<a class="el text-primary" href="https://spdk.io/doc/mmio_8h.html" title="Memory-mapped I/O utility functions.">mmio.h</a>`
* `queue.h` and `queue_extras.h`
* `<a class="el text-primary" href="https://spdk.io/doc/string_8h.html" title="String utility functions.">string.h</a>`

There is also an `spdk_internal` directory that contains header files widely included by libraries within SPDK, but that are not part of the public API and would not be installed on a user's system.

# Scripts

The `scripts` directory contains convenient scripts for a number of operations. The two most important are `check_format.sh`, which will use astyle and pep8 to check C, C++, and Python coding style against our defined conventions, and `setup.sh` which binds and unbinds devices from kernel drivers.

# Tests

The `test` directory contains all of the tests for SPDK's components and the subdirectories mirror the structure of the entire repository. The tests are a mixture of unit tests and functional tests.
