内核初始化 第六部分
===========================================================

仍旧是与系统架构有关的初始化
===========================================================  

在之前的[章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-5.html)我们从 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c)了解了特定于系统架构的初始化事务(在我们的例子中是 `x86_64` 架构)，并且通过 `x86_configure_nx` 函数根据对[NX bit](http://en.wikipedia.org/wiki/NX_bit)的支持配置了 `_PAGE_NX` 标志位。正如我之前写的, `setup_arch` 函数和 `start_kernel` 都非常复杂，所以在这个和下个章节我们将继续学习关于系统架构初始化进程的内容。`x86_configure_nx` 函数的下面是 `parse_early_param` 函数。这个函数定义在 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 中并且你可以从它的名字中了解到，这个函数解析内核命令行并且基于给定的参数创建不同的服务 (所有的内核命令行参数你都可以在 [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt) 找到)。 你可能记得在最前面的 [章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-2.html) 我们是怎样创建 `earlyprintk`地。在前面我们用 [arch/x86/boot/cmdline.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cmdline.c) 里面的 `cmdline_find_option` 和 `__cmdline_find_option`, `__cmdline_find_option_bool` 函数的帮助下寻找内核参数及其值。我们在通用内核部分不依赖于特定的系统架构，在这里我们使用另一种方法。 如果你正在阅读linux内核源代码，你可能注意到这样的调用：

```C
early_param("gbpages", parse_direct_gbpages_on);
```

`early_param` 宏需要两个参数:

* 命令行参数的名称
* 如果给定的参数通过，函数将被调用 

函数定义如下: 

```C
#define early_param(str, fn) \
        __setup_param(str, fn, fn, 1)
```

这个定义可以在 [include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h) 中可以找到.  
正如你所看到的， `early_param` 宏只是调用了 `__setup_param` 宏:

```C
#define __setup_param(str, unique_id, fn, early)                \
        static const char __setup_str_##unique_id[] __initconst \
                __aligned(1) = str; \
        static struct obs_kernel_param __setup_##unique_id      \
                __used __section(.init.setup)                   \
                __attribute__((aligned((sizeof(long)))))        \
                = { __setup_str_##unique_id, fn, early }
```

这个宏内部定义了 `__setup_str_*_id` 变量 (这里的 `*` 取决于给定的函数名称)，然后把给定的命令行参数赋值给这个变量。在下一行中，我们可以看到定义了一个`obs_kernel_param` 类型的变量 `__setup_ *` 并对其进行初始化。   

`obs_kernel_param` 结构体定义如下:

```C
struct obs_kernel_param {
        const char *str;
        int (*setup_func)(char *);
        int early;
};
```

这个结构体包含三个字段:   

* 内核参数的名称  
* 根据不同的参数，选取对应的处理函数 
* 决定参数是否为 early 的标记位 

注意 `__set_param` 宏定义有 `__section(.init.setup)` 属性。这意味着所有 `__setup_str_ *` 都将被放置在 `.init.setup` 区段中，此外正如我们在 [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/vmlinux.lds.h) 中看到的，`.init.setup` 区段被放置在 `__setup_start` 和 `__setup_end` 之间:   

```
#define INIT_SETUP(initsetup_align)                \
                . = ALIGN(initsetup_align);        \
                VMLINUX_SYMBOL(__setup_start) = .; \
                *(.init.setup)                     \
                VMLINUX_SYMBOL(__setup_end) = .;
```

现在我们知道了参数是怎样定义的，让我们一起回到 `parse_early_param` 的实现上来:   

```C
void __init parse_early_param(void)
{
        static int done __initdata;
        static char tmp_cmdline[COMMAND_LINE_SIZE] __initdata;

        if (done)
                return;

        /* All fall through to do_early_param. */
        strlcpy(tmp_cmdline, boot_command_line, COMMAND_LINE_SIZE);
        parse_early_options(tmp_cmdline);
        done = 1;
}
```
`parse_early_param` 函数内部定义了两个静态变量。首先第一个变量 `done` 用来检查 `parse_early_param` 函数是否已经被调用过，第二个变量是用来临时存储内核命令行的。然后我们把 `boot_command_line` 的值赋值给刚刚定义的临时命令行变量中( `tmp_cmdline` ) 并且从相同的源代码文件 `main.c` 中调用 `parse_early_options` 函数。 `parse_early_options`函数从 [kernel/params.c](https://github.com/torvalds/linux/blob/master/) 中调用 `parse_args` 函数, `parse_args` 解析传入的命令行然后调用 `do_early_param` 函数。 [do_early_param](https://github.com/torvalds/linux/blob/master/init/main.c#L413) 函数 从 ` __setup_start` 循环到 `__setup_end` ，如果循环中 `obs_kernel_param` 实例中的 `early` 字段值为1 ,就调用 `obs_kernel_param` 中的第二个函数 `setup_func`。在这之后所有基于早期命令行参数的服务都已经被创建，在 `parse_early_param` 之后的下一个函数调用是 `x86_report_nx` 。 正如我在这章开头所写的，我们已经用 `x86_configure_nx` 函数配置了 `NX-bit` 位。接下来我们使用 [arch/x86/mm/setup_nx.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/setup_nx.c) 中的 `x86_report_nx`函数打印出关于 `NX` 的信息。注意`x86_report_nx` 函数不一定在 `x86_configure_nx` 函数之后调用，但是一定在 `parse_early_param` 之后调用。答案很简单: 因为内核支持 `noexec` 参数，所以我们一定在 `parse_early_param` 调用并且解析  `noexec` 参数之后才能调用 `x86_report_nx` :

```
noexec		[X86]
			On X86-32 available only on PAE configured kernels.
			//在X86-32架构上，仅在配置PAE的内核上可用。
			noexec=on: enable non-executable mappings (default)
			//noexec=on:开启非可执行文件的映射(默认)
			noexec=off: disable non-executable mappings
			//noexec=off: 禁用非可执行文件的映射
```

我们可以在启动的时候看到:

![NX](http://oi62.tinypic.com/swwxhy.jpg)

之后我们可以看到下面函数的调用:   

```C
	memblock_x86_reserve_range_setup_data();
```

这个函数的定义也在 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c) 中，然后这个函数为 `setup_data` 重新映射内存并保留内存块(你可以阅读之前的 [章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-5.html) 了解关于 `setup_data` 的更多内容，你也可以在 [Linux kernel memory management](http://xinqiu.gitbooks.io/linux-insides-cn/content/MM/index.html) 中阅读到关于 `ioremap` and `memblock` 的更多内容)。 

接下来我们来看看下面的条件语句:    

```C
	if (acpi_mps_check()) {
#ifdef CONFIG_X86_LOCAL_APIC
		disable_apic = 1;
#endif
		setup_clear_cpu_cap(X86_FEATURE_APIC);
	}
```

`acpi_mps_check` 函数来自于 [arch/x86/kernel/acpi/boot.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/acpi/boot.c) ，它的结果取决于 `CONFIG_X86_LOCAL_APIC` 和 `CONFIG_x86_MPPARSE` 配置选项:   

```C
int __init acpi_mps_check(void)
{
#if defined(CONFIG_X86_LOCAL_APIC) && !defined(CONFIG_X86_MPPARSE)
        /* mptable code is not built-in*/
        if (acpi_disabled || acpi_noirq) {
                printk(KERN_WARNING "MPS support code is not built-in.\n"
                       "Using acpi=off or acpi=noirq or pci=noacpi "
                       "may have problem\n");
                 return 1;
        }
#endif
        return 0;
}
```

`acpi_mps_check` 函数检查内置的 `MPS` 又称 [多重处理器规范]((http://en.wikipedia.org/wiki/MultiProcessor_Specification)) 表。如果设置了 ` CONFIG_X86_LOCAL_APIC` 但未设置 `CONFIG_x86_MPPAARSE` ，而且传递给内核的命令行选项中有 `acpi=off`、`acpi=noirq` 或者 `pci=noacpi` 参数，那么`acpi_mps_check` 函数就会输出警告信息。如果 `acpi_mps_check` 返回了1，这就表示我们禁用了本地 [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) 
,而且 `setup_clear_cpu_cap` 宏清除了当前CPU中的 `X86_FEATURE_APIC` 位。（你可以阅读 [CPU masks](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-2.html) 了解关于CPU mask的更多内容)。

早期的PCI转储
--------------------------------------------------------------------------------  

接下来我们通过下面的代码来转储 [PCI](http://en.wikipedia.org/wiki/Conventional_PCI) 设备: 

```C
#ifdef CONFIG_PCI
	if (pci_early_dump_regs)
		early_dump_pci_devices();
#endif
```

变量 `pci_early_dump_regs` 定义在 [arch/x86/pci/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/pci/common.c) 中，他的值取决于内核命令行参数：`pci=earlydump` 。我们可以在[drivers/pci/pci.c](https://github.com/torvalds/linux/blob/master/arch) 中看到这个参数的定义:   

```C
early_param("pci", pci_setup);
```

`pci_setup` 函数取出 `pci=` 之后的字符串，然后进行解析。这个函数调用 [drivers/pci/pci.c](https://github.com/torvalds/linux/blob/master/arch) 中用 `_weak` 修饰符定义的 `pcibios_setup` 函数，并且每种架构都重写了 `_weak` 修饰过的函数。 例如， `x86_64` 架构上的该函数版本在 [arch/x86/pci/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/pci/common.c) 中:  

```C
char *__init pcibios_setup(char *str) {
        ...
		...
		...
		} else if (!strcmp(str, "earlydump")) {
                pci_early_dump_regs = 1;
                return NULL;
        }
		...
		...
		...
}
```

如果我们设置了 `CONFIG_PCI` 选项，而且向内核命令行传递了 `pci=earlydump` 选项，那么 [arch/x86/pci/early.c](https://github.com/torvalds/linux/blob/master/arch/x86/pci/early.c) 中的 `early_dump_pci_devices` 函数将会被调用。这个函数像下面这样来检查pci参数 `noearly` :   

```C
if (!early_pci_allowed())
        return;
```

如果条件不成立则返回。每个PCI域可以承载多达 `256` 条总线，并且每条总线可以承载多达32个设备。那么接下来我们进入下面的循环:   

```C
for (bus = 0; bus < 256; bus++) {
                for (slot = 0; slot < 32; slot++) {
                        for (func = 0; func < 8; func++) {
						...
						...
						...
                        }
                }
}
```

然后我们通过 `read_pci_config` 函数来读取 `pci` 配置。 

这就是 pci 加载的全部过程了。我们在这里不会深入研究 `pci` 的细节，不过我们会在 `Drivers/PCI` 章节看到更多的细节。 

内存解析的完成
--------------------------------------------------------------------------------   


在 `early_dump_pci_devices` 函数后面，有一些与可用内存和[e820](http://en.wikipedia.org/wiki/E820)相关的函数，其中 [e820](http://en.wikipedia.org/wiki/E820) 的相关信息我们在 [内核安装的第一步](http://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-2.html) 章节中整理过。   
```C
	/* update the e820_saved too */
	e820_reserve_setup_data();
	finish_e820_parsing();
	...
	...
	...
	e820_add_kernel_range();
	trim_bios_range(void);
	max_pfn = e820_end_of_ram_pfn();
	early_reserve_e820_mpc_new();
```

让我们来一起看看上面的代码。正如你所看到的，第一个函数是 `e820_reserve_setup_data` 。这个函数和我们前面看到的 `memblock_x86_reserve_range_setup_data` 函数做的事情几乎是相同的，但是这个函数同时还会调用 `e820_update_range` 函数，向 `e820map` 中用给定的类型添加新的区域，在我们的例子中，使用的是 `E820_RESERVED_KERN` 类型。接下来的函数是 `finish_e820_parsing`,这个函数使用 `sanitize_e820_map` 函数对 `e820map` 进行清理。除了这两个函数之外，我们还可以看到一些与 [e820](http://en.wikipedia.org/wiki/E820) 有关的函数。你可以在上面的列表中看到这些函数。`e820_add_kernel_range` 函数需要内核开始和结束的物理地址:   

```C
u64 start = __pa_symbol(_text);
u64 size = __pa_symbol(_end) - start;
``` 

函数会检查在 `e820map` 中被标记成 `E820RAM` 的 `.text` `.data` 和 `.bss` 区段，如果没有这些区段，那么就会输出错误信息。接下来的 `trm_bios_range` 函数把 `e820Map` 中的前4096个字节修改为 `E820_RESERVED` 并且再次调用函数 `sanitize_e820_map` 清理 `e820map`。在这之后我们使用 `e820_end_of_ram_pfn` 函数得到最后一个页帧的编号，每个内存页面都有一个唯一的编号 - `页帧号`， `e820_end_of_ram_pfn` 函数调用 `e820_end_pfn` 函数返回最大的页面帧号:   

```C
unsigned long __init e820_end_of_ram_pfn(void)
{
	return e820_end_pfn(MAX_ARCH_PFN);
}
```

`e820_end_pfn` 函数读取特定于系统架构的最大页帧号(对于 `x86_64` 架构来说 `MAX_ARCH_PFN` 是 `0x400000000` )。在 `e820_end_pfn` 函数中我们遍历整个 `e820` 槽，并且检查 `e820` 中是否有 `E820_RAM` 或者 `E820_PRAM` 类型条目，因为我们只能对这些类型计算页面帧号，然后我们得到当前 `e820` 页面帧的基地址和结束地址，同时对这些地址进行检查:   

```C
for (i = 0; i < e820.nr_map; i++) {
		struct e820entry *ei = &e820.map[i];
		unsigned long start_pfn;
		unsigned long end_pfn;

		if (ei->type != E820_RAM && ei->type != E820_PRAM)
			continue;

		start_pfn = ei->addr >> PAGE_SHIFT;
		end_pfn = (ei->addr + ei->size) >> PAGE_SHIFT;

        if (start_pfn >= limit_pfn)
			continue;
		if (end_pfn > limit_pfn) {
			last_pfn = limit_pfn;
			break;
		}
		if (end_pfn > last_pfn)
			last_pfn = end_pfn;
}
```

```C
	if (last_pfn > max_arch_pfn)
		last_pfn = max_arch_pfn;

	printk(KERN_INFO "e820: last_pfn = %#lx max_arch_pfn = %#lx\n",
			 last_pfn, max_arch_pfn);
	return last_pfn;
```

接下来我们检查在循环中得到的 `last_pfn`，`last_pfn` 不得大于特定于系统架构的最大页帧号(在我们的例子中是 `x86_64` 系统架构),然后输出关于最大页帧号的信息，并且返回 `last_pfn`。我们可以在 `dmesg` 的输出中看到 `last_pfn` :

```
...
[    0.000000] e820: last_pfn = 0x41f000 max_arch_pfn = 0x400000000
...
```

在这之后，我们计算出了最大的页帧号，我们要计算 `max_low_pfn` ,这是 `低端内存` 或者低于第一个4GB中的最大页面帧。如果系统安装了超过4GB的内存RAM，`max_low_pfn` 将会是`e820_end_of_low_ram_pfn` 函数的结果，这个函数和 `e820_end_of_ram_pfn` 相似，但是有4GB限制，换句话说 `max_low_pfn` 和 `max_pfn` 的值是一样的:   

```C
if (max_pfn > (1UL<<(32 - PAGE_SHIFT)))
	max_low_pfn = e820_end_of_low_ram_pfn();
else
	max_low_pfn = max_pfn;
		
high_memory = (void *)__va(max_pfn * PAGE_SIZE - 1) + 1;
```

接下来我们通过 `__va` 宏计算 `高端内存` (有更高的内存直接映射上界)中的最大页帧号,并且这个宏会根据给定的物理内存返回一个虚拟地址。 


桌面管理接口   
-------------------------------------------------------------------------------

在处理完不同内存区域和 `e820` 槽之后，接下来就该收集计算机的相关信息了。我们将用下面的函数收集与 [桌面管理接口](http://en.wikipedia.org/wiki/Desktop_Management_Interface) 有关的所有信息:

```C
dmi_scan_machine();
dmi_memdev_walk();
```

首先是定义在 [drivers/firmware/dmi_scan.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/firmware/dmi_scan.c) 中的 `dmi_scan_machine` 函数。这个函数遍历 [System Management BIOS](http://en.wikipedia.org/wiki/System_Management_BIOS) 结构，并从中提取信息。这里有两种方法来访问 `SMBIOS` 表: 第一种是从 [EFI](http://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) 的配置表获得指向 `SMBIOS` 表的指针；第二种是扫描 `0xF0000` 和 `0x10000` 地址之间的物理内存。让我们一起看看第二种方法。`dmi_scan_machine` 函数通过 `dmi_early_remap` 函数将 `0xf0000` 和 `0x10000` 之间的内存重新映射并追加到 `early_ioremap` 上:

```C
void __init dmi_scan_machine(void)
{
	char __iomem *p, *q;
	char buf[32];
	...
	...
	...
	p = dmi_early_remap(0xF0000, 0x10000);
	if (p == NULL)
			goto error;
```
然后迭代所有的 `DMI` 头部地址，并且查找 `_SM_` 字符串:   

```C
memset(buf, 0, 16);
for (q = p; q < p + 0x10000; q += 16) {
		memcpy_fromio(buf + 16, q, 16);
		if (!dmi_smbios3_present(buf) || !dmi_present(buf)) {
			dmi_available = 1;
			dmi_early_unmap(p, 0x10000);
			goto out;
		}
		memcpy(buf, buf + 16, 16);
}
```   
`_SM_` 字符串一定在 `000F0000h` 和 `0x000FFFFF` 地址之间。在这里我们用 `memcpy_fromio` 函数向 `buf` 里面拷贝16个字节，这个函数和 `memcpy` 函数的作用是一样的。然后对这个缓冲区( `buf` ) 执行`dmi_smbios3_present` 和 `dmi_present` 函数。这些函数检查 `buf` 的前4个字节是否是 `__SM__` 字符串，并且获得 `SMBIOS` 的版本和 `_DMI_` 的属性例如 `_DMI_` 的结构表长度、结构表的地址等等... 在其中的一个函数完成之后，你就可以在 `dmesg` 的输出中看到它的运行结果:   

```
[    0.000000] SMBIOS 2.7 present.
[    0.000000] DMI: Gigabyte Technology Co., Ltd. Z97X-UD5H-BK/Z97X-UD5H-BK, BIOS F6 06/17/2014
```

在 `dmi_scan_machine` 函数的最后，我们取消之前映射的内存:   

```C
dmi_early_unmap(p, 0x10000);
```

第二个函数是 - `dmi_memdev_walk`。和你想的一样，这个函数遍历整个内存设备。让我们一起看看这个函数:   

```C
void __init dmi_memdev_walk(void)
{
	if (!dmi_available)
		return;

	if (dmi_walk_early(count_mem_devices) == 0 && dmi_memdev_nr) {
		dmi_memdev = dmi_alloc(sizeof(*dmi_memdev) * dmi_memdev_nr);
		if (dmi_memdev)
			dmi_walk_early(save_mem_devices);
	}
}
```

这个函数检查 `DMI` 是否可用(我们之前在 `dmi_scan_machine` 函数中得到了这个结果，并且保存在 `dmi_available` 变量中)，然后使用 `dmi_walk_early` 和 `dmi_alloc` 函数收集内存设备的有关信息,其中  `dmi_alloc` 的定义如下:   

```
#ifdef CONFIG_DMI
RESERVE_BRK(dmi_alloc, 65536);
#endif
```

定义在 [arch/x86/include/asm/setup.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/setup.h) 中的 `RESERVE_BRK` 函数会在 `brk` 段中预留给定大小的空间:   

-------------------------
	init_hypervisor_platform();
	x86_init.resources.probe_roms();
	insert_resource(&iomem_resource, &code_resource);
	insert_resource(&iomem_resource, &data_resource);
	insert_resource(&iomem_resource, &bss_resource);
	early_gart_iommu_check();    


均衡多处理(SMP)的配置
--------------------------------------------------------------------------------    

接下来的一步是解析 [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing) 的配置信息。我们调用 `find_smp_config` 函数来完成这个任务，这个函数内部调用另一个函数:    

```C
static inline void find_smp_config(void)
{
        x86_init.mpparse.find_smp_config();
}
```

在函数的内部，`x86_init.mpparse.find_smp_config` 函数就是 [arch/x86/kernel/mpparse.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/mpparse.c) 中的 `default_find_smp_config` 函数。我们调用 `default_find_smp_config` 函数扫描内存中的一些区域来寻找 `SMP` 的配置信息,并在找到它们的时候返回:   

```C
if (smp_scan_config(0x0, 0x400) ||
            smp_scan_config(639 * 0x400, 0x400) ||
            smp_scan_config(0xF0000, 0x10000))
            return;
```

首先 `smp_scan_config` 函数内部定义了一些变量:   

```C
unsigned int *bp = phys_to_virt(base);
struct mpf_intel *mpf;
``` 

第一个变量是我们用来扫描 `SMP` 配置的内存区域的虚拟地址；第二个变量是指向 `mpf_intel` 结构体的指针。让我们一起试着去理解 `mpf_intel` 是什么吧。所有的信息都存储在多处理器配置数据结构中。`mpf_intel` 就是这个结构，看下来像是下面这样: 

```C
struct mpf_intel {
        char signature[4];
        unsigned int physptr;
        unsigned char length;
        unsigned char specification;
        unsigned char checksum;
        unsigned char feature1;
        unsigned char feature2;
        unsigned char feature3;
        unsigned char feature4;
        unsigned char feature5;
};
```

正如我们在文档中看到的那样 - 系统 BIOS的主要功能之一就是创建MP浮点型指针结构和MP配置表。而且操作系统必须可以访问关于多处理器配置的有关信息， `mpf_intel` 中存储了多处理器配置表的物理地址(看结构体的第二个变量),然后，`smp_scan_config` 函数在指定的内存区域中循环查找 `MP floating pointer structure` 。这个函数还会检查当前字节是否指向 `SMP` 签名，然后检查签名的校验和，并且检查循环中的 `mpf->specification` 的值是1还是4(这个值只能是1或者是4):   

```C7
while (length > 0) {
if ((*bp == SMP_MAGIC_IDENT) &&
    (mpf->length == 1) &&
    !mpf_checksum((unsigned char *)bp, 16) &&
    ((mpf->specification == 1)
    || (mpf->specification == 4))) {

        mem = virt_to_phys(mpf);
        memblock_reserve(mem, sizeof(*mpf));
        if (mpf->physptr)
            smp_reserve_memory(mpf);
	}
}
```

如果搜索成功，就调用 `memblock_reserve` 函数保留一定的内存块，并且为多处理器配置表保留物理地址。你可以在 [MultiProcessor Specification](http://www.intel.com/design/pentium/datashts/24201606.pdf) 中找到相关的文档。你也可以在 `SMP` 的特定章节阅读更多细节。

其他的早期内存初始化程序
--------------------------------------------------------------------------------

在 `setup_arch` 的下一步，我们可以看到 `early_alloc_pgt_buf` 函数的调用,这个函数在早期阶段分配页表缓冲区。页表缓冲区将被放置在  `brk` 区段中。让我们一起看看这个功能的实现:   


```C
void  __init early_alloc_pgt_buf(void)
{
        unsigned long tables = INIT_PGT_BUF_SIZE;
        phys_addr_t base;

        base = __pa(extend_brk(tables, PAGE_SIZE));

        pgt_buf_start = base >> PAGE_SHIFT;
        pgt_buf_end = pgt_buf_start;
        pgt_buf_top = pgt_buf_start + (tables >> PAGE_SHIFT);
}
```

首先这个函数获得页表缓冲区的大小，它的值是 `INIT_PGT_BUF_SIZE` ，这个值在目前的linux 4.0 内核中是 `(6 * PAGE_SIZE)`。因为我们已经得到了页表缓冲区的大小，现在我们调用 `extend_brk` 函数并且传入两个参数: size和align。你可以从他们的名称中猜到,这个函数扩展 `brk` 区段。正如我们在linux内核链接脚本中看到的，`brk` 区段在内存中的位置恰好就在 [BSS](http://en.wikipedia.org/wiki/.bss) 区段后面:

```C
	. = ALIGN(PAGE_SIZE);
	.brk : AT(ADDR(.brk) - LOAD_OFFSET) {
		__brk_base = .;
		. += 64 * 1024;		/* 64k alignment slop space */
		*(.brk_reservation)	/* areas brk users have reserved */
		__brk_limit = .;
	}
```

我们也可以使用 `readelf` 工具来找到它:    

![brk area](http://oi61.tinypic.com/71lkeu.jpg) 

之后我们用 `_pa` 宏得到了新的 `brk` 区段的物理地址，我们计算页表缓冲区的基地址和结束地址。因为我们之前已经创建好了页面缓冲区，所以现在我们使用 `reserve_brk` 函数为 `brk` 区段保留内存块:   

```C
static void __init reserve_brk(void)
{
	if (_brk_end > _brk_start)
		memblock_reserve(__pa_symbol(_brk_start),
				 _brk_end - _brk_start);

	_brk_start = 0;
}
```

注意在 `reserve_brk` 的最后，我们把 `_brk_start` 赋值为0,因为在这之后我们不会再为 `brk` 分配内存了，我们需要使用 `cleanup_highmap` 函数来释放内核映射中越界的内存区域。请记住内核映射是 `__START_KERNEL_map` 和 `_end - _text` 或者 `level2_kernel_pgt` 对内核 `_text`、`data` 和 `bss` 区段的映射。在 `clean_high_map` 的开始部分我们定义下面这些参数: 

```C
unsigned long vaddr = __START_KERNEL_map;
unsigned long end = roundup((unsigned long)_end, PMD_SIZE) - 1;
pmd_t *pmd = level2_kernel_pgt;
pmd_t *last_pmd = pmd + PTRS_PER_PMD;
```

现在，因为我们已经定义了内核映射的开始和结束位置，所以我们在循环中遍历所有内核页中间目录条目, 并且清除不在 `_text` 和 `end` 区段中的条目:  

```C
for (; pmd < last_pmd; pmd++, vaddr += PMD_SIZE) {
        if (pmd_none(*pmd))
            continue;
        if (vaddr < (unsigned long) _text || vaddr > end)
            set_pmd(pmd, __pmd(0));
}
```

在这之后，我们使用 `memblock_set_current_limit` (你可以在[linux 内存管理 第二章节](https://github.com/MintCN/linux-insides-zh/blob/master/MM/linux-mm-2.md) 阅读关于 `memblock` 的更多内容) 函数来为 `memblock` 分配内存设置一个界限，这个界限可以是 `ISA_END_ADDRESS` 或者 `0x100000` ，然后调用 `memblock_x86_fill` 函数根据 `e820` 来填充 `memblock` 相关信息。你可以在内核初始化的时候看到这个函数运行的结果: 

```
MEMBLOCK configuration:
 memory size = 0x1fff7ec00 reserved size = 0x1e30000
 memory.cnt  = 0x3
 memory[0x0]	[0x00000000001000-0x0000000009efff], 0x9e000 bytes flags: 0x0
 memory[0x1]	[0x00000000100000-0x000000bffdffff], 0xbfee0000 bytes flags: 0x0
 memory[0x2]	[0x00000100000000-0x0000023fffffff], 0x140000000 bytes flags: 0x0
 reserved.cnt  = 0x3
 reserved[0x0]	[0x0000000009f000-0x000000000fffff], 0x61000 bytes flags: 0x0
 reserved[0x1]	[0x00000001000000-0x00000001a57fff], 0xa58000 bytes flags: 0x0
 reserved[0x2]	[0x0000007ec89000-0x0000007fffffff], 0x1377000 bytes flags: 0x0
```

除了 `memblock_x86_fill` 之外的其他函数还有: `early_reserve_e820_mpc_new` 函数在 `e820map` 中为多处理器规格表分配额外的槽， `reserve_real_mode` - 用于保留从 `0x0` 到1M的低端内存用作到实模式的跳板(用于重启等...)，`trim_platform_memory_ranges` 函数用于清除掉以 `0x20050000`, `0x20110000` 等地址开头的内存空间。这些内存区域必须被排除在外，因为 [Sandy Bridge](http://en.wikipedia.org/wiki/Sandy_Bridge) 会在这些内存区域出现一些问题， `trim_low_memory_range` 函数用于保留 `memblock` 中的前4KB页面，`init_mem_mapping` 函数用于在 `PAGE_OFFSET` 处重建物理内存的直接映射, `early_trap_pf_init` 函数用于建立 `#PF` 处理函数(我们将会在有关中断的章节看到它), `setup_real_mode` 函数用于建立到 [实模式](http://en.wikipedia.org/wiki/Real_mode) 代码的跳板。  

这就是本章的全部内容了。您可能注意到这部分并没有包括 `setup_arch` 中的所有函数 (如 `early_gart_iommu_check`、[mtrr](http://en.wikipedia.org/wiki/Memory_type_range_register) 的初始化函数等...)。正如我已经说了很多次的, `setup_arch` 函数很复杂，linux内核也很复杂。这就是为什么我不能包括linux内核中的每一行代码。我认为我们并没有错过重要的东西, 但是你可能会说: 每行代码都很重要。是的, 这没错, 但不管怎样我略过了他们, 因为我认为对于整个linux内核面面俱到是不现实的。无论如何, 我们会经常复习所学的内容, 如果有什么不熟悉的内容, 我们将会深入研究这些内容。

结束语
-------------------------------------------------------------------------------- 

这里是linux 内核初始化进程第六章节的结尾。在这一章节中，我们再次深入研究了 `setup_arch` 函数，然而这是个很长的部分，我们目前还没有学习完。的确, `setup_arch`很复杂，希望下个章节将会是这个函数的最后一个部分。。   

如果你有任何的疑问或者建议，你可以留言，也可以直接发消息给我[twitter](https://twitter.com/0xAX)。  

**很抱歉，英语并不是我的母语，非常抱歉给您阅读带来不便，如果你发现文中描述有任何问题，请提交一个 PR 到 [linux-insides](https://github.com/MintCN/linux-insides-zh).** 

链接
--------------------------------------------------------------------------------

* [MultiProcessor Specification](http://en.wikipedia.org/wiki/MultiProcessor_Specification)
* [NX bit](http://en.wikipedia.org/wiki/NX_bit)
* [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt)
* [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [CPU masks](http://0xax.gitbooks.io/linux-insides/content/Concepts/cpumask.html)
* [Linux kernel memory management](http://xinqiu.gitbooks.io/linux-insides-cn/content/MM/index.html)
* [PCI](http://en.wikipedia.org/wiki/Conventional_PCI)
* [e820](http://en.wikipedia.org/wiki/E820)
* [System Management BIOS](http://en.wikipedia.org/wiki/System_Management_BIOS)
* [System Management BIOS](http://en.wikipedia.org/wiki/System_Management_BIOS)
* [EFI](http://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)
* [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [MultiProcessor Specification](http://www.intel.com/design/pentium/datashts/24201606.pdf)
* [BSS](http://en.wikipedia.org/wiki/.bss)
* [SMBIOS specification](http://www.dmtf.org/sites/default/files/standards/documents/DSP0134v2.5Final.pdf)
* [前一个章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-5.html)
