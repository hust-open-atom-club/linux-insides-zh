内核引导过程. Part 6.
================================================================================

简介
--------------------------------------------------------------------------------

这是`内核引导过程`系列文章的第六部分。在[前一部分](linux-bootstrap-5.md)，我们已经看到了内核引导过程的结尾，但是我们跳过了一些高级部分。

你可能还记得，Linux内核的入口点是 [main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 的`start_kernel`函数，它在`LOAD_PHYSICAL_ADDR`地址开始执行。这个地址依赖于`CONFIG_PHYSICAL_START`内核配置选项，默认为`0x1000000`:

```
config PHYSICAL_START
	hex "Physical address where the kernel is loaded" if (EXPERT || CRASH_DUMP)
	default "0x1000000"
	---help---
	  This gives the physical address where the kernel is loaded.
      ...
      ...
      ...
```

这个选项在内核配置时可以修改，但是加载地址可以选择为一个随机值。为此，`CONFIG_RANDOMIZE_BASE`内核配置选项在内核配置时应该启用。

在这种情况下，Linux内核镜像解压和加载的物理地址会被随机化。我们在这一部分考虑这个选项被启用，并且为了[安全原因](https://en.wikipedia.org/wiki/Address_space_layout_randomization)，内核镜像的加载地址被随机化的情况。

页表的初始化
--------------------------------------------------------------------------------

在内核解压器要开始找随机的内核解压和加载地址之前，应该初始化恒等映射（identity mapped,虚拟地址和物理地址相同）页表。如果[引导加载器](https://en.wikipedia.org/wiki/Booting)使用[16位或32位引导协议](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt)，那么我们已经有了页表。但在任何情况下，如果内核解压器选择它们之外的内存区域，我们需要新的页。这就是为什么我们需要建立新的恒等映射页表。

是的，建立恒等映射页表是随机化加载地址的最早的步骤之一。但是在此之前，让我们回忆一下我们是怎么来到这里的。

在[前一部分](linux-bootstrap-5.md)，我们看到了到[长模式](https://en.wikipedia.org/wiki/Long_mode)的转换，并跳转到了内核解压器的入口点——`extract_kernel`函数。随机化从调用这个函数开始：

```C
void choose_random_location(unsigned long input,
                            unsigned long input_size,
			                unsigned long *output,
                            unsigned long output_size,
			                unsigned long *virt_addr)
{}
```

你可以看到，这个函数有五个参数：

  * `input`;
  * `input_size`;
  * `output`;
  * `output_isze`;
  * `virt_addr`.

让我们试着理解一下这些参数是什么。第一个`input`参数来自源文件 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c) 里的`extract_kernel`函数：

```C
asmlinkage __visible void *extract_kernel(void *rmode, memptr heap,
				                          unsigned char *input_data,
				                          unsigned long input_len,
				                          unsigned char *output,
				                          unsigned long output_len)
{
  ...
  ...
  ...
  choose_random_location((unsigned long)input_data, input_len,
                         (unsigned long *)&output,
				         max(output_len, kernel_total_size),
				         &virt_addr);
  ...
  ...
  ...
}
```

这个参数由 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 的汇编代码传递：

```C
leaq	input_data(%rip), %rdx
```

`input_data`由 [mkpiggy](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/mkpiggy.c) 程序生成。如果你亲手编译过Linux内核源码，你会找到这个程序生成的文件，它应该位于 `linux/arch/x86/boot/compressed/piggy.S`. 在我这里，这个文件是这样的：

```assembly
.section ".rodata..compressed","a",@progbits
.globl z_input_len
z_input_len = 6988196
.globl z_output_len
z_output_len = 29207032
.globl input_data, input_data_end
input_data:
.incbin "arch/x86/boot/compressed/vmlinux.bin.gz"
input_data_end:
```

你能看到它有四个全局符号。前两个`z_input_len`和`z_output_len`是压缩的和解压后的`vmlinux.bin.gz`的大小。第三个是我们的`input_data`，你可以看到，它指向二进制格式（去掉所有调试符号、注释和重定位信息）的Linux内核镜像。最后的`input_data_end`指向压缩的Linux镜像的末尾。

所以我们`choose_random_location`函数的第一个参数是指向嵌入在`piggy.o`目标文件的压缩的内核镜像的指针。

`choose_random_location`函数的第二个参数是我们刚刚看到的`z_input_len`.

`choose_random_location`函数的第三和第四个参数分别是解压后的内核镜像的位置和长度。放置解压后内核的地址来自 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S)，并且它是`startup_32`对齐到 2MB 边界的地址。解压后的内核的大小来自同样的`piggy.S`，并且它是`z_output_len`.

`choose_random_location`函数的最后一个参数是内核加载地址的虚拟地址。我们可以看到，它和默认的物理加载地址相同：

```C
unsigned long virt_addr = LOAD_PHYSICAL_ADDR;
```

它依赖于内核配置：

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

现在，由于我们考虑`choose_random_location`函数的参数，让我们看看它的实现。这个函数从检查内核命令行的`nokaslr`选项开始：

```C
if (cmdline_find_option_bool("nokaslr")) {
	warn("KASLR disabled: 'nokaslr' on cmdline.");
	return;
}
```

如果有这个选项，那么我们就退出`choose_random_location`函数，并且内核的加载地址不会随机化。相关的命令行选项可以在[内核文档](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/kernel-parameters.txt)找到：

```
kaslr/nokaslr [X86]

Enable/disable kernel and module base offset ASLR
(Address Space Layout Randomization) if built into
the kernel. When CONFIG_HIBERNATION is selected,
kASLR is disabled by default. When kASLR is enabled,
hibernation will be disabled.
```

假设我们没有把`nokaslr`传到内核命令行，并且`CONFIG_RANDOMIZE_BASE`启用了内核配置选项。

下一步是以下函数的调用：

```C
initialize_identity_maps();
```

它在 [arch/x86/boot/compressed/pagetable.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/pagetable.c) 源码文件定义。这个函数从初始化`mapping_info`,`x86_mapping_info`结构体的一个实例开始。

```C
mapping_info.alloc_pgt_page = alloc_pgt_page;
mapping_info.context = &pgt_data;
mapping_info.page_flag = __PAGE_KERNEL_LARGE_EXEC | sev_me_mask;
mapping_info.kernpg_flag = _KERNPG_TABLE | sev_me_mask;
```

`x86_mapping_info`结构体在 [arch/x86/include/asm/init.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/init.h) 头文件定义：

```C
struct x86_mapping_info {
	void *(*alloc_pgt_page)(void *);
	void *context;
	unsigned long page_flag;
	unsigned long offset;
	bool direct_gbpages;
	unsigned long kernpg_flag;
};
```

这个结构体提供了关于内存映射的信息。你可能还记得，在前面的部分，我们已经建立了初始的从0到`4G`的页表。现在我们可能需要访问`4G`以上的内存来在随机的位置加载内核。所以，`initialize_identity_maps`函数初始化一个内存区域，它用于可能需要的新页表。首先，让我们尝试查看`x86_mapping_info`结构体的定义。

`alloc_pgt_page`是一个会在为一个页表项分配空间时调用的回调函数。`context`域是一个用于跟踪已分配页表的`alloc_pgt_data`结构体的实例。`page_flag`和`kernpg_flag`是页标志。第一个代表`PMD`或`PUD`表项的标志。第二个`kernpg_flag`域代表会在之后被覆盖的内核页的标志。`direct_gbpages`域代表对大页的支持。最后的`offset`域代表内核虚拟地址到`PMD`级物理地址的偏移。

`alloc_pgt_page`回调函数检查有一个新页的空间，从缓冲区分配新页并返回新页的地址：


```C
entry = pages->pgt_buf + pages->pgt_buf_offset;
pages->pgt_buf_offset += PAGE_SIZE;
```

缓冲区在此结构体中：

```C
struct alloc_pgt_data {
	unsigned char *pgt_buf;
	unsigned long pgt_buf_size;
	unsigned long pgt_buf_offset;
};
```

`initialize_identity_maps`函数最后的目标是初始化`pgdt_buf_size`和`pgt_buf_offset`. 由于我们只是在初始化阶段，`initialize_identity_maps`函数设置`pgt_buf_offset`为0:

```C
pgt_data.pgt_buf_offset = 0;
```

而`pgt_data.pgt_buf_size`会根据引导加载器所用的引导协议（64位或32位）被设置为`77824`或`69632`. `pgt_data.pgt_buf`也是一样。如果引导加载器在`startup_32`引导内核，`pgdt_data.pgdt_buf`会指向已经在 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 初始化的页表的末尾：

```C
pgt_data.pgt_buf = _pgtable + BOOT_INIT_PGT_SIZE;
```

其中`_pgtable`指向这个页表 [_pgtable](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/vmlinux.lds.S) 的开头。另一方面，如果引导加载器用64位引导协议并在`startup_64`加载内核，早期页表应该由引导加载器建立，并且`_pgtable`会被重写：

```C
pgt_data.pgt_buf = _pgtable
```

在新页表的缓冲区被初始化之下，我们回到`choose_random_location`函数。

避开保留的内存范围
--------------------------------------------------------------------------------

在恒等映射页表相关的数据被初始化之后，我们可以开始选择放置解压后内核的随机位置。但是正如你猜的那样，我们不能选择任意地址。在内存的范围中，有一些保留的地址。这些地址被重要的东西占用，如[initrd](https://en.wikipedia.org/wiki/Initial_ramdisk), 内核命令行等等。这个函数：

```C
mem_avoid_init(input, input_size, *output);
```

会帮我们做这件事。所有不安全的内存区域会收集到：

```C
struct mem_vector {
	unsigned long long start;
	unsigned long long size;
};

static struct mem_vector mem_avoid[MEM_AVOID_MAX];
```

数组。其中`MEM_AVOID_MAX`来自[枚举类型](https://en.wikipedia.org/wiki/Enumerated_type#C)`mem_avoid_index`, 它代表不同类型的保留内存区域：

```C
enum mem_avoid_index {
	MEM_AVOID_ZO_RANGE = 0,
	MEM_AVOID_INITRD,
	MEM_AVOID_CMDLINE,
	MEM_AVOID_BOOTPARAMS,
	MEM_AVOID_MEMMAP_BEGIN,
	MEM_AVOID_MEMMAP_END = MEM_AVOID_MEMMAP_BEGIN + MAX_MEMMAP_REGIONS - 1,
	MEM_AVOID_MAX,
};
```

它们都定义在源文件 [arch/x86/boot/compressed/kaslr.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/kaslr.c) 中。

让我们看看`mem_avoid_init`函数的实现。这个函数的主要目标是在`mem_avoid`数组存放关于被`mem_avoid_index`枚举类型描述的保留内存区域的信息，并且在我们新的恒等映射缓冲区为这样的区域创建新页。`mem_avoid_index`函数的几个部分很相似，但是先看看其中一个：

```C
mem_avoid[MEM_AVOID_ZO_RANGE].start = input;
mem_avoid[MEM_AVOID_ZO_RANGE].size = (output + init_size) - input;
add_identity_map(mem_avoid[MEM_AVOID_ZO_RANGE].start,
		 mem_avoid[MEM_AVOID_ZO_RANGE].size);
```

`mem_avoid_init`函数的开头尝试避免用于当前内核解压的内存区域。我们用这个区域的起始地址和大小填写`mem_avoid`数组的一项，并调用`add_identity_map`函数，它会为这个区域建立恒等映射页。`add_identity_map`函数在源文件 [arch/x86/boot/compressed/kaslr.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/kaslr.c) 定义：

```C
void add_identity_map(unsigned long start, unsigned long size)
{
	unsigned long end = start + size;

	start = round_down(start, PMD_SIZE);
	end = round_up(end, PMD_SIZE);
	if (start >= end)
		return;

	kernel_ident_mapping_init(&mapping_info, (pgd_t *)top_level_pgt,
				  start, end);
}
```

你可以看到，它对齐内存到 2MB 边界并检查给定的起始地址和终止地址。

最后它调用`kernel_ident_mapping_init`函数，它在源文件 [arch/x86/mm/ident_map.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/ident_map.c) 中，并传入以上初始化好的`mapping_info`实例、顶层页表的地址和建立新的恒等映射的内存区域的地址。

`kernel_ident_mapping_init`函数为新页设置默认的标志，如果它们没有被给出：

```C
if (!info->kernpg_flag)
	info->kernpg_flag = _KERNPG_TABLE;
```

并且开始建立新的2MB (因为`mapping_info.page_flag`中的`PSE`位) 给定地址相关的页表项（[五级页表](https://lwn.net/Articles/717293/)中的`PGD -> P4D -> PUD -> PMD`或者[四级页表](https://lwn.net/Articles/117749/)中的`PGD -> PUD -> PMD`）。

```C
for (; addr < end; addr = next) {
	p4d_t *p4d;

	next = (addr & PGDIR_MASK) + PGDIR_SIZE;
	if (next > end)
		next = end;

    p4d = (p4d_t *)info->alloc_pgt_page(info->context);
	result = ident_p4d_init(info, p4d, addr, next);

    return result;
}
```

首先我们找给定地址在 `页全局目录` 的下一项，如果它大于给定的内存区域的末地址`end`，我们把它设为`end`.之后，我们用之前看过的`x86_mapping_info`回调函数分配一个新页，然后调用`ident_p4d_init`函数。`ident_p4d_init`函数做同样的事情，但是用于低层的页目录 (`p4d` -> `pud` -> `pmd`).

就是这样。

和保留地址相关的新页表项已经在我们的页表中。这不是`mem_avoid_init`函数的末尾，但是其他部分类似。它建立用于 [initrd](https://en.wikipedia.org/wiki/Initial_ramdisk)、内核命令行等数据的页。

现在我们可以回到`choose_random_location`函数。

物理地址随机化
--------------------------------------------------------------------------------

在保留内存区域存储在`mem_avoid`数组并且为它们建立了恒等映射页之后，我们选择最小可用的地址作为解压内核的随机内存区域：

```C
min_addr = min(*output, 512UL << 20);
```

你可以看到，它应该小于512MB. 选择这个512MB的值只是避免低内存区域中未知的东西。

下一步是选择随机的物理和虚拟地址来加载内核。首先是物理地址：

```C
random_addr = find_random_phys_addr(min_addr, output_size);
```

`find_random_phys_addr`函数在[同一个](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/kaslr.c)源文件中定义：

```
static unsigned long find_random_phys_addr(unsigned long minimum,
                                           unsigned long image_size)
{
	minimum = ALIGN(minimum, CONFIG_PHYSICAL_ALIGN);

	if (process_efi_entries(minimum, image_size))
		return slots_fetch_random();

	process_e820_entries(minimum, image_size);
	return slots_fetch_random();
}
```

`process_efi_entries`函数的主要目标是在整个可用的内存找到所有的合适的内存区域来加载内核。如果内核没有在支持[EFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)的系统中编译和运行，我们继续在[e820](https://en.wikipedia.org/wiki/E820)区域中找这样的内存区域。所有找到的内存区域会存储在

```C
struct slot_area {
	unsigned long addr;
	int num;
};

#define MAX_SLOT_AREA 100

static struct slot_area slot_areas[MAX_SLOT_AREA];
```

数组中。内核解压器应该选择这个数组随机的索引，并且它会是内核解压的随机位置。这个选择会被`slots_fetch_random`函数执行。`slots_fetch_random`函数的主要目标是通过`kaslr_get_random_long`函数从`slot_areas`数组选择随机的内存范围：

```C
slot = kaslr_get_random_long("Physical") % slot_max;
```

`kaslr_get_random_long`函数在源文件 [arch/x86/lib/kaslr.c](https://github.com/torvalds/linux/blob/master/arch/x86/lib/kaslr.c) 中定义，它返回一个随机数。注意这个随机数会通过不同的方式得到，取决于内核配置、系统机会（基于[时间戳计数器](https://en.wikipedia.org/wiki/Time_Stamp_Counter)的随机数、[rdrand](https://en.wikipedia.org/wiki/RdRand)等等）。

这就是随机内存范围的选择方法。

虚拟地址随机化
--------------------------------------------------------------------------------

在内核解压器选择了随机内存区域后，新的恒等映射页会为这个区域按需建立：

```C
random_addr = find_random_phys_addr(min_addr, output_size);

if (*output != random_addr) {
		add_identity_map(random_addr, output_size);
		*output = random_addr;
}
```

这时，`output`会存放内核将会解压的一个内存区域的基地址。但是现在，正如你还记得的那样，我们只是随机化了物理地址。在[x86_64](https://en.wikipedia.org/wiki/X86-64)架构，虚拟地址也应该被随机化：

```C
if (IS_ENABLED(CONFIG_X86_64))
	random_addr = find_random_virt_addr(LOAD_PHYSICAL_ADDR, output_size);

*virt_addr = random_addr;
```

正如你所看到的，对于非`x86_64`架构，随机化的虚拟地址和随机化的物理地址相同。`find_random_virt_addr`函数计算可以保存内存镜像的虚拟内存范围的数量并且调用我们在尝试找到随机的`物理`地址的时候，之前已经看到的`kaslr_get_random_long`函数。

这时，我们同时有了用于解压内核的随机化的物理(`*output`)和虚拟(`*virt_addr`)基地址。

就是这样。

结论
--------------------------------------------------------------------------------

这是关于Linux内核引导过程的第六，并且是最后一部分的结尾。我们不再会看到关于内核引导的帖子（可能有对这篇和之前文章的更新），但是会有很多关于其他内核内部细节的文章。

下一章是关于内核初始化的，我们会看到Linux内核初始化代码的早期步骤。

如果你有什么问题或建议，写个评论或在 [twitter](https://twitter.com/0xAX) 找我。

**如果你发现文中描述有任何问题，请提交一个 PR 到 [linux-insides-zh](https://github.com/MintCN/linux-insides-zh) 。**

Links
--------------------------------------------------------------------------------

* [Address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization)
* [Linux kernel boot protocol](https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.txt)
* [long mode](https://en.wikipedia.org/wiki/Long_mode)
* [initrd](https://en.wikipedia.org/wiki/Initial_ramdisk)
* [Enumerated type](https://en.wikipedia.org/wiki/Enumerated_type#C)
* [four-level page tables](https://lwn.net/Articles/117749/)
* [five-level page tables](https://lwn.net/Articles/717293/)
* [EFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)
* [e820](https://en.wikipedia.org/wiki/E820)
* [time stamp counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [rdrand](https://en.wikipedia.org/wiki/RdRand)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [Previous part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-5.md)
