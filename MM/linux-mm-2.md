内核内存管理. 第二部分.
================================================================================

固定映射地址和输入输出重映射
--------------------------------------------------------------------------------

固定映射地址是一组特殊的编译时确定的地址，它们与物理地址不一定具有减 `__START_KERNEL_map` 的线性映射关系。每一个固定映射的地址都会映射到一个内存页，内核会像指针一样使用它们，但是绝不会修改它们的地址。这是这种地址的主要特点。就像注释所说的那样，“在编译期就获得一个常量地址，只有在引导阶段才会被设定上物理地址。”你在本书的[前面部分](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-1.html)可以看到，我们已经设定了 `level2_fixmap_pgt` ：

```assembly
NEXT_PAGE(level2_fixmap_pgt)
	.fill	506,8,0
	.quad	level1_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE
	.fill	5,8,0

NEXT_PAGE(level1_fixmap_pgt)
	.fill	512,8,0
```

就像我们看到的， `level2_fixmap_pgt` 紧挨着 `level2_kernel_pgt` 保存了内核的 code+data+bss 段。每一个固定映射的地址都由一个整数下标表示，这些整数下标在 [arch/x86/include/asm/fixmap.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/fixmap.h) 的 `fixed_addresses` 枚举类型中定义。比如，它包含了`VSYSCALL_PAGE` 的入口 - 如果合法的 vsyscall 页模拟机制被开启，或是启用了本地 [apic](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) 的 `FIX_APIC_BASE` 选项等等。在虚拟内存中，固定映射区域被放置在模块区域中：

```
       +-----------+-----------------+---------------+------------------+
       |           |                 |               |                  |
	   |kernel text|      kernel     |               |    vsyscalls     |
	   | mapping   |       text      |    Modules    |    fix-mapped    |
       |from phys 0|       data      |               |    addresses     |
       |           |                 |               |                  |
       +-----------+-----------------+---------------+------------------+
__START_KERNEL_map   __START_KERNEL    MODULES_VADDR            0xffffffffffffffff
```

基虚拟地址和固定映射区域的尺寸使用以下两个宏表示：

```C
#define FIXADDR_SIZE	(__end_of_permanent_fixed_addresses << PAGE_SHIFT)
#define FIXADDR_START		(FIXADDR_TOP - FIXADDR_SIZE)
```

在这里 `__end_of_permanent_fixed_addresses` 是 `fixed_addresses` 枚举中的一个元素，如我上文所说：每一个固定映射地址都由一个定义在 `fixed_addresses` 中的整数下标表示。`PAGE_SHIFT` 决定了页的大小。比如，我们可以使用 `1 << PAGE_SHIFT` 来获取一页的大小。在我们的场景下需要获取固定映射区域的尺寸，而不仅仅是一页的大小，这就是我们使用 `__end_of_permanent_fixed_addresses` 来获取固定映射区域尺寸的原因。在我的系统中这个值可能略大于 `536` KB。在你的系统上这个值可能会不同，因为这个值取决于固定映射地址的数目，而这个数目又取决于内核的配置。

The second `FIXADDR_START` macro just substracts fix-mapped area size from the last address of the fix-mapped area to get its base virtual address. `FIXADDR_TOP` is a rounded up address from the base address of the [vsyscall](https://lwn.net/Articles/446528/) space:
第二个 `FIXADDR_START` 宏只是从固定映射区域的末地址减去了固定映射区域的尺寸，这样就可以获得它的基虚拟地址。 `FIXADDR_TOP` 是一个从 [vsyscall](https://lwn.net/Articles/446528/) 空间的基址取整产生的地址：

```C
#define FIXADDR_TOP     (round_up(VSYSCALL_ADDR + PAGE_SIZE, 1<<PMD_SHIFT) - PAGE_SIZE)
```

`fixed_addresses` 枚举量被 `fix_to_virt` 函数用做下标用于获取虚拟地址。这个函数的实现很简单：

```C
static __always_inline unsigned long fix_to_virt(const unsigned int idx)
{
        BUILD_BUG_ON(idx >= __end_of_fixed_addresses);
        return __fix_to_virt(idx);
}
```

首先它调用 `BUILD_BUG_ON` 宏检查了给定的 `fixed_addresses` 枚举量不大于等于 `__end_of_fixed_addresses`，然后返回了 `__fix_to_virt` 宏的运算结果：

```C
#define __fix_to_virt(x)        (FIXADDR_TOP - ((x) << PAGE_SHIFT))
```

在这里我们用 `PAGE_SHIFT` 左移了给定的固定映射地址下标，就像我上文所述它决定了页的地址，然后将 `FIXADDR_TOP` 减去这个值，`FIXADDR_TOP` 是固定映射区域的最高地址。以下是从虚拟地址获取对应固定映射地址的转换函数：

```C
static inline unsigned long virt_to_fix(const unsigned long vaddr)
{
        BUG_ON(vaddr >= FIXADDR_TOP || vaddr < FIXADDR_START);
        return __virt_to_fix(vaddr);
}
```

`virt_to_fix` 以虚拟地址为参数，检查了这个地址是否位于 `FIXADDR_START` 和 `FIXADDR_TOP` 之间，然后调用 `__virt_to_fix` ，这个宏实现如下：

```C
#define __virt_to_fix(x)        ((FIXADDR_TOP - ((x)&PAGE_MASK)) >> PAGE_SHIFT)
```

一个 PFN 是一块页大小物理内存的下标。一个物理地址的 PFN 可以简单地定义为 (page_phys_addr >> PAGE_SHIFT)；

`__virt_to_fix` 会清空给定地址的前 12 位，然后用固定映射区域的末地址(`FIXADDR_TOP`)减去它并右移 `PAGE_SHIFT` 即 12 位。让我们来解释它的工作原理。就像我已经写的那样，这个宏会使用 `x & PAGE_MASK` 来清空前 12 位。然后我们用 `FIXADDR_TOP` 减去它，就会得到 `FIXADDR_TOP` 的后 12 位。我们知道虚拟地址的前 12 位代表这个页的偏移量，当我们右移 `PAGE_SHIFT` 后就会得到 `Page frame number` ，即虚拟地址的所有位，包括最开始的 12 个偏移位。固定映射地址在[内核中多处使用](http://lxr.free-electrons.com/ident?i=fix_to_virt)。 `IDT` 描述符保存在这里，[英特尔可信赖执行技术](http://en.wikipedia.org/wiki/Trusted_Execution_Technology) UUID 储存在固定映射区域，以 `FIX_TBOOT_BASE` 下标开始。另外， [Xen](http://en.wikipedia.org/wiki/Xen) 引导映射等也储存在这个区域。我们已经在[内核初始化的第五部分](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-5.html)看到了一部分关于固定映射地址的知识。接下来让我们看看什么是 `ioremap`，看看它是怎样实现的，与固定映射地址又有什么关系呢？

输入输出重映射
--------------------------------------------------------------------------------

内核提供了许多不同的内存管理原语。现在我们将要接触 `I/O 内存`。每一个设备都通过读写它的寄存器来控制。比如，驱动可以通过向它的寄存器中写来打开或关闭设备，也可以通过读它的寄存器来获取设备状态。除了寄存器之外，许多设备都拥有一块可供驱动读写的缓冲区。如我们所知，现在有两种方法来访问设备的寄存器和数据缓冲区：

* 通过 I/O 端口；
* 将所有寄存器映射到内存地址空间；

第一种情况，设备的所有控制寄存器都具有一个输入输出端口号。该设备的驱动可以用 `in` 和 `out` 指令来从端口中读写。你可以通过访问 `/proc/ioports` 来获取设备当前的 I/O 端口号。

```
$ cat /proc/ioports
0000-0cf7 : PCI Bus 0000:00
  0000-001f : dma1
  0020-0021 : pic1
  0040-0043 : timer0
  0050-0053 : timer1
  0060-0060 : keyboard
  0064-0064 : keyboard
  0070-0077 : rtc0
  0080-008f : dma page reg
  00a0-00a1 : pic2
  00c0-00df : dma2
  00f0-00ff : fpu
    00f0-00f0 : PNP0C04:00
  03c0-03df : vesafb
  03f8-03ff : serial
  04d0-04d1 : pnp 00:06
  0800-087f : pnp 00:01
  0a00-0a0f : pnp 00:04
  0a20-0a2f : pnp 00:04
  0a30-0a3f : pnp 00:04
0cf8-0cff : PCI conf1
0d00-ffff : PCI Bus 0000:00
...
...
...
```


`/proc/ioports` 提供了驱动使用 I/O 端口的内存区域地址。所有的这些内存区域，比如 `0000-0cf7` ，都是使用 [include/linux/ioport.h](https://github.com/torvalds/linux/blob/master/include/linux/ioport.h) 头文件中的 `request_region` 来声明的。实际上 `request_region` 是一个宏，它的定义如下：

```C
#define request_region(start,n,name)   __request_region(&ioport_resource, (start), (n), (name), 0)
```

正如我们所看见的，它有三个参数：

* `start` - 区域的起点;
* `n`     - 区域的长度;
* `name`  - 区域需求者的名字。

`request_region` 分配 I/O 端口区域。通常在 `request_region` 之前会调用 `check_region` 来检查传入的地址区间是否可用，然后 `release_region` 会释放这个内存区域。`request_region` 返回指向 `resource` 结构体的指针。 `resource` 结构体是对系统资源的树状子集的抽象。我们已经在[内核初始化的第五部分](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-5.html)见到过它了，它的定义是这样的：

```C
struct resource {
        resource_size_t start;
        resource_size_t end;
        const char *name;
        unsigned long flags;
        struct resource *parent, *sibling, *child;
};
```

它包含起止地址、名字等等。每一个 `resource` 结构体包含一个指向 `parent`、`slibling` 和 `child` 资源的指针。它有父节点和子节点，这就意味着每一个资源的子集都有一个根节点。比如，对 I/O 端口来说有一个 `ioport_resource` 结构体：

```C
struct resource ioport_resource = {
         .name   = "PCI IO",
         .start  = 0,
         .end    = IO_SPACE_LIMIT,
        .flags  = IORESOURCE_IO,
};
EXPORT_SYMBOL(ioport_resource);
```

或者对 `iomem` 来说，有一个 `iomem_resource` 结构体：

```C
struct resource iomem_resource = {
        .name   = "PCI mem",
        .start  = 0,
        .end    = -1,
        .flags  = IORESOURCE_MEM,
};
```

就像我所写的，`request_region` 用于注册 I/O 端口区域，这个宏用于[内核中的许多地方](http://lxr.free-electrons.com/ident?i=request_region)。比如让我们来看看 [drivers/char/rtc.c](https://github.com/torvalds/linux/blob/master/char/rtc.c)。这个源文件提供了内核中的[实时时钟](http://en.wikipedia.org/wiki/Real-time_clock)接口。与其他内核模块一样， `rtc` 模块包含一个 `module_init` 定义：

```C
module_init(rtc_init);
```

在这里 `rtc_init` 是 `rtc` 模块的初始化函数。这个函数也定义在 `rtc.c` 文件中。在 `rtc_init` 函数中我们可以看到许多对 `rtc_request_region` 函数的调用，实际上这是 `request_region` 的包装：

```C
r = rtc_request_region(RTC_IO_EXTENT);
```

`rtc_request_region` 中调用了:

```C
r = request_region(RTC_PORT(0), size, "rtc");
```

在这里 `RTC_TO_EXTENT` 是一个内存区域的尺寸，在这里是 `0x8`， `"rtc"` 是区域的名字，`RTC_PORT` 是：

```C
#define RTC_PORT(x)     (0x70 + (x))
```

所以使用 `request_region(RTC_PORT(0), size, "rtc")` 我们注册了一个内存区域， 以 `0x70` 开始，大小为 `0x8`。 让我们看看 `/proc/ioports`:

```
~$ sudo cat /proc/ioports | grep rtc
0070-0077 : rtc0
```

看，我们可以获取了它的信息。这就是端口。第二种途径是使用 I/O 内存。就像我上面写的，这是将设备的控制寄存器和内存映射到内存地址空间中。I/O 内存是一组由设备通过总线提供给 CPU 的相邻的地址。所有的 I/O 映射地址都不能由内核直接访问。有一个 `ioremap` 函数用来将总线上的物理地址转化为内核的虚拟地址，或者说，`ioremap` 映射了 I/O 物理地址来让他们能够在内核中使用。这个函数有两个参数：

* 内存区域的开始；
* 内存区域的结束；

I/O 内存映射 API 提供了用来检查、请求与释放内存区域的函数，就像 I/O 端口 API 一样。这里有三个函数：

* `request_mem_region`
* `release_mem_region`
* `check_mem_region`

```
~$ sudo cat /proc/iomem
...
...
...
be826000-be82cfff : ACPI Non-volatile Storage
be82d000-bf744fff : System RAM
bf745000-bfff4fff : reserved
bfff5000-dc041fff : System RAM
dc042000-dc0d2fff : reserved
dc0d3000-dc138fff : System RAM
dc139000-dc27dfff : ACPI Non-volatile Storage
dc27e000-deffefff : reserved
defff000-deffffff : System RAM
df000000-dfffffff : RAM buffer
e0000000-feafffff : PCI Bus 0000:00
  e0000000-efffffff : PCI Bus 0000:01
    e0000000-efffffff : 0000:01:00.0
  f7c00000-f7cfffff : PCI Bus 0000:06
    f7c00000-f7c0ffff : 0000:06:00.0
    f7c10000-f7c101ff : 0000:06:00.0
      f7c10000-f7c101ff : ahci
  f7d00000-f7dfffff : PCI Bus 0000:03
    f7d00000-f7d3ffff : 0000:03:00.0
      f7d00000-f7d3ffff : alx
...
...
...
```

这些地址中的一部分源于对 `e820_reserve_resources` 函数的调用。我们可以在 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c) 中找到对这个函数的调用，这个函数本身定义在 [arch/x86/kernel/e820.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/e820.c) 中。这个函数遍历了 [e820](http://en.wikipedia.org/wiki/E820) 的映射然后将内存区域插入了根 `iomen` 结构体中。所有具有以下类型的 `e820` 内存区域都会被插入到 `iomem` 结构体中：

```C
static inline const char *e820_type_to_string(int e820_type)
{
	switch (e820_type) {
	case E820_RESERVED_KERN:
	case E820_RAM:	return "System RAM";
	case E820_ACPI:	return "ACPI Tables";
	case E820_NVS:	return "ACPI Non-volatile Storage";
	case E820_UNUSABLE:	return "Unusable memory";
	default:	return "reserved";
	}
}
```

我们可以在 `/proc/iomem` 中看到它们。

现在让我们尝试着理解 `ioremap` 是如何工作的。我们已经了解了一部分 `ioremap` 的知识，我们在[内核初始化的第五部分](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-5.html)见过它。如果你读了那个章节，你就会记得 [arch/x86/mm/ioremap.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/ioremap.c) 文件中对 `early_ioremap_init` 函数的调用。对 `ioremap` 的初始化分为两个部分：有一部分在我们正常使用 `ioremap` 之前，但是要首先进行 `vmalloc` 的初始化并调用 `paging_init` 才能进行正常的 `ioremap` 调用。我们现在还不了解 `vmalloc` 的知识，先看看第一部分的初始化。首先 `early_ioremap_init` 会检查固定映射是否与页中部目录对齐：

```C
BUILD_BUG_ON((fix_to_virt(0) + PAGE_SIZE) & ((1 << PMD_SHIFT) - 1));
```

更多关于 `BUILD_BUG_ON` 的内容你可以在[内核初始化的第一部分](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-1.html)看到。如果给定的表达式为真，`BUILD_BUG_ON` 宏就会抛出一个编译时错误。在检查后的下一步，我们可以看到对 `early_ioremap_setup` 函数的调用，这个函数定义在 [mm/early_ioremap.c](https://github.com/torvalds/linux/blob/master/mm/early_ioremap.c) 文件中。这个函数代表了对 `ioremap` 的大体初始化。`early_ioremap_setup` 函数用初期固定映射的地址填充了 `slot_virt` 数组。所有初期固定映射地址在内存中都在 `__end_of_permanent_fixed_addresses` 后面，它们从 `FIX_BITMAP_BEGIN` 开始，到 `FIX_BITMAP_END` 结束。实际上初期 `ioremap` 会使用 `512` 个临时引导时映射：

```
#define NR_FIX_BTMAPS		64
#define FIX_BTMAPS_SLOTS	8
#define TOTAL_FIX_BTMAPS	(NR_FIX_BTMAPS * FIX_BTMAPS_SLOTS)
```

`early_ioremap_setup` 如下：

```C
void __init early_ioremap_setup(void)
{
        int i;

        for (i = 0; i < FIX_BTMAPS_SLOTS; i++)
                if (WARN_ON(prev_map[i]))
                        break;

        for (i = 0; i < FIX_BTMAPS_SLOTS; i++)
                slot_virt[i] = __fix_to_virt(FIX_BTMAP_BEGIN - NR_FIX_BTMAPS*i);
}
```

`slot_virt` 和其他数组定义在同一个源文件中：

```C
static void __iomem *prev_map[FIX_BTMAPS_SLOTS] __initdata;
static unsigned long prev_size[FIX_BTMAPS_SLOTS] __initdata;
static unsigned long slot_virt[FIX_BTMAPS_SLOTS] __initdata;
```

`slot_virt` 包含了固定映射区域的虚拟地址，`prev_map` 数组包含了初期 `ioremap` 区域的地址。注意我在上文中提到的：`实际上初期 ioremap 会使用 512 个临时引导时映射`，同时你可以看到所有的数组都使用 `__initdata` 定义，这意味着这些内存都会在内核初始化结束后释放掉。在 `early_ioremap_setup` 结束后，我们获得了页中部目录，以 `early_ioremap_pmd` 函数开始的早期 `ioremap`，`early_ioremap_pmd` 函数只能获得内存全局目录以及为给定地址计算页中部目录：

```C
static inline pmd_t * __init early_ioremap_pmd(unsigned long addr)
{
	pgd_t *base = __va(read_cr3());
	pgd_t *pgd = &base[pgd_index(addr)];
	pud_t *pud = pud_offset(pgd, addr);
	pmd_t *pmd = pmd_offset(pud, addr);
	return pmd;
}
```

之后我们用 0 填充 `bm_pte` (早期 `ioremap` 页表入口)，然后调用 `pmd_populate_kernel` 函数：

```C
pmd = early_ioremap_pmd(fix_to_virt(FIX_BTMAP_BEGIN));
memset(bm_pte, 0, sizeof(bm_pte));
pmd_populate_kernel(&init_mm, pmd, bm_pte);
```

`pmd_populate_kernel` 函数有三个参数:

* `init_mm` - `init` 进程的内存描述符 (你可以在[前文](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-5.html)中看到)；
* `pmd`     - `ioremap` 固定映射开始处的页中部目录；
* `bm_pte`  - 初期 `ioremap` 页表入口数组定义为：

```C
static pte_t bm_pte[PAGE_SIZE/sizeof(pte_t)] __page_aligned_bss;
```

 `pmd_popularte_kernel` 函数定义在 [arch/x86/include/asm/pgalloc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/pgalloc.h) 中。它会用给定的页表入口(`bm_pte`)生成给定页中部目录(`pmd`):

```C
static inline void pmd_populate_kernel(struct mm_struct *mm,
                                       pmd_t *pmd, pte_t *pte)
{
        paravirt_alloc_pte(mm, __pa(pte) >> PAGE_SHIFT);
        set_pmd(pmd, __pmd(__pa(pte) | _PAGE_TABLE));
}
```

`set_pmd` 声明如下：

```C
#define set_pmd(pmdp, pmd)              native_set_pmd(pmdp, pmd)
```

`native_set_pmd` 声明如下：

```C
static inline void native_set_pmd(pmd_t *pmdp, pmd_t pmd)
{
        *pmdp = pmd;
}
```

到这里 初期 `ioremap` 就可以使用了。在 `early_ioremap_init` 函数中有许多检查，但是都不重要，总之 `ioremap` 的初始化结束了。

初期输入输出重映射的使用
--------------------------------------------------------------------------------

初期 `ioremap` 初始化完成后，我们就能使用它了。它提供了两个函数：

* early_ioremap
* early_iounmap

用于从 IO 物理地址 映射/解除映射 到虚拟地址。这俩函数都依赖于 `CONFIG_MMU` 编译配置选项。[内存管理单元](http://en.wikipedia.org/wiki/Memory_management_unit)是内存管理的一种特殊块。这种块的主要用途是将物理地址转换为虚拟地址。技术上看内存管理单元可以从 `cr3` 控制寄存器中获取高等级页表地址(`pgd`)。如果 `CONFIG_MMU` 选项被设为 `n`，`early_ioremap` 就会直接返回物理地址，而 `early_iounmap` 就会什么都不做。另一方面，如果设为 `y` ，`early_ioremap` 就会调用 `__early_ioremap`，它有三个参数：

* `phys_addr` - 要映射到虚拟地址上的 I/O 内存区域的基物理地址；
* `size` - I/O 内存区域的尺寸；
* `prot` - 页表入口位。

在 `__early_ioremap` 中我们首先遍历了所有初期 `ioremap` 固定映射槽并检查 `prev_map` 数组中第一个空闲元素，然后将这个值存在了 `slot` 变量中，另外设置了尺寸：

```C
slot = -1;
for (i = 0; i < FIX_BTMAPS_SLOTS; i++) {
	if (!prev_map[i]) {
		slot = i;
		break;
	}
}
...
...
...
prev_size[slot] = size;
last_addr = phys_addr + size - 1;
```

在下一步中我们会看到以下代码：

```C
offset = phys_addr & ~PAGE_MASK;
phys_addr &= PAGE_MASK;
size = PAGE_ALIGN(last_addr + 1) - phys_addr;
```

在这里我们使用了 `PAGE_MASK` 用于清空除前 12 位之外的整个 `phys_addr`。`PAGE_MASK` 宏定义如下：

```C
#define PAGE_MASK       (~(PAGE_SIZE-1))
```

我们知道页的尺寸是 4096 个字节或用二进制表示为 `1000000000000` 。`PAGE_SIZE - 1` 就会是 `111111111111` ，但是使用 `~` 运算后我们就会得到 `000000000000` ，然后使用 `~PAGE_MASK` 又会返回 `111111111111` 。在第二行我们做了同样的事情但是只是清空了前 12 个位，然后在第三行获取了这个区域的页对齐尺寸。我们获得了对齐区域，接下来就需要获取新的 `ioremap` 区域所占用的页的数量然后计算固定映射下标：

```C
nrpages = size >> PAGE_SHIFT;
idx = FIX_BTMAP_BEGIN - NR_FIX_BTMAPS*slot;
```

现在我们用给定的物理地址填充了固定映射区域。循环中的每一次迭代，我们都调用一次 [arch/x86/mm/ioremap.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/ioremap.c) 中的 `__early_set_fixmap` 函数，为给定的物理地址加上页的大小 `4096`，然后更新下标和页的数量：

```C
while (nrpages > 0) {
	__early_set_fixmap(idx, phys_addr, prot);
	phys_addr += PAGE_SIZE;
	--idx;
    --nrpages;
}
```

 `__early_set_fixmap` 函数为给定的物理地址获取了页表入口(保存在 `bm_pte` 中，见上文)：

```C
pte = early_ioremap_pte(addr);
```

在 `early_ioremap_pte` 的下一步中我们用 `pgprot_val` 宏检查了给定的页标志，依赖这个标志选择调用 `set_pte` 还是 `pte_clear` ：

```C
if (pgprot_val(flags))
		set_pte(pte, pfn_pte(phys >> PAGE_SHIFT, flags));
	else
		pte_clear(&init_mm, addr, pte);
```

As you can see above, we passed `FIXMAP_PAGE_IO` as flags to the `__early_ioremap`. `FIXMPA_PAGE_IO` expands to the:
就像你看到的，我们将 `FIXMAP_PAGE_IO` 作为标志传入了 `__early_ioremap`。`FIXMPA_PAGE_IO` 从以下

```C
(__PAGE_KERNEL_EXEC | _PAGE_NX)
```

标志拓展而来， 所以我们调用 `set_pte` 来设置页表入口，就像 `set_pmd` 一样，只不过用于 `PTE`(见上文)。我们在循环中设定了所有 `PTE`，我们可以看到 `__flush_tlb_one` 的函数调用：

```C
__flush_tlb_one(addr);
```

这个函数定义在 [arch/x86/include/asm/tlbflush.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/tlbflush.h)中，并通过判断 `cpu_has_invlpg` 的值来决定调用 `__flush_tlb_single` 还是 `__flush_tlb` ：

```C
static inline void __flush_tlb_one(unsigned long addr)
{
        if (cpu_has_invlpg)
                __flush_tlb_single(addr);
        else
                __flush_tlb();
}
```

`__flush_tlb_one` 函数使 [TLB](http://en.wikipedia.org/wiki/Translation_lookaside_buffer) 中的给定地址失效。就像你看到的我们更新了页结构，但是 `TLB` 还没有改变，这就是我们需要手动做这件事情的原因。有两种方法做这件事。第一种是更新 `cr3` 寄存器， `__flush_tlb` 函数就是这么做的：

```C
native_write_cr3(native_read_cr3());
```

第二种方法是使用 `invlpg` 命令来使 `TLB` 入口失效。让我们看看  `__flush_tlb_one` 的实现。就像我们所看到的，它首先检查了 `cpu_has_invlpg` ，定义如下：

```C
#if defined(CONFIG_X86_INVLPG) || defined(CONFIG_X86_64)
# define cpu_has_invlpg         1
#else
# define cpu_has_invlpg         (boot_cpu_data.x86 > 3)
#endif
```

如果 CPU 支持 `invlpg` 指令，我们就调用 `__flush_tlb_single` 宏，它拓展自 `__native_flush_tlb_single`：

```C
static inline void __native_flush_tlb_single(unsigned long addr)
{
        asm volatile("invlpg (%0)" ::"r" (addr) : "memory");
}
```

`__flush_tlb` 的调用知识更新了 `cr3` 寄存器。在这步结束之后 `__early_set_fixmap` 函数就执行完了，我们又可以回到 `__early_ioremap` 的实现了。因为我们为给定的地址设定了固定映射区域，我们需要将 I/O 重映射的区域的基虚拟地址用 `slot` 下标保存在 `prev_map` 数组中。

```C
prev_map[slot] = (void __iomem *)(offset + slot_virt[slot]);
```

然后返回它。

第二个函数是 `early_iounmap` ，它会解除对一个 I/O 内存区域的映射。这个函数有两个参数：基地址和 I/O 区域的大小，这看起来与 `early_ioremap` 很像。它同样遍历了固定映射槽并寻找给定地址的槽。这样它就获得了这个固定映射槽的下标，然后通过判断 `after_paging_init` 的值决定是调用 `__late_clear_fixmap` 还是 `__early_set_fixmap` 。当这个值是 0 时会调用 `__early_set_fixmap`。最终它会将 I/O 内存区域设为 `NULL`：

```C
prev_map[slot] = NULL;
```

这就是关于 `fixmap` 和 `ioremap` 的全部内容。当然这部分不可能包含所有 `ioremap` 的特性，仅仅是讲解了初期 `ioremap`，常规的 `ioremap` 没有讲。这主要是因为在讲解它之前需要了解更多内容才行。

就是这样！

结束语
--------------------------------------------------------------------------------

讲解内核内存管理的第一部分到此结束，如果你有任何的问题或者建议，你可以直接发消息给我[twitter](https://twitter.com/0xAX)，也可以给我发[邮件](anotherworldofworld@gmail.com)或是直接创建一个 [issue](https://github.com/MintCN/linux-insides-zh/issues/new)。

**英文不是我的母语。如果你发现我的英文描述有任何问题，请提交一个PR到[linux-insides](https://github.com/MintCN/linux-insides-zh).**

相关连接：
--------------------------------------------------------------------------------

* [apic](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [vsyscall](https://lwn.net/Articles/446528/)
* [Intel Trusted Execution Technology](http://en.wikipedia.org/wiki/Trusted_Execution_Technology)
* [Xen](http://en.wikipedia.org/wiki/Xen)
* [Real Time Clock](http://en.wikipedia.org/wiki/Real-time_clock)
* [e820](http://en.wikipedia.org/wiki/E820)
* [Memory management unit](http://en.wikipedia.org/wiki/Memory_management_unit)
* [TLB](http://en.wikipedia.org/wiki/Translation_lookaside_buffer)
* [Paging](http://xinqiu.gitbooks.io/linux-insides-cn/content/Theory/linux-theory-1.html)
* [内核内存管理第一部分](http://xinqiu.gitbooks.io/linux-insides-cn/content/MM/linux-mm-1.html)
