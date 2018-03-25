内核初始化 第六部分
===========================================================

仍旧是与系统架构有关的初始化
===========================================================  

在之前的[章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-5.html)我们从 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c)了解了特定系统架构 (在我们的例子中是 `x86_64` )的初始化内容，并且通过 `x86_configure_nx` 函数根据对[NX bit](http://en.wikipedia.org/wiki/NX_bit)的支持设置 `_PAGE_NX` 标志。正如我之前写的, `setup_arch` 函数和 `start_kernel` 都非常庞大，所以在这个和下个章节我们将继续学习关于系统架构初始化进程的内容。`x86_configure_nx` 函数的下面是 `parse_early_param`。这个函数的定义在 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 并且你可以从它的名字中了解到，这个函数解析内核命令行并且基于给定的参数 (所有的内核命令行参数你都可以在 [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt) 找到)。 你可能记得在最前面的 [章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-2.html) 我们怎样安装 `earlyprintk`。在早期我们在 [arch/x86/boot/cmdline.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cmdline.c)里面的 `cmdline_find_option` 函数和 `__cmdline_find_option`, `__cmdline_find_option_bool` 的帮助下寻找内核参数和他们的值。我们在通用内核部分不依赖于系统架构，在这里我们使用另一种方法。 如果你正在阅读linux内核源代码，你可能注意到这样的调用：

```C
early_param("gbpages", parse_direct_gbpages_on);
```

`early_param` 宏需要两个参数:

* 命令行参数的名称
* 如果给定的参数通过，函数将被调用 

并且定义如下: 

```C
#define early_param(str, fn) \
        __setup_param(str, fn, fn, 1)
```

这个定义可以在 [include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h)中可以找到.  
正如你所看到的  `early_param` 宏只是调用了 `__setup_param` 宏:

```C
#define __setup_param(str, unique_id, fn, early)                \
        static const char __setup_str_##unique_id[] __initconst \
                __aligned(1) = str; \
        static struct obs_kernel_param __setup_##unique_id      \
                __used __section(.init.setup)                   \
                __attribute__((aligned((sizeof(long)))))        \
                = { __setup_str_##unique_id, fn, early }
```

这个宏定义了 `__setup_str_*_id` 变量 (这里的 `*` 取决于被给定的函数名称) 并且把给定的命令行参数赋值给这个变量。在下一行中，我们可以看到类型为 `obs_kernel_param` 的变量 `__setup_ *` 的定义及其初始化。   

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
* 功能取决于参数的函数  
* 决定参数是否为 early 的标记位 (译者注:这个参数是个标记位，有0和1两种值，两种值的后续调用是不一样的)

注意 `__set_param` 宏定义有 `__section(.init.setup)` 属性。这意味着所有 `__setup_str_ *` 将被放置在 `.init.setup` 区段中，此外正如我们在 [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/vmlinux.lds.h) 中看到的，他们将被放置在 `__setup_start` 和 `__setup_end` 之间:   

```
#define INIT_SETUP(initsetup_align)                \
                . = ALIGN(initsetup_align);        \
                VMLINUX_SYMBOL(__setup_start) = .; \
                *(.init.setup)                     \
                VMLINUX_SYMBOL(__setup_end) = .;
```

现在我们知道了参数是怎样定义的，让我们一起回到 `parse_early_param` 的实现:   

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
`parse_early_param` 函数定义了两个静态变量。首先第一个变量 `done` 用来检查 `parse_early_param` 函数是否已经被调用，第二个变量是用来临时存储内核命令行的。在这之后我们把 `boot_command_line` 赋值到我们刚才定义的临时命令行变量中( `tmp_cmdline` ) 并且从相同的源代码文件 `main.c` 中调用 `parse_early_options` 函数。 `parse_early_options`函数从 [kernel/params.c](https://github.com/torvalds/linux/blob/master/) 中调用 `parse_args` 函数, `parse_args` 解析传入的命令行并且调用 `do_early_param` 函数。 这个 [函数](https://github.com/torvalds/linux/blob/master/init/main.c#L413) 从 ` __setup_start` 循环到 `__setup_end` ，并且如果 `obs_kernel_param` 中的 `early` 字段值为early(1) ,从 `obs_kernel_param` 调用函数(译者注:调用结构体中的第二个参数，这个参数是个函数)。在这之后所有基于早期命令行参数的服务都被创建，在 `parse_early_param` 之后的下一个调用是 `x86_report_nx` 。 正如我在这章开头所写的，我们已经用 `x86_configure_nx` 配置了 `NX-bit` 位。接下来的 [arch/x86/mm/setup_nx.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/setup_nx.c)中的 `x86_report_nx`函数仅仅打印出关于 `NX` 的信息。注意我们 `x86_report_nx` 不一定在 `x86_configure_nx` 之后调用，但是一定在 `parse_early_param` 之后调用。答案很简单: 因为内核支持 `noexec` 参数，所以我们在 `parse_early_param` 之后调用 `x86_report_nx` :

```
noexec		[X86]
			On X86-32 available only on PAE configured kernels.
			noexec=on: enable non-executable mappings (default)
			noexec=off: disable non-executable mappings
```

我们可以在启动的时候看到:

![NX](http://oi62.tinypic.com/swwxhy.jpg)

在这之后我们可以看到下面函数的调用:   

```C
	memblock_x86_reserve_range_setup_data();
```

这个函数被定义在相同的源代码文件 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c) 中，并且为 `setup_data` 重新映射内存，保留内存块(你可以阅读之前的 [章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-5.html) 了解关于 `setup_data` 的更多内容，你也可以在[Linux kernel memory management](http://xinqiu.gitbooks.io/linux-insides-cn/content/MM/index.html) 中阅读到关于 `ioremap` and `memblock` 的内容)。 

接下来我们来看看下面的条件语句:    

```C
	if (acpi_mps_check()) {
#ifdef CONFIG_X86_LOCAL_APIC
		disable_apic = 1;
#endif
		setup_clear_cpu_cap(X86_FEATURE_APIC);
	}
```

[arch/x86/kernel/acpi/boot.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/acpi/boot.c) 中的函数 `acpi_mps_check` 取决于 `CONFIG_X86_LOCAL_APIC` 和 `CONFIG_x86_MPPARSE` 配置选项:   

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

这个函数检查内置的 `MPS` 又称 [多重处理器规范]((http://en.wikipedia.org/wiki/MultiProcessor_Specification)) 表。如果设置了 ` CONFIG_X86_LOCAL_APIC` 但未设置 `CONFIG_x86_MPPAARSE` ，并且传递给内核的命令行参数是 `acpi=off`、`acpi=noirq` 或者 `pci=noacpi`，那么`acpi_mps_check` 就会打印出警告信息。如果 `acpi_mps_check` 返回1，这表示我们警用了本地 [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) 
,而且通过 `setup_clear_cpu_cap` 宏清除了当前CPU中的 `X86_FEATURE_APIC` 位。（你可以阅读[CPU masks](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-2.html)了解更多关于CPU mask的内容)。

早期的PCI转储
--------------------------------------------------------------------------------  

接下来我们使用下面的代码来转储 [PCI](http://en.wikipedia.org/wiki/Conventional_PCI) 设备: 

```C
#ifdef CONFIG_PCI
	if (pci_early_dump_regs)
		early_dump_pci_devices();
#endif
```

`pci_early_dump_regs` 定义在 [arch/x86/pci/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/pci/common.c) 中，并且他的值是取决于内核命令行参数：`pci=earlydump` 。我们可以在[drivers/pci/pci.c](https://github.com/torvalds/linux/blob/master/arch) 中看到这个参数的定义:   

```C
early_param("pci", pci_setup);
```

`pci_setup` 函数获得 `pci=` 之后的字符串，然后进行分析。这个函数调用了在 [drivers/pci/pci.c](https://github.com/torvalds/linux/blob/master/arch) 中用 `_weak` 修饰符定义的 `pcibios_setup` 函数，并且每种架构都重写了 `_weak` 修饰过的函数。 例如 依赖于 `x86_64` 架构的版本写在[arch/x86/pci/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/pci/common.c)中:  

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

所以，如果 `CONFIG_PCI` 选项被设置，而且我们向内核命令行传递了 `pci=earlydump` 选项，下一个被调用的函数是在 [arch/x86/pci/early.c](https://github.com/torvalds/linux/blob/master/arch/x86/pci/early.c)中的 `early_dump_pci_devices` 。这个函数使用下面代码检查 `noearly` pci 参数:   

```C
if (!early_pci_allowed())
        return;
```

同时如果条件通过则返回。每个PCI域可以承载多达 `256` 条总线，并且每条总线可以承载多达32个设备。那么接下来我们进行下面的循环:   

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

并且通过 `read_pci_config` 函数读取 `pci` 配置。 

这就是 pci 加载的全部了。我们在这里不会深入研究 `pci` 的细节，不过我们会在 `Drivers/PCI` 章节看到更多的细节。 

完成内存解析  
--------------------------------------------------------------------------------   


在 `early_dump_pci_devices` 之后，有一些与可用内存和[e820](http://en.wikipedia.org/wiki/E820)相关的函数并且 [e820](http://en.wikipedia.org/wiki/E820) ,其中 [e820](http://en.wikipedia.org/wiki/E820) 我们在 [内核安装的第一步](http://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-2.html) 章节中整理过。   
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

让我们来一起看看上面的代码。正如你所看到的，第一个函数是 `e820_reserve_setup_data` 。这个函数和我们前面看到的 `memblock_x86_reserve_range_setup_data` 函数做的事情几乎是相同的，但是这个函数同时还会调用 `e820_update_range` 函数，向 `e820map` 中用给定的类型添加新的区域，在我们的例子中，使用的是 `E820_RESERVED_KERN` 类型。接下来的函数是 `finish_e820_parsing` 。除了这两个函数之外，我们还可以看到一些与 [e820](http://en.wikipedia.org/wiki/E820) 有关的函数。你可以在上面的清单中看到这些函数。`e820_add_kernel_range` 函数需要内核开始和结束的物理地址:   

```C
u64 start = __pa_symbol(_text);
u64 size = __pa_symbol(_end) - start;
``` 

函数会检查在 `e820map` 中被标记成 `E820RAM` 的 `.text` `.data` 和 `.bss` 段，如果没有这些字段，那么就会输出错误信息。接下来的 `trm_bios_range` 函数把 `e820Map` 中的前4096个字节更新为 `E820_RESERVED` 并且调用函数 `sanitize_e820_map` 进行处理。在这之后我们使用 `e820_end_of_ram_pfn` 得到最后一个页面帧的编号，每个内存页面都有一个唯一的编号 - `页面帧编号`， `e820_end_of_ram_pfn` 函数调用 `e820_end_pfn` 函数返回这个最大的页面帧编号:   
```C
unsigned long __init e820_end_of_ram_pfn(void)
{
	return e820_end_pfn(MAX_ARCH_PFN);
}
```

`e820_end_pfn` 函数读取当前系统架构的最大页面帧编号(对于 `x86_64` 架构来说 `MAX_ARCH_PFN` 是 `0x400000000` )。在 `e820_end_pfn` 函数中我们遍历整个 `e820` 插槽，并且检查 `e820` 入口是否有 `E820_RAM` 或者 `E820_PRAM` 类型，因为我们只能对这些类型计算页面帧编码，得到当前 `e820` 入口页面帧的基地址和结束地址，同时对这些地址进行检查:   

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

接下来我们检查在循环中得到的 `last_pfn` 不得大于特定系统架构的最大页面帧编号(在我们的例子中是 `x86_64` 系统架构),输出关于最大页面帧编码的信息，并且返回它。我们可以在 `dmesg` 的输出中看到 `last_pfn` :

```
...
[    0.000000] e820: last_pfn = 0x41f000 max_arch_pfn = 0x400000000
...
```

在这之后，我们计算出了最大的页面帧编号，我们要计算 `max_low_pfn` ,这是 `小内存` 或者低于第一个4GB中的的的最大页面帧。如果安装了超过4GB的内存RAM，`max_low_pfn` 将会是 `e820_end_of_low_ram_pfn` 函数的结果，这个函数和 `e820_end_of_ram_pfn` 相似，但是有4GB自己的限制，换句话说 `max_low_pfn` 和 `max_pfn` 的值是一样的:   

```C
if (max_pfn > (1UL<<(32 - PAGE_SHIFT)))
	max_low_pfn = e820_end_of_low_ram_pfn();
else
	max_low_pfn = max_pfn;
		
high_memory = (void *)__va(max_pfn * PAGE_SIZE - 1) + 1;
```

接下来我们通过 `__va` 宏计算 `大内存` 中的最大页面帧编号(有更高的直接内存映射上界),这个宏根据给定的物理内存返回一个虚拟地址。 


直接媒体结构扫描   
-------------------------------------------------------------------------------

在处理完不同内存区域和 `e820` 插槽之后的下一步就是收集有关计算机的信息。我们将获得与 [桌面管理接口](http://en.wikipedia.org/wiki/Desktop_Management_Interface) 和下面函数的所有信息:

```C
dmi_scan_machine();
dmi_memdev_walk();
```

首先是定义在 [drivers/firmware/dmi_scan.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/firmware/dmi_scan.c) 中的 `dmi_scan_machine` 函数。这个函数浏览 [System Management BIOS](http://en.wikipedia.org/wiki/System_Management_BIOS) 结构，从中提取信息。制定了两种方法来获得 `SMBIOS` 表的访问权: 第一种是从 [EFI](http://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) 的配置表获得指向 `SMBIOS` 表的指针；第二种是扫描 `0xF0000` 和 `0x10000` 地址之间的物理地址。让我们一起看看第二种方法。`dmi_scan_machine` 函数通过 `dmi_early_remap` 函数将 `0xf0000` 和 `0x10000` 之间的内存重新映射到 `early_ioremap`:

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
然后遍历所有的 `DMI` 头地址，并且查找 `_SM_` 字符串:   

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
`_SM_` 字符串一定在 `000F0000h` 和 `0x000FFFFF` 地址之间。在这里我们用 `memcpy_fromio` 函数向 `buf` 里面复制16个字节，这个函数和 `memcpy` 函数的作用是一样的。然后对这个缓冲区( `buf` ) 执行`dmi_smbios3_present` 和 `dmi_present` 函数。这些函数检查 `buf`的前4个字节是否是 `__SM__` 字符串，得到 `SMBIOS` 的版本号，并且获得 `_DMI_` 的属性例如 `_DMI_` 结构表长度、表的地址等等...每当其中的一个函数完成之后，你就会在 `dmesg` 的输出中看到函数的运行结果:   
```
[    0.000000] SMBIOS 2.7 present.
[    0.000000] DMI: Gigabyte Technology Co., Ltd. Z97X-UD5H-BK/Z97X-UD5H-BK, BIOS F6 06/17/2014
```
在 `dmi_scan_machine` 函数的最后，我们取消之前重新映射的内存:   

```C
dmi_early_unmap(p, 0x10000);
```

第二个函数是 - `dmi_memdev_walk`。正如你理解的那样，这个函数遍历整个内存设备。让我们一起看看这个函数:   

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

这个函数检查 `DMI` 是否可用(我们之前在 `dmi_scan_machine` 函数中得到了这个结果)并且使用 `dmi_walk_early` 和 `dmi_alloc` 函数收集关于内存设备的信息,其中  `dmi_alloc` 的定义如下:   

```
#ifdef CONFIG_DMI
RESERVE_BRK(dmi_alloc, 65536);
#endif
```

`RESERVE_BRK` 定义在 [arch/x86/include/asm/setup.h](http://en.wikipedia.org/wiki/Desktop_Management_Interface)中，并且这个函数在 `brk` 段中保留给定大小的空间:   

-------------------------
	init_hypervisor_platform();
	x86_init.resources.probe_roms();
	insert_resource(&iomem_resource, &code_resource);
	insert_resource(&iomem_resource, &data_resource);
	insert_resource(&iomem_resource, &bss_resource);
	early_gart_iommu_check();    


均衡多处理(SMP)的配置
--------------------------------------------------------------------------------    

接下来的一步是解析 [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing) 的配置信息。我们调用 `find_smp_config` 函数来完成这个任务，这个函数只是调用另一个函数:    

```C
static inline void find_smp_config(void)
{
        x86_init.mpparse.find_smp_config();
}
```

在函数的内部，`x86_init.mpparse.find_smp_config` 就是 [arch/x86/kernel/mpparse.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/mpparse.c) 中的 `default_find_smp_config` 函数。在 `default_find_smp_config` 函数中我们扫描一些内存区域来寻找 `SMP` 的配置信息,并在找到它们的时候返回:   

```C
if (smp_scan_config(0x0, 0x400) ||
            smp_scan_config(639 * 0x400, 0x400) ||
            smp_scan_config(0xF0000, 0x10000))
            return;
```

首先 `smp_scan_config` 函数定义了一些变量:   

```C
unsigned int *bp = phys_to_virt(base);
struct mpf_intel *mpf;
``` 

第一个变量是我们用来扫描 `SMP` 配置内存区域的虚拟地址；第二个变量是指向 `mpf_intel` 结构体的指针。让我们一起试着去理解 `mpf_intel` 是什么吧。所有的信息都存储在多处理器配置数据结构中。`mpf_intel` 呈现了这个结构，看下来像是下面这样: 

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