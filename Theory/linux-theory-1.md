分页
================================================================================

简介
--------------------------------------------------------------------------------

在 Linux 内核启动过程中的[第五部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-5.html)，我们学到了内核在启动的最早阶段都做了哪些工作。接下来，在我们明白内核如何运行第一个 init 进程之前，内核初始化其他部分，比如加载 `initrd` ，初始化 lockdep ，以及许多许多其他的工作。

是的，那将有很多不同的事，但是还有更多更多更多关于**内存**的工作。

在我看来，一般而言，内存管理是 Linux 内核和系统编程最复杂的部分之一。这就是为什么在我们学习内核初始化过程之前，需要了解分页。

分页是将线性地址转换为物理地址的机制。如果我们已经读过了这本书之前的部分，你可能记得我们在实模式下有分段机制，当时物理地址是由左移四位段寄存器加上偏移算出来的。我们也看了保护模式下的分段机制，其中我们使用描述符表得到描述符，进而得到基地址，然后加上偏移地址就获得了实际物理地址。由于我们在 64 位模式，我们将看分页机制。

正如 Intel 手册中说的：

> 分页机制提供一种机制，为了实现常见的按需分页，比如虚拟内存系统就是将一个程序执行环境中的段按照需求被映射到物理地址。

所以... 在这个帖子中我将尝试解释分页背后的理论。当然它将与64位版本的 Linux 内核关系密切，但是我们将不会深入太多细节（至少在这个帖子里面）。

开启分页
--------------------------------------------------------------------------------

有三种分页模式：

* 32 位分页模式；
* PAE 分页；
* IA-32e 分页。

我们这里将只解释最后一种模式。为了开启 `IA-32e 分页模式`，我们需要做如下事情：

* 设置 `CR0.PG` 位；
* 设置 `CR4.PAE` 位；
* 设置 `IA32_EFER.LME` 位。

我们已经在 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 中看见了这些位被设置了：

```assembly
movl	$(X86_CR0_PG | X86_CR0_PE), %eax
movl	%eax, %cr0
```

and

```assembly
movl	$MSR_EFER, %ecx
rdmsr
btsl	$_EFER_LME, %eax
wrmsr
```

分页数据结构
--------------------------------------------------------------------------------

分页将线性地址分为固定尺寸的页。页会被映射进入物理地址空间或外部存储设备。这个固定尺寸在 `x86_64` 内核中是 `4096` 字节。为了将线性地址转换位物理地址，需要使用到一些特殊的数据结构。每个结构都是 `4096` 字节并包含 `512` 项（这只为 `PAE` 和 `IA32_EFER.LME` 模式）。分页结构是层次级的， Linux 内核在 `x86_64` 框架中使用4层的分层机制。CPU使用一部分线性地址去确定另一个分页结构中的项，这个分页结构可能在最低层，物理内存区域（页框），在这个区域的物理地址（页偏移）。最高层的分页结构的地址存储在 `cr3` 寄存器中。我们已经从 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 这个文件中已经看到了。

```assembly
leal	pgtable(%ebx), %eax
movl	%eax, %cr3
```

我们构建页表结构并且将这个最高层结构的地址存放在 `cr3` 寄存器中。这里 `cr3` 用于存储最高层结构的地址，在 Linux 内核中被称为 `PML4` 或 `Page Global Directory` 。 `cr3` 是一个64位的寄存器，并且有着如下的结构：

```
63                  52 51                                                        32
 --------------------------------------------------------------------------------
|                     |                                                          |
|    Reserved MBZ     |            Address of the top level structure            |
|                     |                                                          |
 --------------------------------------------------------------------------------
31                                  12 11            5     4     3 2             0
 --------------------------------------------------------------------------------
|                                     |               |  P  |  P  |              |
|  Address of the top level structure |   Reserved    |  C  |  W  |    Reserved  |
|                                     |               |  D  |  T  |              |
 --------------------------------------------------------------------------------
```

这些字段有着如下的意义：

* 第 0 到第 2 位 - 忽略； 
* 第 12 位到第 51 位 - 存储最高层分页结构的地址；
* 第 3 位 到第 4 位 - PWT 或 Page-Level Writethrough 和 PCD 或 Page-level Cache Disable 显示。这些位控制页或者页表被硬件缓存处理的方式；
* 保留位 - 保留，但必须为 0 ；
* 第 52 到第 63 位 - 保留，但必须为 0 ；

线性地址转换过程如下所示：

* 一个给定的线性地址传递给 [MMU](http://en.wikipedia.org/wiki/Memory_management_unit) 而不是存储器总线；
* 64位线性地址分为很多部分。只有低 48 位是有意义的，它意味着 `2^48` 或 256TB 的线性地址空间在任意给定时间内都可以被访问；
* `cr3` 寄存器存储这个最高层分页数据结构的地址；
* 给定的线性地址中的第 39 位到第 47 位存储一个第 4 级分页结构的索引，第 30 位到第 38 位存储一个第3级分页结构的索引，第 29 位到第 21 位存储一个第 2 级分页结构的索引，第 12 位到第 20 位存储一个第 1 级分页结构的索引，第 0 位到第 11 位提供物理页的字节偏移；

按照图示，我们可以这样想象它：

![四层分页](http://oi58.tinypic.com/207mb0x.jpg)

每一个对线性地址的访问不是一个管态访问就是用户态访问。这个访问是被 `CPL (Current Privilege Level)` 所决定。如果 `CPL < 3` ，那么它是管态访问级，否则，它就是用户态访问级。比如，最高级页表项包含访问位和如下的结构：

```
63  62                  52 51                                                    32
 --------------------------------------------------------------------------------
| N |                     |                                                     |
|   |     Available       |     Address of the paging structure on lower level  |
| X |                     |                                                     |
 --------------------------------------------------------------------------------
31                                              12 11  9 8 7 6 5   4   3 2 1     0
 --------------------------------------------------------------------------------
|                                                |     | M |I| | P | P |U|W|    |
| Address of the paging structure on lower level | AVL | B |G|A| C | W | | |  P |
|                                                |     | Z |N| | D | T |S|R|    |
 --------------------------------------------------------------------------------
```

其中：

* 第 63 位 - N/X 位（不可执行位）显示被这个页表项映射的所有物理页执行代码的能力；
* 第 52 位到第 62 位 - 被CPU忽略，被系统软件使用；
* 第 12 位到第 51 位 - 存储低级分页结构的物理地址；
* 第 9 位到第 11 位 - 被 CPU 忽略；
* MBZ - 必须为 0 ；
* 忽略位；
* A - 访问位暗示物理页或者页结构被访问；
* PWT 和 PCD 用于缓存；
* U/S - 用户/管理位控制对被这个页表项映射的所有物理页用户访问；
* R/W - 读写位控制着被这个页表项映射的所有物理页的读写权限
* P - 存在位。当前位表示页表或物理页是否被加载进内存；

好的，我们知道了分页结构和它们的表项。现在我们来看一下 Linux 内核中的 4 级分页机制的一些细节。

Linux 内核中的分页结构
--------------------------------------------------------------------------------

就如我们已经看到的那样， `x86_64`Linux 内核使用4级页表。它们的名字是：

* 全局页目录
* 上层页目录
* 中间页目录
* 页表项

在你已经编译和安装 Linux 内核之后，你可以看到保存了内核函数的虚拟地址的文件 `System.map`。例如：

```
$ grep "start_kernel" System.map
ffffffff81efe497 T x86_64_start_kernel
ffffffff81efeaa2 T start_kernel
```

这里我们可以看见 `0xffffffff81efe497` 。我怀疑你是否真的有安装这么多内存。但是无论如何， `start_kernel` 和  `x86_64_start_kernel` 将会被执行。在 `x86_64` 中，地址空间的大小是 `2^64` ，但是它太大了，这就是为什么我们使用一个较小的地址空间，只是 48 位的宽度。所以一个情况出现，虽然物理地址空间限制到 48 位，但是寻址仍然使用 64 位指针。 这个问题是如何解决的？看下面的这个表。

```
0xffffffffffffffff  +-----------+
                    |           |
                    |           | Kernelspace
                    |           |
 0xffff800000000000 +-----------+
                    |           |
                    |           |
                    |   hole    |
                    |           |
                    |           |
0x00007fffffffffff  +-----------+
                    |           |
                    |           |  Userspace
                    |           |
0x0000000000000000  +-----------+
```

这个解决方案是 `sign extension` 。这里我们可以看到一个虚拟地址的低 48 位可以被用于寻址。第 48 位到第 63 位全是 0 或 1 。注意这个虚拟地址空间被分为两部分：

* 内核空间
* 用户空间

用户空间占用虚拟地址空间的低部分，从 `0x000000000000000` 到 `0x00007fffffffffff` ，而内核空间占据从 `0xffff8000000000` 到 `0xffffffffffffffff` 的高部分。注意，第 48 位到第 63 位是对于用户空间是 0 ，对于内核空间是 1 。内核空间和用户空间中的所有地址是标准地址，而在这些内存区域中间有非标准区域。这两块内存区域（内核空间和用户空间）合起来是 48 位宽度。我们可以在 [Documentation/x86/x86_64/mm.txt](https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/mm.txt) 找到 4 级页表下的虚拟内存映射:

```
0000000000000000 - 00007fffffffffff (=47 bits) user space, different per mm
hole caused by [48:63] sign extension
ffff800000000000 - ffff87ffffffffff (=43 bits) guard hole, reserved for hypervisor
ffff880000000000 - ffffc7ffffffffff (=64 TB) direct mapping of all phys. memory
ffffc80000000000 - ffffc8ffffffffff (=40 bits) hole
ffffc90000000000 - ffffe8ffffffffff (=45 bits) vmalloc/ioremap space
ffffe90000000000 - ffffe9ffffffffff (=40 bits) hole
ffffea0000000000 - ffffeaffffffffff (=40 bits) virtual memory map (1TB)
... unused hole ...
ffffec0000000000 - fffffc0000000000 (=44 bits) kasan shadow memory (16TB)
... unused hole ...
ffffff0000000000 - ffffff7fffffffff (=39 bits) %esp fixup stacks
... unused hole ...
ffffffff80000000 - ffffffffa0000000 (=512 MB)  kernel text mapping, from phys 0
ffffffffa0000000 - ffffffffff5fffff (=1525 MB) module mapping space
ffffffffff600000 - ffffffffffdfffff (=8 MB) vsyscalls
ffffffffffe00000 - ffffffffffffffff (=2 MB) unused hole
```

这里我们可以看到用户空间，内核空间和非标准空间的内存映射。用户空间的内存映射很简单。让我们来更近地查看内核空间。我们可以看到它始于为管理程序 (hypervisor) 保留的防御空洞 (guard hole) 。我们可以在 [arch/x86/include/asm/page_64_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_64_types.h) 这个文件中看到防御空洞的概念！

```C
#define __PAGE_OFFSET _AC(0xffff880000000000, UL)
```

以前防御空洞和 `__PAGE_OFFSET` 是从 `0xffff800000000000` 到 `0xffff80ffffffffff` ，用来防止对非标准区域的访问，但是后来为了管理程序扩展了 3 位。

紧接着是内核空间中最低的可用空间 - `ffff880000000000` 。这个虚拟地址空间是为了所有的物理内存的直接映射。在这块空间之后，还是防御空洞。它位于所有物理内存的直接映射地址和被 vmalloc 分配的地址之间。在第一个 1TB 的虚拟内存映射和无用的空洞之后，我们可以看到 `ksan` 影子内存 (shadow memory) 。它是通过 [commit](https://github.com/torvalds/linux/commit/ef7f0d6a6ca8c9e4b27d78895af86c2fbfaeedb2) 提交到内核中，并且保持内核空间无害。在紧接着的无用空洞之后，我们可以看到 `esp` 固定栈（我们会在本书其他部分讨论它）。内核代码段的开始从物理地址 - `0` 映射。我们可以在相同的文件中找到将这个地址定义为 `__PAGE_OFFSET` 。

```C
#define __START_KERNEL_map      _AC(0xffffffff80000000, UL)
```

通常内核的 `.text` 段开始于 `CONFIG_PHYSICAL_START` 偏移。我们已经在 [ELF64](https://github.com/MintCN/linux-insides-zh/blob/master/Theory/ELF.md) 相关帖子中看见。

```
readelf -s vmlinux | grep ffffffff81000000
     1: ffffffff81000000     0 SECTION LOCAL  DEFAULT    1 
 65099: ffffffff81000000     0 NOTYPE  GLOBAL DEFAULT    1 _text
 90766: ffffffff81000000     0 NOTYPE  GLOBAL DEFAULT    1 startup_64
```

这里我将 `CONFIG_PHYSICAL_START` 设置为 `0x1000000` 来检查 `vmlinux` 。所以我们有内核代码段的起始点 - `0xffffffff80000000` 和 偏移 - `0x1000000` ，计算出来的虚拟地址将会是 `0xffffffff80000000 + 1000000 = 0xffffffff81000000` 。

在内核代码段之后有一个为内核模块 `vsyscalls` 准备的虚拟内存区域和 2M 无用的空洞。

我们已经看见内核虚拟内存映射是如何布局的以及虚拟地址是如何转换位物理地址。让我们以下面的地址为例：

```
0xffffffff81000000
```
 
在二进制内它将是：

```
1111111111111111 111111111 111111110 000001000 000000000 000000000000
      63:48        47:39     38:30     29:21     20:12      11:0
```

这个虚拟地址将被分为如下描述的几部分：

* `48-63` - 不使用的位；
* `37-49` - 给定线性地址的这些位描述一个 4 级分页结构的索引；
* `30-38` - 这些位存储一个 3 级分页结构的索引；
* `21-29` - 这些位存储一个 2 级分页结构的索引；
* `12-20` - 这些位存储一个 1 级分页结构的索引；
* `0-11`  - 这些位提供物理页的偏移；


就这样了。现在你知道了一些关于分页理论，而且我们可以在内核源码上更近一步，查看那些最先的初始化步骤。

总结
--------------------------------------------------------------------------------

这简短的关于分页理论的部分至此已经结束了。当然，这个帖子不可能包含分页的所有细节，但是我们很快会看到在实践中 Linux 内核如何构建分页结构以及使用它们工作。

链接
--------------------------------------------------------------------------------

* [Paging on Wikipedia](http://en.wikipedia.org/wiki/Paging)
* [Intel 64 and IA-32 architectures software developer's manual volume 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [MMU](http://en.wikipedia.org/wiki/Memory_management_unit)
* [ELF64](https://github.com/0xAX/linux-insides/blob/master/Theory/ELF.md)
* [Documentation/x86/x86_64/mm.txt](https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/mm.txt)
* [Last part - Kernel booting process](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-5.html)
