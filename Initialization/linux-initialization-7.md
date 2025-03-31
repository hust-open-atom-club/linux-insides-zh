内核初始化 第七部分
================================================================================

架构相关初始化尾声：最后一程
================================================================================

这是 Linux 内核初始化过程的第七部分，主要解析 [arch/x86/Kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c#L861) 文件中 `setup_arch` 函数内部机制。通过[前文](https://0xax.gitbook.io/linux-insides/summary/initialization)可知，`setup_arch` 函数执行架构相关（本文以 [x86_64](http://en.wikipedia.org/wiki/X86-64) 为例）的初始化工作，包括为内核代码/数据/bss 保留内存，[桌面管理界面](http://en.wikipedia.org/wiki/Desktop_Management_Interface)的早期扫描，[PCI](http://en.wikipedia.org/wiki/PCI) 设备的早期转储等众多操作。如果您已经阅读了前面的[部分](https://0xax.gitbook.io/linux-insides/summary/initialization/linux-initialization-6)，会记得我们是在 `setup_real_mode` 函数处结束的。接下来，在我们限制 [memblock](https://0xax.gitbook.io/linux-insides/summary/mm/linux-mm-1) 为所有已映射页后，可以看到 [kernel/printk/printk.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/printk/printk.c) 中调用了 `setup_log_buf` 函数。

`setup_log_buf` 函数用于设置内核循环缓冲区，其长度取决于 `CONFIG_LOG_BUF_SHIFT` 配置选项。从文档中可知，`CONFIG_LOG_BUF_SHIFT `取值范围在 `12` 到 `21` 之间。在内部实现中，该缓冲区定义为字符数组：

```c
#define __LOG_BUF_LEN (1 << CONFIG_LOG_BUF_SHIFT)
static char __log_buf[__LOG_BUF_LEN] __aligned(LOG_ALIGN);
static char *log_buf = __log_buf;
```

现在我们来看 `setup_log_buf` 函数的实现。它首先会检查当前缓冲区是否为空（由于刚刚初始化，此时缓冲区必须为空），同时还会检查是否为早期初始化阶段。如果内核日志缓冲区的设置不属于早期初始化阶段，则会调用 `log_buf_add_cpu` 函数，该函数会为每个 CPU 扩展缓冲区的大小：

```C
if (log_buf != __log_buf)
    return;

if (!early && !new_log_buf_len)
    log_buf_add_cpu();
```

这里我们暂不深入分析 `log_buf_add_cpu` 函数，因为正如 `setup_arch` 中所示，我们是通过以下方式调用 `setup_log_buf` 的：

```C
setup_log_buf(1);
```

其中参数 `1` 表示当前处于早期初始化阶段。接下来，我们会检查 `new_log_buf_len` 变量（该变量表示更新后的内核日志缓冲区长度），并通过 `memblock_virt_alloc` 函数为其分配新的缓冲区空间，否则直接返回。

当内核日志缓冲区准备就绪后，下一个执行的是 `reserve_initrd` 函数。您可能还记得，在[内核初始化第四部分](https://0xax.gitbook.io/linux-insides/summary/initialization/linux-initialization-4)中我们已经调用过 `early_reserve_initrd` 函数。现在，由于我们已在 `init_mem_mapping` 函数中重建了直接内存映射，因此需要将[初始RAM磁盘](http://en.wikipedia.org/wiki/Initrd)移入直接映射内存区域。`reserve_initrd` 函数首先确定 initrd 的基地址和结束地址，并检查 bootloader 是否提供了 initrd —— 这些操作与我们在 `early_reserve_initrd` 中看到的完全一致。但不同于之前通过调用 `memblock_reserve` 在 memblock 区域中保留空间的做法，这里我们会获取直接内存映射区域的映射大小，并通过以下方式确保 initrd 的大小不超过该区域：

```C
mapped_size = memblock_mem_size(max_pfn_mapped);
if (ramdisk_size >= (mapped_size>>1))
    panic("initrd too large to handle, "
	      "disabling initrd (%lld needed, %lld available)\n",
	      ramdisk_size, mapped_size>>1);
```

可以看到，这里我们调用了 `memblock_mem_size` 函数，并将 `max_pfn_mapped` 作为参数传入。其中，`max_pfn_mapped` 存储的是当前直接映射的最高页帧号（Page Frame Number）。若不记得什么是页帧号，这里简单说明：虚拟地址的低 `12` 位表示物理页（页帧）的偏移量。当我们右移虚拟地址的 `12` 位时，将丢弃偏移部分，从而得到页帧号。在 `memblock_mem_size` 函数内部，我们会遍历所有 memblock 的 `mem` 区域（不包括保留区域），计算已映射页面的总大小，并将结果返回给 `mapped_size` 变量（参见上文代码）。获取到直接映射内存的总量后，我们会检查 `initrd` 的大小是否超过已映射的页面范围。如果超出，则直接调用 `panic` 函数终止系统运行，并打印著名的[内核恐慌](http://en.wikipedia.org/wiki/Kernel_panic)信息。

接下来，我们会打印关于 `initrd` 大小的信息。通过 `dmesg` 命令的输出可以看到如下结果：

```C
[0.000000] RAMDISK: [mem 0x36d20000-0x37687fff]
```

随后通过 `relocate_initrd` 函数将 `initrd` 重定位到直接映射区域。在 `relocate_initrd` 函数起始处，我们会尝试使用 `memblock_find_in_range` 函数来寻找可用内存区域：

```C
relocated_ramdisk = memblock_find_in_range(0, PFN_PHYS(max_pfn_mapped), area_size, PAGE_SIZE);

if (!relocated_ramdisk)
    panic("Cannot find place for new RAMDISK of size %lld\n",
	       ramdisk_size);
```

`memblock_find_in_range` 函数会尝试在指定范围内（本例中是从 `0` 到最大已映射物理地址）寻找可用区域，且该区域大小必须等于经过对齐处理的 `initrd` 大小。如果未能找到符合要求的区域，系统将再次调用 `panic` 函数终止运行。若成功找到合适区域，我们将在下一步将 RAM 磁盘重定位至直接映射内存的末端。

在 `reserve_initrd` 函数的最后阶段，通过调用以下函数释放原 RAM 磁盘占用的 memblock 内存：

```C
memblock_free(ramdisk_image, ramdisk_end - ramdisk_image);
```

在完成 `initrd` RAM 磁盘镜像的重定位后，接下来执行的是位于 [arch/x86/kernel/vsmp_64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vsmp_64.c) 的 `vsmp_init` 函数。该函数用于初始化 `ScaleMP vSMP` 架构支持。如先前章节所述，本文不会涉及与 `x86_64` 初始化无关的内容（例如当前的 `ACPI` 等）。因此我们将暂时跳过其具体实现，留待后续讲解并行计算技术时再作探讨。

随后调用的是 [arch/x86/kernel/io_delay.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/io_delay.c) 中的 `io_delay_init` 函数。该函数允许覆盖默认的 I/O 延迟端口 `0x80`。我们已在[进入保护模式前的最后准备](https://0xax.gitbook.io/linux-insides/summary/booting/linux-bootstrap-3)中接触过 I/O 延迟的概念，现在让我们深入分析 `io_delay_init` 的具体实现：

```C
void __init io_delay_init(void)
{
    if (!io_delay_override)
        dmi_check_system(io_delay_0xed_port_dmi_table);
}
```

该函数会检查 `io_delay_override` 变量，若该变量被设置，则覆盖默认的 I/O 延迟端口。我们可以通过向内核命令行传递 `io_delay` 参数来设置 `io_delay_override` 变量。根据 [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst) 文档说明，`io_delay` 选项是：

```
io_delay=	[X86] I/O delay method
    0x80
        Standard port 0x80 based delay
    0xed
        Alternate port 0xed based delay (needed on some systems)
    udelay
        Simple two microseconds delay
    none
        No delay
```

可以看到，在 [arch/x86/kernel/io_delay.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/io_delay.c) 文件中，`io_delay` 命令行参数是通过 `early_param` 宏进行设置的：

```C
early_param("io_delay", io_delay_param);
```

关于 `early_param` 宏的更多细节，您可以在[第六部分](https://0xax.gitbook.io/linux-insides/summary/initialization/linux-initialization-6)中查阅。因此，用于设置 `io_delay_override` 变量的 `io_delay_param` 函数将会在 [do_early_param](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c#L413) 函数中被调用。`io_delay_param` 函数通过解析 `io_delay` 内核命令行参数，根据传入值设置相应的 `io_delay_type`：

```C
static int __init io_delay_param(char *s)
{
        if (!s)
                return -EINVAL;

        if (!strcmp(s, "0x80"))
                io_delay_type = CONFIG_IO_DELAY_TYPE_0X80;
        else if (!strcmp(s, "0xed"))
                io_delay_type = CONFIG_IO_DELAY_TYPE_0XED;
        else if (!strcmp(s, "udelay"))
                io_delay_type = CONFIG_IO_DELAY_TYPE_UDELAY;
        else if (!strcmp(s, "none"))
                io_delay_type = CONFIG_IO_DELAY_TYPE_NONE;
        else
                return -EINVAL;

        io_delay_override = 1;
        return 0;
}
```

在 `io_delay_init` 之后，接下来执行的函数依次是 `acpi_boot_table_init`、`early_acpi_boot_init` 和 `initmem_init`。不过正如前文所述，在当前的「Linux 内核初始化流程」章节中，我们将不涉及与 [ACPI](http://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface) 相关的内容。

为DMA分配域
--------------------------------------------------------------------------------

下一步我们需要通过 `dma_contiguous_reserve` 函数为[直接内存访问（DMA）](http://en.wikipedia.org/wiki/Direct_memory_access)分配专用区域，该函数定义于 [drivers/base/dma-contiguous.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/base/dma-contiguous.c)。DMA 是一种特殊工作模式，设备无需 CPU 介入即可直接与内存通信。请注意，我们向 `dma_contiguous_reserve` 函数传递了一个关键参数——`max_pfn_mapped << PAGE_SHIFT`。从该表达式可以理解，此参数表示可保留内存的上限地址（将最大页帧号转换为字节地址）。让我们分析该函数的实现，它从定义以下变量开始：

```C
phys_addr_t selected_size = 0;
phys_addr_t selected_base = 0;
phys_addr_t selected_limit = limit;
bool fixed = false;
```

其中第一个参数表示保留区域的大小（以字节为单位），第二个参数是保留区域的基地址，第三个参数是保留区域的结束地址，最后一个 `fixed` 参数用于指定保留区域的放置方式。当 `fixed` 为 `1` 时，直接通过 `memblock_reserve` 保留内存区域；若为 `0` ，则使用`kmemleak_alloc` 动态分配空间。在接下来的步骤中，函数会检查 `size_cmdline` 变量。若该变量不等于 `-1`（表示已通过命令行参数指定大小），则将使用来自 `cma` 内核命令行参数的值填充上述所有变量：

```C
if (size_cmdline != -1) {
   ...
   ...
   ...
}
```

在该源码文件中可以找到如下早期参数的定义：

```C
early_param("cma", early_cma);
```

其中 `cma` 表示:

```
cma=nn[MG]@[start[MG][-end[MG]]]
		[ARM,X86,KNL]
		Sets the size of kernel global memory area for
		contiguous memory allocations and optionally the
		placement constraint by the physical address range of
		memory allocations. A value of 0 disables CMA
		altogether. For more information, see
		include/linux/dma-contiguous.h
```

如果未向内核命令行传递 `cma` 参数，则 `size_cmdline` 将保持默认值 `-1`。此时，系统需要根据以下内核配置选项来计算保留区域的大小：

* `CONFIG_CMA_SIZE_SEL_MBYTES` -  该选项表示以兆字节（MB）为单位的默认全局连续内存分配器（CMA）区域大小，其计算公式为 `CMA_SIZE_MBYTES * SZ_1M` 或等价的 `CONFIG_CMA_SIZE_MBYTES * 1M`；
* `CONFIG_CMA_SIZE_SEL_PERCENTAGE` - 总内存的百分比；
* `CONFIG_CMA_SIZE_SEL_MIN` - 取较小值；
* `CONFIG_CMA_SIZE_SEL_MAX` - 取较大值；

在计算出保留区域的大小后，系统将通过调用 `dma_contiguous_reserve_area` 函数来实际保留该内存区域。该函数的执行流程首先会调用以下函数：

```
ret = cma_declare_contiguous(base, size, limit, 0, 0, fixed, res_cma);
```

`cma_declare_contiguous` 函数用于从指定的基地址开始保留一块连续的物理内存区域，其大小由参数指定。完成 DMA 区域的内存保留后，接下来会调用 `memblock_find_dma_reserve` 函数——顾名思义，该函数用于统计 DMA 区域中已保留的页框数量。由于 CMA（连续内存分配器）和 DMA 相关实现较为复杂，本文暂不深入探讨所有细节。我们将在后续专门讲解 Linux 内核内存管理的章节中，详细分析连续内存分配器及其内存区域的实现机制。

稀疏内存初始化
--------------------------------------------------------------------------------

接下来将调用 `x86_init.paging.pagetable_init` 函数。若您在内核源码中追溯该函数的实现，最终会发现以下宏定义：

```C
#define native_pagetable_init        paging_init
```

该宏如您所见，会展开调用 [arch/x86/mm/init_64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/mm/init_64.c) 中的 `paging_init` 函数。`paging_init` 函数负责初始化稀疏内存和内存区域大小。首先需要了解什么是内存区域（zone）以及什么是 `Sparsemem`。`Sparsemem` 是 Linux 内核内存管理器中用于在 [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access) 系统中将内存区域划分为不同内存组（memory bank）的特殊基础架构。让我们看看 `paging_init` 函数的实现：

```C
void __init paging_init(void)
{
        sparse_memory_present_with_active_regions(MAX_NUMNODES);
        sparse_init();

        node_clear_state(0, N_MEMORY);
        if (N_MEMORY != N_NORMAL_MEMORY)
                node_clear_state(0, N_NORMAL_MEMORY);

        zone_sizes_init();
}
```

可以看到，这里调用了 `sparse_memory_present_with_active_regions` 函数，该函数会为每个 NUMA 节点记录内存区域到 `mem_section` 结构数组中，该结构包含指向 `struct page` 数组结构的指针。随后的 `sparse_init` 函数会分配非线性的 `mem_section` 和 `mem_map`。接下来我们会清除可移动内存节点的状态并初始化各内存区域(zone)的大小。每个 NUMA 节点都被划分为多个称为"zone"的部分。因此，来自 [arch/x86/mm/init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/mm/init.c) 的 `zone_sizes_init` 函数就是用于初始化这些zone的大小的。

再次说明，本部分及后续部分不会完整详细地涵盖这一主题。关于 NUMA 将有专门的章节进行讲解。

vsyscall 映射
--------------------------------------------------------------------------------

在 `SparseMem` 初始化之后，下一步是设置 `trampoline_cr4_features`，它必须包含 `cr4` [控制寄存器](http://en.wikipedia.org/wiki/Control_register)的内容。首先我们需要检查当前 CPU 是否支持 `cr4` 寄存器，如果支持，就将其内容保存到 `trampoline_cr4_features` ，这是在实模式下存储 `cr4` 的地方。

```C
if (boot_cpu_data.cpuid_level >= 0) {
    mmu_cr4_features = __read_cr4();
	if (trampoline_cr4_features)
	    *trampoline_cr4_features = mmu_cr4_features;
}
```

接下来您会看到的是 [arch/x86/entry/vsyscall/vsyscall_64.c](https://github.com/torvalds/linux/blob/master/arch/x86/entry/vsyscall/vsyscall_64.c) 的 `map_vsyscall` 函数。这个函数为 [vsyscalls](https://lwn.net/Articles/446528/) 映射内存空间，其功能依赖于内核配置选项 `CONFIG_X86_VSYSCALL_EMULATION`。实际上，`vsyscall` 是一个特殊的段，它提供了对某些系统调用(如 `getcpu` 等)的快速访问。该函数的具体实现如下：

```C
void __init map_vsyscall(void)
{
        extern char __vsyscall_page;
        unsigned long physaddr_vsyscall = __pa_symbol(&__vsyscall_page);

        if (vsyscall_mode != NONE)
                __set_fixmap(VSYSCALL_PAGE, physaddr_vsyscall,
                             vsyscall_mode == NATIVE
                             ? PAGE_KERNEL_VSYSCALL
                             : PAGE_KERNEL_VVAR);

        BUILD_BUG_ON((unsigned long)__fix_to_virt(VSYSCALL_PAGE) !=
                     (unsigned long)VSYSCALL_ADDR);
}
```

在 `map_vsyscall` 函数的开头，我们可以看到两个变量的定义。第一个是外部变量 `__vsyscall_page`。作为外部变量，它实际上是在其他源文件中定义的。我们可以在 [arch/x86/entry/vsyscall/vsyscall_emu_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/vsyscall/vsyscall_emu_64.S) 中找到 `__vsyscall_page` 的定义。`__vsyscall_page` 符号指向对齐的 vsyscalls 调用，如 `gettimeofday` 等：

```assembly
	.globl __vsyscall_page
	.balign PAGE_SIZE, 0xcc
	.type __vsyscall_page, @object
__vsyscall_page:

	mov $__NR_gettimeofday, %rax
	syscall
	ret

	.balign 1024, 0xcc
	mov $__NR_time, %rax
	syscall
	ret
    ...
    ...
    ...
```

第二个变量是 `physaddr_vsyscall`，它仅存储 `__vsyscall_page` 符号的物理地址。在接下来的步骤中，我们会检查 `vsyscall_mode` 变量，如果它不等于 `NONE`（默认情况下是 `EMULATE` 模式）：

```C
static enum { EMULATE, NATIVE, NONE } vsyscall_mode = EMULATE;
```

随后我们会看到调用 `__set_fixmap` 函数，该函数会以相同的参数调用 `native_set_fixmap`：

```C
void native_set_fixmap(enum fixed_addresses idx, unsigned long phys, pgprot_t flags)
{
        __native_set_fixmap(idx, pfn_pte(phys >> PAGE_SHIFT, flags));
}

void __native_set_fixmap(enum fixed_addresses idx, pte_t pte)
{
        unsigned long address = __fix_to_virt(idx);

        if (idx >= __end_of_fixed_addresses) {
                BUG();
                return;
        }
        set_pte_vaddr(address, pte);
        fixmaps_set++;
}
```

这里我们可以看到 `native_set_fixmap` 根据给定的物理地址（在我们的例子中是 `__vsyscall_page` 符号的物理地址）生成页表项的值，并调用内部函数 `__native_set_fixmap`。这个内部函数获取给定 `fixed_addresses` 索引（在我们的例子中是 `VSYSCALL_PAGE`）的虚拟地址，并检查给定的索引不超过 fix-mapped 地址的结束范围。之后我们通过调用 `set_pte_vaddr` 函数设置页表项，并增加 fix-mapped 地址的计数。在 `map_vsyscall` 的最后，我们检查 `VSYSCALL_PAGE`（这是 `fixed_addresses` 中的第一个索引）的虚拟地址不超过 `VSYSCALL_ADDR`，即 `-10UL << 20` 或 `ffffffffff600000`，这是通过 `BUILD_BUG_ON` 宏实现的。

```C
BUILD_BUG_ON((unsigned long)__fix_to_virt(VSYSCALL_PAGE) !=
                     (unsigned long)VSYSCALL_ADDR);
```

现在，`vsyscall` 区域已被置于固定映射（fix-mapped）地址区域。关于 `map_vsyscall` 的内容就是这些。如果您对固定映射地址不熟悉，可以参考[《固定映射地址与 ioremap》](https://0xax.gitbook.io/linux-insides/summary/mm/linux-mm-2)一文。我们将在后续关于 `vsyscalls` 和 `vdso` 的章节中更详细地探讨 `vsyscalls` 的实现机制。

获取 SMP 配置
--------------------------------------------------------------------------------

您可能还记得我们在前一部分中是如何搜索 [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing) 配置的。现在，如果找到了 SMP 配置，我们需要获取它。为此，我们检查在 `smp_scan_config` 函数中设置的 `smp_found_config` 变量（关于该函数请参阅前一部分），并调用 `get_smp_config` 函数：

```C
if (smp_found_config)
	get_smp_config();
```

`get_smp_config` 展开为 `x86_init.mpparse.default_get_smp_config` 函数，该函数定义于 [arch/x86/kernel/mpparse.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/mpparse.c)。这个函数定义了指向多处理器浮点指针结构 `mpf_intel` 的指针（您可以在前文[第六部分](https://0xax.gitbook.io/linux-insides/summary/initialization/linux-initialization-6)中阅读相关内容），并执行以下检查：

```C
struct mpf_intel *mpf = mpf_found;

if (!mpf)
    return;

if (acpi_lapic && early)
   return;
```

这里我们可以看到，如果在 `smp_scan_config` 函数中找到了多处理器配置就继续执行，否则直接返回函数。接下来的检查是 `acpi_lapic` 和 `early` 标志。完成这些检查后，我们开始读取 SMP 配置。读取完成后，下一步是调用 `prefill_possible_map` 函数，该函数会预先填充可能的 CPU 的 `cpumask`（更多关于此的内容可参阅 [cpumasks 简介](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-2)）。

setup_arch 的剩余部分
--------------------------------------------------------------------------------

现在我们已经接近 `setup_arch` 函数的尾声。虽然剩余部分也很重要，但本部分不会详细讨论这些内容。我们将简要浏览这些函数，因为它们主要涉及 `NUMA`、`SMP`、`ACPI` 和 `APIC` 等非通用内核特性。首先调用的是 `init_apic_mappings` 函数，它负责设置本地 [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) 的地址。接着是 `x86_io_apic_ops.init` 函数，用于初始化 I/O APIC（关于 APIC 的完整细节将在中断和异常处理章节介绍）。随后通过 `x86_init.resources.reserve_resources` 调用保留标准 I/O 资源（如 `DMA`、`TIMER`、`FPU` 等）。然后是初始化机器检查异常的 `mcheck_init` 函数，最后是注册 [jiffy](http://en.wikipedia.org/wiki/Jiffy_%28time%29) 的 `register_refined_jiffies`（内核定时器将有专门章节讨论）。

至此，我们完成了对 `setup_arch` 这个庞大函数的分析。虽然如我多次提到的，我们尚未涵盖该函数的全部细节，但不必担心。在后续不同章节中，我们还会多次回顾这个函数，以理解各种平台相关部分是如何初始化的。

现在我们可以从 `setup_arch` 返回到 `start_kernel` 函数继续分析了。

回到 main.c
================================================================================

如前所述，我们已经完成了对 `setup_arch` 函数的分析，现在可以回到 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 中的 `start_kernel` 函数。您可能记得或已经注意到，`start_kernel` 函数与 `setup_arch` 一样庞大，因此接下来的几个部分将专门学习这个函数。

在 `setup_arch` 之后，我们可以看到 `mm_init_cpumask` 函数的调用。这个函数将 [cpumask](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-2) 指针设置到内存描述符的 `cpumask` 中。让我们看看它的实现：

```C
static inline void mm_init_cpumask(struct mm_struct *mm)
{
#ifdef CONFIG_CPUMASK_OFFSTACK
        mm->cpu_vm_mask_var = &mm->cpumask_allocation;
#endif
        cpumask_clear(mm->cpu_vm_mask_var);
}
```

如您所见，在 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 中我们将init进程的内存描述符传递给 `mm_init_cpumask` 函数，并根据 `CONFIG_CPUMASK_OFFSTACK` 配置选项来决定是否清除 [TLB](http://en.wikipedia.org/wiki/Translation_lookaside_buffer) 并切换 `cpumask`。

在接下来的步骤中，我们会看到以下函数的调用：

```C
setup_command_line(command_line);
```

该函数接收内核命令行指针，并分配两个缓冲区来存储命令行。我们需要两个缓冲区，因为一个用于将来引用和访问命令行，另一个用于参数解析。我们将为以下缓冲区分配空间：

* `saved_command_line` - 将保存启动命令行；
* `initcall_command_line` - 将保存启动命令行，将在 `do_initcall_level` 中使用；
* `static_command_line` - 将保存用于参数解析的命令行。

我们将使用 `memblock_virt_alloc` 函数分配空间。这个函数调用 `memblock_virt_alloc_try_nid`，如果 [slab](http://en.wikipedia.org/wiki/Slab_allocation) 不可用，则用 `memblock_reserve` 分配启动内存块，否则使用 `kzalloc_node`（更多相关内容将在Linux内存管理章节介绍）。`memblock_virt_alloc` 使用 `BOOTMEM_LOW_LIMIT`（值为 `PAGE_OFFSET + 0x1000000` 的物理地址）和 `BOOTMEM_ALLOC_ACCESSIBLE`（等于当前 `memblock.current_limit` 的值）作为内存区域的最小地址和最大地址。

让我们看看 `setup_command_line` 的实现：

```C
static void __init setup_command_line(char *command_line)
{
        saved_command_line =
                memblock_virt_alloc(strlen(boot_command_line) + 1, 0);
        initcall_command_line =
                memblock_virt_alloc(strlen(boot_command_line) + 1, 0);
        static_command_line = memblock_virt_alloc(strlen(command_line) + 1, 0);
        strcpy(saved_command_line, boot_command_line);
        strcpy(static_command_line, command_line);
 }
```

这里我们为三个缓冲区分配了空间，这些缓冲区将存储用于不同目的的内核命令行（如上所述）。完成空间分配后，我们将 `boot_command_line` 存入 `saved_command_line`，并将来自 `setup_arch` 的 `command_line`（内核命令行）存入 `static_command_line`。

在 `setup_command_line` 之后的下一个函数是 `setup_nr_cpu_ids`。该函数根据 `cpu_possible_mask` 的最后一位来设置 `nr_cpu_ids`（CPU的数量）。关于此概念的更多细节，您可以阅读描述 [cpumasks](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-2) 概念的章节。让我们看看它的实现：

```C
void __init setup_nr_cpu_ids(void)
{
        nr_cpu_ids = find_last_bit(cpumask_bits(cpu_possible_mask),NR_CPUS) + 1;
}
```

这里 `nr_cpu_ids` 表示实际可用的CPU数量，而 `NR_CPUS` 表示在配置时可设置的最大CPU数量：

![CONFIG_NR_CPUS](images/CONFIG_NR_CPUS.png)

实际上我们需要调用这个函数，因为 `NR_CPUS` 可能会大于您计算机中实际的CPU数量。这里我们可以看到调用了 `find_last_bit` 函数并传递了两个参数：

- `cpu_possible_mask` 位图；
- CPU 的最大数量。

在 `setup_arch` 中，我们可以找到 `prefill_possible_map` 函数的调用，它计算实际 CPU 数量并写入 `cpu_possible_mask`。`find_last_bit` 函数接收地址和最大搜索范围作为参数，返回第一个置位(1)的位号。我们传入了 `cpu_possible_mask` 位图和CPU的最大数量。

首先，`find_last_bit` 函数将给定的 `unsigned long` 地址分割成[字](http://en.wikipedia.org/wiki/Word_%28computer_architecture%29)：

```C
words = size / BITS_PER_LONG;
```

其中在 `x86_64` 架构上，`BITS_PER_LONG` 的值为 `64`。当我们获得搜索数据给定大小中的字数后，需要通过以下检查确认给定大小是否包含不完整的字：

```C
if (size & (BITS_PER_LONG-1)) {
         tmp = (addr[words] & (~0UL >> (BITS_PER_LONG
                                 - (size & (BITS_PER_LONG-1)))));
         if (tmp)
                 goto found;
}
```

如果存在不完整的字，我们将对最后一个字进行掩码处理并检查它。如果最后一个字不为零，则表明当前字至少包含一个置位。此时程序将跳转到 `found` 标签处继续执行：

```C
found:
    return words * BITS_PER_LONG + __fls(tmp);
```

这里您可以看到 `__fls` 函数，它通过 `bsr`（Bit Scan Reverse）指令的帮助返回给定字中最后一个置位的位号：

```C
static inline unsigned long __fls(unsigned long word)
{
        asm("bsr %1,%0"
            : "=r" (word)
            : "rm" (word));
        return word;
}
```

`bsr` 指令会扫描给定的操作数以查找第一个置位。如果最后一个字不是部分字，我们将遍历给定地址中的所有字，尝试找到第一个置位：

```C
while (words) {
    tmp = addr[--words];
    if (tmp) {
found:
        return words * BITS_PER_LONG + __fls(tmp);
    }
}
```

这里我们将最后一个字存入 `tmp` 变量，并检查 `tmp` 是否包含至少一个置位。如果找到置位，就返回该位的编号。如果所有字都不包含置位，则直接返回给定的搜索范围大小：

```C
return size;
```

完成这些操作后，`nr_cpu_ids` 将包含正确的可用 CPU 数量。

至此关于架构相关初始化分析完毕。

总结
================================================================================

这是关于 Linux 内核初始化过程的第七部分的结尾。在本次分析中，我们最终完成了对 `setup_arch` 函数的研究，并返回到 `start_kernel` 函数。在下一部分中，我们将继续学习 `start_kernel` 中的通用内核代码，沿着内核启动路径深入，直到第一个 `init` 进程的创建。

如果您有任何疑问或建议，欢迎在评论区留言，或通过 [Twitter](https://twitter.com/0xAX) 与我联系。

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

链接
================================================================================

* [Desktop Management Interface](http://en.wikipedia.org/wiki/Desktop_Management_Interface)
* [x86_64](http://en.wikipedia.org/wiki/X86-64)
* [initrd](http://en.wikipedia.org/wiki/Initrd)
* [Kernel panic](http://en.wikipedia.org/wiki/Kernel_panic)
* [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst)
* [ACPI](http://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)
* [Direct memory access](http://en.wikipedia.org/wiki/Direct_memory_access)
* [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access)
* [Control register](http://en.wikipedia.org/wiki/Control_register)
* [vsyscalls](https://lwn.net/Articles/446528/)
* [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [jiffy](http://en.wikipedia.org/wiki/Jiffy_%28time%29)
* [Previous part](https://0xax.gitbook.io/linux-insides/summary/initialization/linux-initialization-6)