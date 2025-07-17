Kernel initialization. Part 8.
================================================================================

调度器初始化
================================================================================

这是 Linux 内核初始化过程章节的第八[部分](https://0xax.gitbook.io/linux-insides/summary/initialization)，在[上一部分](https://github.com/0xAX/linux-insides/blob/master/Initialization/linux-initialization-7.md)中我们停在了 `setup_nr_cpu_ids` 函数处。

本部分的重点是[调度器](http://en.wikipedia.org/wiki/Scheduling_%28computing%29)的初始化。但在开始学习调度器的初始化过程之前，我们需要完成一些准备工作。接下来是在 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 中的 `setup_per_cpu_areas` 函数，该函数为 `percpu` 变量设置内存区域。更多相关信息您可以在关于 [Per-CPU 变量](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-1) 的专门章节中阅读。在 `percpu` 区域启动并运行后，下一步是 `smp_prepare_boot_cpu` 函数。

`smp_prepare_boot_cpu` 函数为[对称多处理](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)做一些准备工作。由于此函数是针对特定架构的，它位于 [arch/x86/include/asm/smp.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/smp.h#L78) Linux 内核头文件中。让我们看看这个函数的定义：

```C
static inline void smp_prepare_boot_cpu(void)
{
         smp_ops.smp_prepare_boot_cpu();
}
```

可以看到，这里实际上调用了 `smp_ops` 结构体的 `smp_prepare_boot_cpu` 回调函数。如果我们查看 [arch/x86/kernel/smp.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/smp.c) 源文件中该结构体实例的定义，会发现 `smp_prepare_boot_cpu` 展开为对 `native_smp_prepare_boot_cpu` 函数的调用：

```C
struct smp_ops smp_ops = {
    ...
    ...
    ...
    smp_prepare_boot_cpu = native_smp_prepare_boot_cpu,
    ...
    ...
    ...
}
EXPORT_SYMBOL_GPL(smp_ops);
```

函数 `native_smp_prepare_boot_cpu` 形式如下：

```C
void __init native_smp_prepare_boot_cpu(void)
{
        int me = smp_processor_id();
        switch_to_new_gdt(me);
        cpumask_set_cpu(me, cpu_callout_mask);
        per_cpu(cpu_state, me) = CPU_ONLINE;
}
```

该函数主要执行以下操作：首先通过 `smp_processor_id` 函数获取当前 CPU `id`（此时为引导处理器 BSP，其 `id` 为 0）。关于 `smp_processor_id` 的工作原理，此处不再赘述，因为我们已在[内核入口点](https://0xax.gitbook.io/linux-insides/summary/initialization/linux-initialization-4)部分详细分析过。获取处理器 `id` 后，函数通过 `switch_to_new_gdt` 为指定 CPU 重新加载[全局描述符表 GDT](http://en.wikipedia.org/wiki/Global_Descriptor_Table)：

```C
void switch_to_new_gdt(int cpu)
{
        struct desc_ptr gdt_descr;

        gdt_descr.address = (long)get_cpu_gdt_table(cpu);
        gdt_descr.size = GDT_SIZE - 1;
        load_gdt(&gdt_descr);
        load_percpu_segment(cpu);
}
```

这里的 `gdt_descr` 变量表示指向 `GDT` 描述符的指针（`desc_ptr` 结构定义已在[早期中断和异常处理](https://0xax.gitbook.io/linux-insides/summary/initialization/linux-initialization-2)部分见过）。我们通过指定 `id` 获取对应 `CPU` 的 `GDT` 描述符地址和大小。其中 `GDT_SIZE` 的值为 `256`，或者更准确地说：

```C
#define GDT_SIZE (GDT_ENTRIES * 8)
```

而描述符的地址将通过 `get_cpu_gdt_table` 函数获取：

```C
static inline struct desc_struct *get_cpu_gdt_table(unsigned int cpu)
{
        return per_cpu(gdt_page, cpu).gdt;
}
```

`get_cpu_gdt_table` 函数通过 `per_cpu` 宏来获取指定 CPU 编号（本例中为 `ID` 0 的 BSP）对应的 `gdt_page` percpu 变量值。

您可能会问：既然我们能访问 `gdt_page` 这个 percpu 变量，它是在哪里定义的呢？实际上我们在本书前文已经见过它。如果您阅读过本章第一[部分](https://0xax.gitbook.io/linux-insides/summary/initialization/linux-initialization-1)，会记得我们在 [arch/x86/kernel/head_64.S](https://github.com/0xAX/linux/blob/0a07b238e5f488b459b6113a62e06b6aab017f71/arch/x86/kernel/head_64.S) 中看到过 `gdt_page` 的定义：

```assembly
early_gdt_descr:
	.word	GDT_ENTRIES*8-1
early_gdt_descr_base:
	.quad	INIT_PER_CPU_VAR(gdt_page)
```

如果我们查看[链接器](https://github.com/0xAX/linux/blob/0a07b238e5f488b459b6113a62e06b6aab017f71/arch/x86/kernel/vmlinux.lds.S)脚本，可以看到它被放在 `__per_cpu_load` 符号之后：

```C
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(gdt_page);
```

并且在 [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/cpu/common.c#L94) 中完成了对 `gdt_page` 的初始化：

```C
DEFINE_PER_CPU_PAGE_ALIGNED(struct gdt_page, gdt_page) = { .gdt = {
#ifdef CONFIG_X86_64
	[GDT_ENTRY_KERNEL32_CS]		= GDT_ENTRY_INIT(0xc09b, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_CS]		= GDT_ENTRY_INIT(0xa09b, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_DS]		= GDT_ENTRY_INIT(0xc093, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER32_CS]	= GDT_ENTRY_INIT(0xc0fb, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_DS]	= GDT_ENTRY_INIT(0xc0f3, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_CS]	= GDT_ENTRY_INIT(0xa0fb, 0, 0xfffff),
    ...
    ...
    ...
```

关于 `percpu` 变量的更多细节，您可以在 [Per-CPU 变量](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-1)章节中阅读。当我们获取到 `GDT` 描述符的地址和大小后，通过 `load_gdt` 函数重新加载 `GDT`，该函数实际执行 `lgdt` 指令，并通过以下函数加载 `percpu_segment`：

```C
void load_percpu_segment(int cpu) {
    loadsegment(gs, 0);
    wrmsrl(MSR_GS_BASE, (unsigned long)per_cpu(irq_stack_union.gs_base, cpu));
    load_stack_canary_segment();
}
```

`percpu` 区域的基地址必须存放在 `gs` 寄存器（x86 架构下也可能是 `fs` 寄存器）中，因此我们使用 `loadsegment` 宏并传入 `gs`。接下来的步骤会写入 [IRQ](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 栈的基地址，并设置栈 [canary](http://en.wikipedia.org/wiki/Buffer_overflow_protection)（仅针对 `x86_32`）。在加载新的 `GDT` 后，我们会用当前 CPU 填充 `cpu_callout_mask` 位图，并通过设置当前处理器的 `cpu_state` percpu 变量为 `CPU_ONLINE` 来标记 CPU 状态为在线。

```C
cpumask_set_cpu(me, cpu_callout_mask);
per_cpu(cpu_state, me) = CPU_ONLINE;
```

那么，什么是 `cpu_callout_mask` 位图？在初始化引导处理器（x86 架构中第一个启动的处理器）后，多处理器系统中的其他处理器被称为`次级处理器`。Linux 内核使用以下两个位掩码：

* `cpu_callout_mask`
* `cpu_callin_mask`

当 BSP 初始化完成后，它会更新 `cpu_callout_mask` 来指示接下来可以初始化哪个次级处理器。所有其他次级处理器在初始化前会检查引导处理器设置的 `cpu_callout_mask` 位图。只有当引导处理器在 `cpu_callout_mask` 中标记了某个次级处理器后，该次级处理器才会继续其剩余的初始化工作。完成初始化流程后，该次级处理器会在 `cpu_callin_mask` 中设置自己的位。当引导处理器检测到当前次级处理器在 `cpu_callin_mask` 中的位被置位后，就会对剩余的次级处理器重复相同的初始化流程。简而言之，其工作原理如我所描述，但我们将在关于 `SMP`（对称多处理）的章节中看到更多细节。

至此，我们已完成所有的 `SMP` 启动准备工作。

Zonelists 构建
-----------------------------------------------------------------------

接下来，我们会看到 `build_all_zonelists` 函数的调用。该函数设置了内存分配时优先从哪些内存区域（zone）获取的顺序。我们稍后会解释什么是 zone 以及这个顺序的含义。首先让我们看看 Linux 内核如何管理物理内存。物理内存被划分为称为节点（`nodes`）的存储单元。如果您的硬件不支持 `NUMA`（非统一内存访问架构），您将只会看到一个节点：

```
$ cat /sys/devices/system/node/node0/numastat
numa_hit 72452442
numa_miss 0
numa_foreign 0
interleave_hit 12925
local_node 72452442
other_node 0
```

在 Linux 内核中，每个 `node` 都由 `struct pglist_data` 结构体表示，其又被划分为若干特殊的区块，称为内存区域（`zones`）。每个 zone 在内核中由 `zone struct` 表示，并属于以下类型之一（通过 `zone_type` 枚举定义）：

* `ZONE_DMA` - 0-16MB 内存区域；
* `ZONE_DMA32` - 32 位 DMA 设备专用区域（仅能访问 4GB 以下内存）；
* `ZONE_NORMAL` - `x86_64` 架构上 4GB 以上普通内存区域；
* `ZONE_HIGHMEM` - 在 x86_64 架构上不存在（仅用于 32 位系统）；
* `ZONE_MOVABLE` - 包含可移动页面的特殊区域。

可以通过以下方式获取 zone 的详细信息：

```
$ cat /proc/zoneinfo
Node 0, zone      DMA
  pages free     3975
        min      3
        low      3
        ...
        ...
Node 0, zone    DMA32
  pages free     694163
        min      875
        low      1093
        ...
        ...
Node 0, zone   Normal
  pages free     2529995
        min      3146
        low      3932
        ...
        ...
```

如前所述，所有内存节点在内核中均通过 `pglist_data`（即 `pg_data_t`）结构体描述，该结构定义于 [include/linux/mmzone.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/mmzone.h)。[mm/page_alloc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/mm/page_alloc.c) 中的 `build_all_zonelists` 函数会构建一个有序的 `zonelist`（包含 `DMA`、`DMA32`、`NORMAL`、`HIGH_MEMORY`、`MOVABLE` 等不同内存区域），用于指定当所选 `zone` 或 `node` 无法满足分配请求时应尝试访问的备用区域/节点顺序。关于 `NUMA` 和多处理器系统的更多细节，将在后续专题章节深入讨论。

调度器初始化前的剩余步骤
--------------------------------------------------------------------------------

在深入探讨 Linux 内核调度器初始化过程之前，我们需要完成几项准备工作。首先是位于 [mm/page_alloc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/mm/page_alloc.c) 中的 `page_alloc_init` 函数。这个函数看似简单：

```C
void __init page_alloc_init(void)
{
    int ret;

    ret = cpuhp_setup_state_nocalls(CPUHP_PAGE_ALLOC_DEAD,
                                    "mm/page_alloc:dead", NULL,
                                    page_alloc_cpu_dead);
    WARN_ON(ret < 0);
}
```

该函数为 CPU [热插拔](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)状态 `CPUHP_PAGE_ALLOC_DEAD` 设置 `startup` 和 `teardown` 回调函数（第二和第三个参数）。当然，这个函数的实现取决于内核配置选项 `CONFIG_HOTPLUG_CPU` ——若启用该选项，系统将根据各个 CPU 的热插拔状态为其设置相应的回调函数。CPU [热插拔](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)机制是一个复杂主题，本书将不做详细探讨。

完成此函数后，我们能在初始化输出中看到内核命令行信息：

![kernel command line](images/kernel_command_line.png)

接下来会调用几个处理 Linux 内核命令行的函数，包括 `parse_early_param` 和 `parse_args`。您可能记得我们在内核初始化章节的第六[部分](https://0xax.gitbook.io/linux-insides/summary/initialization/linux-initialization-6)已经见过 `parse_early_param` 的调用，那么为什么这里需要再次调用呢？原因很简单：之前是在架构特定代码（如 x86_64）中调用的，但并非所有架构都会调用此函数。而这里调用 `parse_args` 是为了解析和处理非早期的命令行参数。

随后我们可以看到来自 [kernel/jump_label.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/jump_label.c) 的 `jump_label_init` 函数调用，它用于初始化[跳转标签（jump label）](https://lwn.net/Articles/412072/)机制。

接着是 `setup_log_buf` 函数调用，该函数设置 [printk](http://www.makelinux.net/books/lkd2/ch18lev1sec3) 日志缓冲区。我们在 Linux 内核初始化流程的第七[部分](https://0xax.gitbook.io/linux-insides/summary/initialization/linux-initialization-7)已经分析过这个函数。

PID 哈希初始化
--------------------------------------------------------------------------------

接下来是 `pidhash_init` 函数。如您所知，每个进程都会被分配一个唯一的编号，称为进程标识符（`Process Identification Number`，简称 `PID`）。内核会为通过 fork 或 clone 生成的每个新进程自动分配一个唯一的 `PID` 值。`PID` 的管理主要围绕两个特殊数据结构展开：`struct pid` 与 `struct upid`。第一个结构表示内核中 `PID` 信息。第二个结构表示特定命名空间可见的 `PID` 信息。所有 `PID` 实例都存储在专门的哈希表中：

```C
static struct hlist_head *pid_hash;
```

该哈希表用于通过数字 `PID` 值查找对应的 `pid` 实例。因此，`pidhash_init` 函数负责初始化这个哈希表。在 `pidhash_init` 函数的开头，我们可以看到调用了 `alloc_large_system_hash` 函数：

```C
pid_hash = alloc_large_system_hash("PID", sizeof(*pid_hash), 0, 18,
                                   HASH_EARLY | HASH_SMALL,
                                   &pidhash_shift, NULL,
                                   0, 4096);
```

`pid_hash` 哈希表的元素数量取决于 `RAM` 内存配置，其范围介于 $2^4$ 到 $2^{12}$ 之间。`pidhash_init` 函数会计算所需大小并分配存储空间（此处使用的是 `hlist` 结构——与[双向链表](https://0xax.gitbook.io/linux-insides/summary/datastructures/linux-datastructures-1)类似，但在 [struct hlist_head](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/types.h) 中仅包含单个指针）。如果传递 `HASH_EARLY` 标志时（如当前场景），`alloc_large_system_hash` 函数使用 `memblock_virt_alloc_nopanic` 分配大型系统哈希表，否则改用 `__vmalloc` 进行分配。

执行结果可通过 `dmesg` 命令查看：

```
$ dmesg | grep hash
[    0.000000] PID hash table entries: 4096 (order: 3, 32768 bytes)
...
...
...
```

以上就是调度器初始化前的所有准备工作。剩余需要执行的函数包括：

 - `vfs_caches_init_early` 执行[虚拟文件系统（VFS）](http://en.wikipedia.org/wiki/Virtual_file_system)的早期初始化（详细分析将在 VFS 专题章节展开）；
 - `sort_main_extable` 对内核内置的异常表项（位于 `__start___ex_table` 和 `__stop___ex_table` 之间）进行排序；
 - `trap_init` 初始化陷阱处理程序（后两个函数将在中断相关章节详细探讨）。

调度器初始化前最后一步是通过 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 中的 `mm_init` 函数完成内存管理器的初始化。我们可以看到，该函数负责初始化 Linux 内核内存管理器各个部分：

```C
page_ext_init_flatmem();
mem_init();
kmem_cache_init();
percpu_init_late();
pgtable_init();
vmalloc_init();
```

首先是 `page_ext_init_flatmem` 函数，其功能取决于内核配置选项 `CONFIG_SPARSEMEM`，主要用于初始化每页的扩展数据处理。`mem_init` 负责释放所有 `bootmem` 内存，`kmem_cache_init` 初始化内核缓存，`percpu_init_late` 将 `percpu` 内存块替换为 [SLUB](http://en.wikipedia.org/wiki/SLUB_%28software%29) 分配器分配的内存块，`pgtable_init` 初始化 `page->ptl` 内核缓存，而 `vmalloc_init` 则用于初始化 `vmalloc` 机制。**请注意**，我们不会深入探讨这些函数和概念的具体细节，但您可以在 [Linux 内核内存管理](https://0xax.gitbook.io/linux-insides/summary/mm)章节中详细了解它们。

至此，准备工作已全部完成。接下来，我们可以开始探讨 `scheduler`（调度器）的初始化过程。

调度器初始化
--------------------------------------------------------------------------------

现在，我们来到本部分的核心目标——任务调度器的初始化。需要再次说明的是（正如我已多次强调），这里不会完整解释调度器的全部内容，后续会有专门的章节来详细介绍。这里主要描述最先初始化的调度机制。让我们开始吧。

当前我们关注的是位于 [kernel/sched/core.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/sched/core.c) 内核源代码文件中的 `sched_init` 函数，顾名思义，它负责初始化调度器。让我们开始深入这个函数，尝试理解调度器是如何初始化的。在 `sched_init` 函数的开头，我们可以看到以下调用：

```C
sched_clock_init();
```

`sched_clock_init` 是一个相当简单的函数，其主要功能是设置 `sched_clock_init` 变量：

```C
void sched_clock_init(void)
{
	sched_clock_running = 1;
}
```

该变量将在后续流程中使用。接下来的步骤是初始化 `waitqueues` 数组：

```C
for (i = 0; i < WAIT_TABLE_SIZE; i++)
	init_waitqueue_head(bit_wait_table + i);
```

其中 `bit_wait_table` 定义如下：

```C
#define WAIT_TABLE_BITS 8
#define WAIT_TABLE_SIZE (1 << WAIT_TABLE_BITS)
static wait_queue_head_t bit_wait_table[WAIT_TABLE_SIZE] __cacheline_aligned;
```

`bit_wait_table` 是一个等待队列数组，用于根据特定位的值来等待/唤醒进程。初始化 `waitqueues` 数组后，下一步是计算为 `root_task_group` 分配的内存大小。可以看到，这个大小取决于以下两个内核配置选项：

```C
#ifdef CONFIG_FAIR_GROUP_SCHED
         alloc_size += 2 * nr_cpu_ids * sizeof(void **);
#endif
#ifdef CONFIG_RT_GROUP_SCHED
         alloc_size += 2 * nr_cpu_ids * sizeof(void **);
#endif
```

* `CONFIG_FAIR_GROUP_SCHED`；
* `CONFIG_RT_GROUP_SCHED`。

这两个选项提供了两种不同的调度模型。根据[文档](https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt)说明，当前使用的调度器——完全公平调度器（`CFS`）采用了一个简洁的设计理念。它将进程调度建模为理想的多任务处理器，其中每个可运行进程都能获得 $\displaystyle\frac 1 n$ 的处理器时间（`n` 代表可运行进程数量）。该调度器遵循特定的规则集，这些规则决定了何时以及如何选择新进程运行，被称为`调度策略`。

`完全公平调度器`支持以下`普通`（即`非实时`）调度策略：

* `SCHED_NORMAL`;
* `SCHED_BATCH`;
* `SCHED_IDLE`.

`SCHED_NORMAL` 用于普通应用程序，每个进程占用的 CPU 时间主要由 [nice](http://en.wikipedia.org/wiki/Nice_%28Unix%29) 值决定；`SCHED_BATCH` 用于 100% 非交互式任务；而 `SCHED_IDLE` 仅在处理器没有其他任务可运行时才会调度。

对于时间敏感的实时应用，内核也支持两种策略：`SCHED_FIFO` 和 `SCHED_RR`。了解 Linux 内核调度器的读者会知道其采用模块化设计，这意味着它支持不同算法来调度不同类型的进程。这种模块化通常称为`调度类`（scheduler classes），这些模块封装了调度策略细节，由调度器核心统一处理而无需了解具体实现。

现在回到代码，我们关注两个配置选项：`CONFIG_FAIR_GROUP_SCHED` 和 `CONFIG_RT_GROUP_SCHED`。虽然调度器调度的基本单位是单个任务或线程，但进程并非唯一可调度实体。这两个选项均提供了对组调度的支持——前者针对`完全公平调度器`策略，后者则对应`实时`策略。

简单来说，组调度是一种功能，允许我们将一组任务视为单个任务进行调度。例如，如果你创建一个包含两个任务的组，从内核的角度来看，这个组就像一个普通任务。当组被调度时，调度器会从该组中选择一个任务在组内进行调度。这种机制让我们能够构建层级结构并管理它们的资源。虽然调度的最小单位是进程，但 Linux 内核调度器在底层并不直接使用 `task_struct` 结构，而是通过专门的 `sched_entity` 结构作为调度单元。

因此，当前的目标是计算为根任务组的 `sched_entity` 分配的空间大小，我们通过以下两种情况进行计算：

```C
#ifdef CONFIG_FAIR_GROUP_SCHED
         alloc_size += 2 * nr_cpu_ids * sizeof(void **);
#endif
#ifdef CONFIG_RT_GROUP_SCHED
         alloc_size += 2 * nr_cpu_ids * sizeof(void **);
#endif
```

第一种情况是启用了`完全公平`调度器的任务组调度功能，第二种情况则是针对`实时`调度器的相同用途。这里我们计算的大小等于指针大小乘以系统中的 CPU 数量，再乘以 `2`。需要乘以 `2` 是因为我们要为以下两项分配空间：

* 调度实体结构；
* `runqueue`。

计算完大小后，我们通过 `kzalloc` 函数分配空间，并在此设置 `sched_entity` 和 `runqueues` 的指针：

```C
ptr = (unsigned long)kzalloc(alloc_size, GFP_NOWAIT);

#ifdef CONFIG_FAIR_GROUP_SCHED
        root_task_group.se = (struct sched_entity **)ptr;
        ptr += nr_cpu_ids * sizeof(void **);

        root_task_group.cfs_rq = (struct cfs_rq **)ptr;
        ptr += nr_cpu_ids * sizeof(void **);
#endif
#ifdef CONFIG_RT_GROUP_SCHED
		root_task_group.rt_se = (struct sched_rt_entity **)ptr;
		ptr += nr_cpu_ids * sizeof(void **);

		root_task_group.rt_rq = (struct rt_rq **)ptr;
		ptr += nr_cpu_ids * sizeof(void **);

#endif
```

如前所述，Linux 的组调度机制支持层级结构定义。这类层级结构的根节点是 `root_runqueuetask_group` 任务组结构体。该结构体包含多个字段，但目前我们重点关注 `se`、`rt_se`、`cfs_rq` 和 `rt_rq` 这四个成员：

其中 `se` 和 `rt_se` 是 `sched_entity` 结构体的实例，该结构定义于 [include/linux/sched.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/sched.h) 头文件，是调度器的基本调度单元。

```C
struct task_group {
    ...
    ...
    struct sched_entity **se;
    struct cfs_rq **cfs_rq;
    ...
    ...
}
```

`cfs_rq` 和 `rt_rq` 表示运行队列（`run queues`）。运行队列是一种特殊的 `per-cpu` 结构，Linux 内核调度器用它来存储 `active` 线程，换句话说，这些线程集合中的线程可能会被调度器选中运行。

空间分配完成后，下一步是初始化 `real-time`（实时）和 `deadline`（截止时间）任务的 CPU 带宽：

```C
init_rt_bandwidth(&def_rt_bandwidth,
                  global_rt_period(), global_rt_runtime());
init_dl_bandwidth(&def_dl_bandwidth,
                  global_rt_period(), global_rt_runtime());
```

所有任务组都必须能够依赖预设的 CPU 时间配额。以下两个结构体：`def_rt_bandwidth` 和 `def_dl_bandwidth`，分别表示实时任务和截止时间任务的默认带宽配置。虽然这些结构体的具体定义目前并不重要，但我们重点关注以下两个参数值：

* `sched_rt_period_us`;
* `sched_rt_runtime_us`.

第一个参数表示周期长度，第二个参数表示在 `sched_rt_period_us` 周期内为 `real-time` 任务分配的时间量（quantum）。这些参数的全局默认值可以在以下位置查看：

```
$ cat /proc/sys/kernel/sched_rt_period_us
1000000

$ cat /proc/sys/kernel/sched_rt_runtime_us
950000
```

与任务组相关的参数可通过 `<cgroup>/cpu.rt_period_us` 和 `<cgroup>/cpu.rt_runtime_us` 进行配置。由于此时尚未挂载任何文件系统，`def_rt_bandwidth` 和 `def_dl_bandwidth` 将使用 `global_rt_period` 与 `global_rt_runtime` 函数返回的默认值进行初始化。

至此已完成 `real-time` 和 `deadline` 任务的带宽设置。接下来，根据是否启用 [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)，我们将初始化 `root domain`：

```C
#ifdef CONFIG_SMP
	init_defrootdomain();
#endif
```

实时调度器需要全局资源来做出调度决策。但随着 CPU 数量增加，可扩展性瓶颈就会出现。引入根域（`root domains`）概念正是为了提升可扩展性并避免此类瓶颈。调度器不再需要遍历所有运行队列，而是通过 `root_domain` 结构获取可以推送/拉取实时任务的 CPU 信息。该结构定义于 [kernel/sched/sched.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/sched/sched.h) 头文件，主要跟踪可用于进程迁移的 CPU 集合。

完成根域初始化后，我们像之前一样为根任务组的实时任务初始化带宽控制：

```C
#ifdef CONFIG_RT_GROUP_SCHED
	init_rt_bandwidth(&root_task_group.rt_bandwidth,
			global_rt_period(), global_rt_runtime());
#endif
```

并使用相同的默认值进行初始化。

接下来，根据内核配置选项 `CONFIG_CGROUP_SCHED` 的设置，我们会为 `task_group` 结构分配 `slab` 缓存，并初始化根任务组的 `siblings` 和 `children` 链表。根据内核文档说明，`CONFIG_CGROUP_SCHED` 选项的作用是：

> 允许通过 cgroup 伪文件系统创建任意任务组，并控制分配给每个任务组的 CPU 带宽。

完成链表初始化后，我们可以看到调用了 `autogroup_init` 函数：

```C
#ifdef CONFIG_CGROUP_SCHED
         list_add(&root_task_group.list, &task_groups);
         INIT_LIST_HEAD(&root_task_group.children);
         INIT_LIST_HEAD(&root_task_group.siblings);
         autogroup_init(&init_task);
#endif
```

该函数用于初始化自动进程组调度机制。`autogroup` 特性的作用是通过 [setsid](https://linux.die.net/man/2/setsid) 系统调用创建新会话时，自动创建并填充新的任务组。

完成此操作后，我们会遍历所有 `possible` CPU（您可能记得 `possible` CPU 存储在 `cpu_possible_mask` 位图中，表示系统中可能存在的所有 CPU），并为每个 `possible` CPU 初始化运行队列：

```C
for_each_possible_cpu(i) {
    struct rq *rq;
    ...
    ...
    ...
```

Linux 内核中的 `rq` 结构体定义于 [kernel/sched/sched.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/sched/sched.h#L625)。如之前所述，运行队列是调度过程中的核心数据结构，调度器通过它决定下一个要运行的任务。该结构体包含众多字段，我们不会在此逐一说明，而是在后续实际使用时具体分析。

在为每个 CPU 的运行队列完成默认值初始化后，我们需要设置系统中第一个任务的`负载权重`（`load weight`）：

```C
set_load_weight(&init_task);
```

首先我们来理解什么是进程的`负载权重`。如果查看 `sched_entity` 结构的定义，会发现它起始于 `load` 字段：

```C
struct sched_entity {
	struct load_weight		load;
    ...
    ...
    ...
}
```

`load weight` 结构仅包含两个字段，分别表示调度实体的实际权重及其优化计算的固定倒数：

```C
struct load_weight {
	unsigned long	weight;
	u32				inv_weight;
};
```

您可能知道系统中每个进程都有`优先级`（`priority`）。更高的优先级意味着可以获得更多的运行时间。进程的`负载权重`实际上反映了该进程优先级与其时间片配额之间的关系。每个进程都包含以下三个与优先级相关的字段：

```C
struct task_struct {
...
...
...
	int				prio;
	int				static_prio;
	int				normal_prio;
...
...
...
}
```

第一个是`动态优先级`（`dynamic priority`），它基于进程的静态优先级和交互性，在进程生命周期内不可更改。`static_prio` 包含初始优先级，即您可能熟知的 `nice` 值。除非用户主动修改，否则内核不会改变这个值。最后一个 `normal_priority` 虽然也基于 `static_prio` 的值，但同时还取决于进程的调度策略。

`set_load_weight` 函数的主要目标是为 `init` 任务初始化 `load_weight` 字段：

```C
static void set_load_weight(struct task_struct *p)
{
	int prio = p->static_prio - MAX_RT_PRIO;
	struct load_weight *load = &p->se.load;

	if (idle_policy(p->policy)) {
		load->weight = scale_load(WEIGHT_IDLEPRIO);
		load->inv_weight = WMULT_IDLEPRIO;
		return;
	}

	load->weight = scale_load(sched_prio_to_weight[prio]);
	load->inv_weight = sched_prio_to_wmult[prio];
}
```

可以看到，我们根据 `init` 任务的初始 `static_prio` 值计算出初始 `prio`，并将其作为 `sched_prio_to_weight` 和 `sched_prio_to_wmult` 数组的索引来设置 `weight` 和 `inv_weight` 值。这两个数组包含了基于优先级值的`负载权重`。对于 `idle` 进程，我们设置最小负载权重。

至此，我们已完成 Linux 内核调度器的初始化过程。最后的步骤包括：将当前进程（即首个 `init` 进程）设为 `idle` 状态（当 CPU 没有其他进程可运行时执行），计算下一次 CPU 负载计算的时间周期，以及初始化 `fair` 调度类：

```C
__init void init_sched_fair_class(void)
{
#ifdef CONFIG_SMP
	open_softirq(SCHED_SOFTIRQ, run_rebalance_domains);
#endif
}
```

这里我们注册了一个[软中断](https://0xax.gitbook.io/linux-insides/summary/interrupts/linux-interrupts-9)，该中断将调用 `run_rebalance_domains` 处理程序。当 `SCHED_SOFTIRQ` 被触发时，`run_rebalance` 函数会被调用来重新平衡当前 CPU 的运行队列。

`sched_init` 函数的最后两个步骤是初始化调度器统计信息和设置 `scheduler_running` 变量：

```C
scheduler_running = 1;
```

至此，Linux 内核调度器已完成初始化。当然，我们在此跳过了许多细节和解释，因为需要先理解 Linux 内核中各种概念（如进程和进程组、运行队列、RCU 等）的工作原理。不过我们已经对调度器初始化过程有了基本了解，更详细的内容将在专门讨论调度器的独立章节中深入分析。

总结
--------------------------------------------------------------------------------

这是关于 Linux 内核初始化过程的第八部分的结尾。在本部分中，我们探讨了调度器的初始化流程，并将在下一部分继续深入 Linux 内核初始化过程，重点分析 [RCU](http://en.wikipedia.org/wiki/Read-copy-update) 机制及其他核心组件的初始化工作。

如果您有任何疑问或建议，请通过 [Twitter](https://twitter.com/0xAX) 或者 [email](https://github.com/hust-open-atom-club/linux-insides-zh/issues/anotherworldofworld@gmail.com) 与我联系，或者创建一个 [issue](https://github.com/hust-open-atom-club/linux-insides-zh/issues/new)。

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

链接
--------------------------------------------------------------------------------

* [CPU masks](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-2)
* [high-resolution kernel timer](https://www.kernel.org/doc/Documentation/timers/hrtimers.txt)
* [spinlock](http://en.wikipedia.org/wiki/Spinlock)
* [Run queue](http://en.wikipedia.org/wiki/Run_queue)
* [Linux kernel memory manager](https://0xax.gitbook.io/linux-insides/summary/mm)
* [slub](http://en.wikipedia.org/wiki/SLUB_%28software%29)
* [virtual file system](http://en.wikipedia.org/wiki/Virtual_file_system)
* [Linux kernel hotplug documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)
* [IRQ](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [Global Descriptor Table](http://en.wikipedia.org/wiki/Global_Descriptor_Table)
* [Per-CPU variables](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-1)
* [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [RCU](http://en.wikipedia.org/wiki/Read-copy-update)
* [CFS Scheduler documentation](https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt)
* [Real-Time group scheduling](https://www.kernel.org/doc/Documentation/scheduler/sched-rt-group.txt)
* [Previous part](https://0xax.gitbook.io/linux-insides/summary/initialization/linux-initialization-7)
