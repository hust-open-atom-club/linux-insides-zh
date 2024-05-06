Linux内核中的定时器和时间管理 第6部分
================================================================================

x86_64相关的时钟资源
--------------------------------------------------------------------------------

这是[本章](/Timers/)的第六部分，描述了Linux内核中的定时器和时间管理的相关内容。上一节中，我们了解了clockevents框架，现在继续深入研究Linux内核中的时间管理相关内容，本节将讲述x86架构中时钟源的实现（更多关于时钟源的概念可以参考本章第二节）。

首先，我们需要知道x86架构上可以使用哪些时钟源。这个问题很容易从[sysfs](https://en.wikipedia.org/wiki/Sysfs)或文件`/sys/devices/system/clocksource/clocksource0/available_clocksource`中获得答案。文件夹`/sys/devices/system/clocksource/clocksourceN`内有两个特殊文件保存：

* `available_clocksource` - 提供系统中可用的时钟资源信息。
* `current_clocksource`   - 提供系统中当前使用的时钟资源。

所以，来试一下：

```
$ cat /sys/devices/system/clocksource/clocksource0/available_clocksource 
tsc hpet acpi_pm 
```

可以看到有三个已注册的时钟资源：

* `tsc` - [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter);
* `hpet` - [High Precision Event Timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer);
* `acpi_pm` - [ACPI Power Management Timer](http://uefi.org/sites/default/files/resources/ACPI_5.pdf).


现在来看第二个文件，其中记录了最好的时钟资源（系统中，拥有最高频率的时钟资源）:
```
$ cat /sys/devices/system/clocksource/clocksource0/current_clocksource 
tsc
```
作者的系统中是[Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)。由[本章第二节内容](/Timers/linux-timers-2.md)可知，系统中最好的时钟源是具有最佳（最高）等级的时钟源，或者说是具有最高频率的时钟源。

[ACPI](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)电源管理时钟的频率是3.579545MHz。而[High Precision Event Timer(高精度事件定时器)](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)的频率至少是10MHz，而[Time Stamp Counter(时间戳计数器)](https://en.wikipedia.org/wiki/Time_Stamp_Counter)的频率取决于处理器。例如在较早的处理器上，`TSC`用来计算处理器内部的时钟周期，就是说当处理器的频率比生变化时，其频率也会发生变化。这种现象在较新的处理器上有所改善。新的处理器有一个不变的时间戳计数器，无论处理器在什么状态下都会以恒定的速率递增。我们可以在`/proc/cpuinfo`的输出中获得它的频率。例如：

```
$ cat /proc/cpuinfo
...
model name	: Intel(R) Core(TM) i7-4790K CPU @ 4.00GHz
...
```
而尽管英特尔的开发者手册说，`TSC`的频率虽然是恒定的，但不一定是处理器的最大频率或者品牌名称中中给出的频率。总之，可以发现，TSC远超`ACPI PM`计时器以及`HPET`的频率，而且具有最佳速度或最高频率的时钟源是系统中当前正在使用的时钟。

注意到，除了这三个时钟源之外，在`/sys/devices/system/clocksource/clocksource0/available_clocksource`的输出中没有看到另外两个熟悉的时钟源，`jiffy`和`refined_jiffies`。之所以看不到它们，是因为这个文件只映射高分辨率的时钟源，也就是带有[CLOCK_SOURCE_VALID_FOR_HRES](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clocksource.h#L113)标志的时钟源。

正如上面所述，本节将会涵盖所有这三个时钟源，将按照它们初始化的顺序来逐一分析。

* `hpet`
* `acpi_pm`
* `tsc`

在dmesg的输出中，有确定的顺序：

```
$ dmesg | grep clocksource
[    0.000000] clocksource: refined-jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 1910969940391419 ns
[    0.000000] clocksource: hpet: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 133484882848 ns
[    0.094369] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 1911260446275000 ns
[    0.186498] clocksource: Switched to clocksource hpet
[    0.196827] clocksource: acpi_pm: mask: 0xffffff max_cycles: 0xffffff, max_idle_ns: 2085701024 ns
[    1.413685] tsc: Refined TSC clocksource calibration: 3999.981 MHz
[    1.413688] clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x73509721780, max_idle_ns: 881591102108 ns
[    2.413748] clocksource: Switched to clocksource tsc
```

第一个时钟源是 [High Precision Event Timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)，那就从它开始。

High Precision Event Timer
--------------------------------------------------------------------------------
用于[x86](https://en.wikipedia.org/wiki/X86)架构的HPET的内核代码位于[arch/x86/kernel/hpet.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/hpet.c)文件中。它的初始化是从调用`hpet_enable`函数开始的。这个函数在Linux内核初始化时被调用。从[init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)文件中的`start_kernel`函数中可以发现，在所有那些'架构特有的'的事物被初始化之后，以及'early console'被禁用，并且时间管理子系统已经准备就绪时，调用以下函数。
```C
if (late_time_init)
	late_time_init();
```
该函数在早期jiffy计数器被初始化后，对后期的架构特有的定时器进行初始化。`x86`架构的`late_time_init`函数的定义位于[arch/x86/kernel/time.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/time.c) 文件中。它看起来这样：

```C
static __init void x86_late_time_init(void)
{
	x86_init.timers.timer_init();
	tsc_init();
}
```

可以看到，这里完成`x86`相关定时器的初始化和`TSC`的初始化。现在来考虑调用函数`x86_init.timers.timer_init`。`timer_init`指向同一源文件中的`hpet_time_init`。可以通过查看 `x86_init`结构图的定义来验证这一点。
[arch/x86/kernel/x86_init.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/x86_init.c):

```C
struct x86_init_ops x86_init __initdata = {
   ...
   ...
   ...
   .timers = {
		.setup_percpu_clockev	= setup_boot_APIC_clock,
		.timer_init		= hpet_time_init,
		.wallclock_init		= x86_init_noop,
   },
   ...
   ...
   ...
```

如果`HPET`支持没有开启，那么函数`hpet_time_init` 会初始化[programmable interval timer](https://en.wikipedia.org/wiki/Programmable_interval_timer)，并且设置默认时钟[IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29):

```C
void __init hpet_time_init(void)
{
	if (!hpet_enable())
		setup_pit_timer();
	setup_default_timer_irq();
}
```

首先，函数`hpet_enable`通过调用`is_hpet_capable'检查能否在系统中启用`HPET`，如果可以，我们就为它映射一个虚拟地址空间。

```C
int __init hpet_enable(void)
{
	if (!is_hpet_capable())
		return 0;

    hpet_set_mapping();
}
```

函数`is_hpet_capable`确认没有向内核命令行传递`hpet=disable`，并且`hpet_address`是来自表[ACPI HPET](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)。函数`hpet_set_mapping`为时钟相关寄存器映射虚拟地址空间。

```C
hpet_virt_address = ioremap_nocache(hpet_address, HPET_MMAP_SIZE);
```

[IA-PC HPET (High Precision Event Timers) Specification](http://www.intel.com/content/dam/www/public/us/en/documents/technical-specifications/software-developers-hpet-spec-1-0a.pdf) 有讲述：

> 时钟寄存器空间有1024字节

因此，`HPET_MMAP_SIZE` 也是 `1024`字节。

```C
#define HPET_MMAP_SIZE		1024
```

在为`HPET`映射了虚拟地址空间之后，就可以通过读寄存器`HPET_ID`得到时钟号:

```C
id = hpet_readl(HPET_ID);

last = (id & HPET_ID_NUMBER) >> HPET_ID_NUMBER_SHIFT;
```

这个数字是用来为`HPET`的 `配置寄存器` 分配适当大小的空间。

```C
cfg = hpet_readl(HPET_CFG);

hpet_boot_cfg = kmalloc((last + 2) * sizeof(*hpet_boot_cfg), GFP_KERNEL);
```

在为 `HPET`的配置寄存器分配空间后，主计时钟开始运行，并可以通过配置寄存器的`HPET_CFG_ENABLE`位，为每一个时钟设置定时器中断。前提是，所有的时钟都通过配置寄存器中的`HPET_CFG_ENABLE`位所启用。最后，我们仅通过调用`hpet_clocksource_register`函数来注册新的时钟源。

```C
if (hpet_clocksource_register())
	goto out_nohpet;
```

这个函数调用已经很熟悉了：

```C
clocksource_register_hz(&clocksource_hpet, (u32)hpet_freq);
```

其中`clocksource_hpet`是`clocksource`结构体对象，成员`rating`是`250`（之前`refined_jiffies`时钟源的`rating`是`2`），`hpet`和`read_hpet`两个回调函数用于读取`HPET`提供的原子计数器。

```C
static struct clocksource clocksource_hpet = {
	.name		= "hpet",
	.rating		= 250,
	.read		= read_hpet,
	.mask		= HPET_MASK,
	.flags		= CLOCK_SOURCE_IS_CONTINUOUS,
	.resume		= hpet_resume_counter,
	.archdata	= { .vclock_mode = VCLOCK_HPET },
};
```
在注册`clocksource_hpet`后，可以回看[arch/x86/kernel/time.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/time.c)源文件中的函数`hpet_time_init()`。最后一步的调用：

```C
setup_default_timer_irq();
```

函数`setup_default_timer_irq`检查`legacy`IRQ是否存在，也就是对[i8259](https://en.wikipedia.org/wiki/Intel_8259)的支持，并且配置[IRQ0](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29#Master_PIC)。

代码到这里[High Precision Event Timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)，时钟源在Linux内核的时钟框架中完成注册，可以在内核中使用`read_hpet`。

```C
static cycle_t read_hpet(struct clocksource *cs)
{
	return (cycle_t)hpet_readl(HPET_COUNTER);
}
```
该函数读取并返回`Main Counter Register`中的原子计数器。

ACPI PM timer
--------------------------------------------------------------------------------

第二个时钟源是[ACPI Power Management Timer](http://uefi.org/sites/default/files/resources/ACPI_5.pdf)。这个时钟源的实现位于[drivers/clocksource/acpi_pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/clocksource_acpi_pm.c)源文件中，从`fs`[initcall](https://kernelnewbies.org/Documents/InitcallMechanism)中调用`init_acpi_pm_clocksource`函数开始。
如果看一下 `init_acpi_pm_clocksource`函数的实现，会发现它是从检查 `pmtmr_ioport`变量的值开始的。

```C
static int __init init_acpi_pm_clocksource(void)
{
    ...
    ...
    ...
	if (!pmtmr_ioport)
		return -ENODEV;
    ...
    ...
    ...
```
变量`pmtmr_ioport`包含`Power Management Timer Control Register Block`的扩展地址。在源文件[arch/x86/kernel/acpi/boot.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/acpi/boot.c) 中定义的函数`acpi_parse_fadt`中获取其值。该函数解析 `FADT` 或 `Fixed ACPI Description Table` [ACPI](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface) 并获取包含扩展地址的 `X_PM_TMR_BLK` 字段的值`Power Management Timer Control Register Blcok`, 并以结构体`Generic Address Structure`格式表示：

```C
static int __init acpi_parse_fadt(struct acpi_table_header *table)
{
#ifdef CONFIG_X86_PM_TIMER
        ...
        ...
        ...
		pmtmr_ioport = acpi_gbl_FADT.xpm_timer_block.address;
        ...
        ...
        ...
#endif
	return 0;
}
```
因此，如果内核配置`CONFIG_X86_PM_TIMER`被禁用，或者`acpi_parse_fadt`函数出错，就不能访问`Power Management Timer`中的寄存器，并从`init_acpi_pm_clocksource`返回。也就是说，如果`pmtmr_ioport`变量的值不是0，就会检查这个时钟的速率，并通过调用下面这个函数来注册这个时钟源。

```C
clocksource_register_hz(&clocksource_acpi_pm, PMTMR_TICKS_PER_SEC);
```
调用函数`clocksource_register_hs`之后，`acpi_pm` 时钟源被注册到`clocksource` 内核框架中:
```C
static struct clocksource clocksource_acpi_pm = {
	.name		= "acpi_pm",
	.rating		= 200,
	.read		= acpi_pm_read,
	.mask		= (cycle_t)ACPI_PM_MASK,
	.flags		= CLOCK_SOURCE_IS_CONTINUOUS,
};
```
成员`rating` 是 `200`，并且`acpi_pm_read`回调函数读`apci_pm`时钟源提供的原子计数器。 函数`acpi_pm_read`正是执行`read_pmtmr`:

```C
static cycle_t acpi_pm_read(struct clocksource *cs)
{
	return (cycle_t)read_pmtmr();
}
```

这个函数读`Power Management Timer`寄存器的值。寄存器结构如下：

```
+-------------------------------+----------------------------------+
|                               |                                  |
|  upper eight bits of a        |      running count of the        |
| 32-bit power management timer |     power management timer       |
|                               |                                  |
+-------------------------------+----------------------------------+
31          E_TMR_VAL           24               TMR_VAL           0
```
这个寄存器的地址是存在`Fixed ACPI Description Table` [ACPI](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface) 表中，并且可以通过`pmtmr_ioport`访问。所以，函数`read_pmtmr`的实现就非常简单了：

```C
static inline u32 read_pmtmr(void)
{
	return inl(pmtmr_ioport) & ACPI_PM_MASK;
}
```
只需要读去寄存器`Power Management Timer`的值，并且取出第`24`位。

现在来看本章最后一个时钟源`Time Stamp Counter`。

Time Stamp Counter
--------------------------------------------------------------------------------

这第三个也是最后一个时钟源是[Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)，它的实现位于源文件[arch/x86/kernel/tsc.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/tsc.c)。前文已经看到过函数`x86_late_time_init`，以及[Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)的初始化函数，也从这个开始，这个函数调用了`tsc_init()` 。

在函数`tsc_init`开始的地方，可以看到它确认处理器是否支持`Time Stamp Counter`:

```C
void __init tsc_init(void)
{
	u64 lpj;
	int cpu;

	if (!cpu_has_tsc) {
		setup_clear_cpu_cap(X86_FEATURE_TSC_DEADLINE_TIMER);
		return;
	}
    ...
    ...
    ...
```

宏`cpu_has_tsc`展开，调用宏`cpu_has` macro:

```C
#define cpu_has_tsc		boot_cpu_has(X86_FEATURE_TSC)
#define boot_cpu_has(bit)	cpu_has(&boot_cpu_data, bit)
#define cpu_has(c, bit)							\
	(__builtin_constant_p(bit) && REQUIRED_MASK_BIT_SET(bit) ? 1 :	\
	 test_cpu_cap(c, bit))
```
上面的宏检查在内核初始化时填充的`boot_cpu_data`数组中的给定位，这里是`X86_FEATURE_TSC_DEADLINE_TIMER`。如果处理器支持`Time Stamp Counter`，通过调用同一源代码文件中的`calibrate_tsc`函数来获得`TSC`的频率，该函数会尝试从不同的时钟源获得频率，如[MSR](https://en.wikipedia.org/wiki/Model-specific_register)，通过[programmable interval timer](https://en.wikipedia.org/wiki/Programmable_interval_timer)校准等等，之后为系统中所有处理器初始化频率和比例因子。

```C
tsc_khz = x86_platform.calibrate_tsc();
cpu_khz = tsc_khz;

for_each_possible_cpu(cpu) {
	cyc2ns_init(cpu);
	set_cyc2ns_scale(cpu_khz, cpu);
}
```

因为只有第一个引导处理器会调用 `tsc_init`，此后，检查`TSC`是否被禁用。

```
if (tsc_disabled > 0)
	return;
...
...
...
check_system_tsc_reliable();
```

并调用函数`check_system_tsc_reliable`，如果bootstrap处理器有`X86_FEATURE_TSC_RELIABLE`特性，则设置`tsc_clocksource_reliable`。注意，到这里函数`tsc_init`结束，但没有注册时钟源。实际注册`TSC`时钟源是在:

```C
static int __init init_tsc_clocksource(void)
{
	if (!cpu_has_tsc || tsc_disabled > 0 || !tsc_khz)
		return 0;
    ...
    ...
    ...
    if (boot_cpu_has(X86_FEATURE_TSC_RELIABLE)) {
		clocksource_register_khz(&clocksource_tsc, tsc_khz);
		return 0;
	}
```
这个函数在`device`[initcall](https://kernelnewbies.org/Documents/InitcallMechanism)期间调用。这样做是为了确保`TSC` 时钟源在[HPET](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)时钟源之后被注册。
在这之后，所有三个时钟源都在 `clocksource`框架中注册，`TSC`时钟源将被选为当前时钟源，因为它其他时钟源中具有最高等级。

```C
static struct clocksource clocksource_tsc = {
	.name                   = "tsc",
	.rating                 = 300,
	.read                   = read_tsc,
	.mask                   = CLOCKSOURCE_MASK(64),
	.flags                  = CLOCK_SOURCE_IS_CONTINUOUS | CLOCK_SOURCE_MUST_VERIFY,
	.archdata               = { .vclock_mode = VCLOCK_TSC },
};
```

Conclusion
--------------------------------------------------------------------------------

这是[本章](/Timers/)的第六节，描述了Linux内核中的时钟和时钟管理。上一节中，熟悉了`clockevents`框架。这一节中，继续学习了Linux内核中时钟管理，并且看到了在[x86](https://en.wikipedia.org/wiki/X86)架构中使用的三种不同的时钟源。下一节将是[本章](/Timers/)的最后一节，将看到一些与用户空间有关的事情，即一些与时间有关的[系统调用](https://en.wikipedia.org/wiki/System_call)如何在Linux内核中实现。
如果有问题或建议，请随时在twitter[0xAX](https://twitter.com/0xAX)上与我联系，给我发[email](mailto:anotherworldofworld@gmail.com)或直接创建[issue](https://github.com/0xAX/linux-insides/issues/new)。
**请注意，英语不是我的第一语言，我真的很抱歉给你带来的不便。如果你发现任何错误，请给我发送PR到[linux-insides](https://github.com/0xAX/linux-insides)**。

链接
--------------------------------------------------------------------------------

* [x86](https://en.wikipedia.org/wiki/X86)
* [sysfs](https://en.wikipedia.org/wiki/Sysfs)
* [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [High Precision Event Timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)
* [ACPI Power Management Timer (PDF)](http://uefi.org/sites/default/files/resources/ACPI_5.pdf)
* [frequency](https://en.wikipedia.org/wiki/Frequency).
* [dmesg](https://en.wikipedia.org/wiki/Dmesg)
* [programmable interval timer](https://en.wikipedia.org/wiki/Programmable_interval_timer)
* [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 
* [IA-PC HPET (High Precision Event Timers) Specification](http://www.intel.com/content/dam/www/public/us/en/documents/technical-specifications/software-developers-hpet-spec-1-0a.pdf)
* [IRQ0](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29#Master_PIC)
* [i8259](https://en.wikipedia.org/wiki/Intel_8259)
* [initcall](http://www.compsoc.man.ac.uk/~moz/kernelnewbies/documents/initcall/kernel.html)
* [previous part](/Timers/linux-timers-5.md)
