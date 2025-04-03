# Linux 定时器和时间管理 第一部分

## 简介

这是本书 [linux-inside](https://github.com/hust-open-atom-club/linux-insides-zh/blob/master) 的新章节。前一[部分](../SysCall/linux-syscall-4.md)描述了[系统调用](https://en.wikipedia.org/wiki/System_call)的概念，现在是时候开始新的一章了。正如标题所示，本章将专门介绍 Linux内核中的 `定时器` 和 `时间管理`。本章的主题选择并非偶然。定时器(通常还有时间管理)非常重要，广泛地应用在 Linux 内核中。Linux 内核在多种任务中使用定时器，例如 [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) 中的不同超时实现，内核知道当前时间、调度异步函数、下一个事件中断调度等等。

因此，我们将在这一部分开始学习不同的与时间管理相关内容的实现。我们将看到不同类型的定时器，以及在不同的 Linux 内核子系统如何使用它们。与往常一样，我们将从 Linux 内核最早的部分开始，完成 Linux 内核的初始化过程。我们在描述 Linux 内核初始化过程的特殊[章](../Initialization/README.md)中已经这样做了，但您可能还记得，我们在那里遗漏了一些东西。其中之一是定时器的初始化。

我们开始吧！

## 非标准 PC 硬件时钟初始化

在 Linux 内核解压缩之后(有关这方面的更多信息，您可以在[内核解压](../Booting/linux-bootstrap-5.md)部分中阅读)，体系结构无关的代码在 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 源代码文件中开始工作。在初始化[锁验证器](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)、初始化 [cgroups](https://en.wikipedia.org/wiki/Cgroups) 和设置 [canary](https://en.wikipedia.org/wiki/Buffer_overflow_protection) 值之后，我们可以看到对 `setup_arch` 函数的调用。

首先是:

```C
x86_init.timers.wallclock_init();
```

在描述 Linux 内核初始化的章节中，我们已经看到了 `x86_init` 结构体。该结构包含了指向不同平台的默认设置函数的指针，如 [Intel MID](https://en.wikipedia.org/wiki/Mobile_Internet_device#Intel_MID_platforms)、[Intel CE4100](http://www.wpgholdings.com/epaper/US/newsRelease_20091215/255874.pdf) 等。`x86_init` 结构定义在 [arch/x86/kernel/x86_init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/x86_init.c#L36) 中，如你所见，它确定了标准 PC 硬件是默认选项。

如你所见，`x86_init` 结构的类型是 `x86_init_ops`，它为特定平台的设置提供了一组函数，如预留标准资源、平台相关的内存设置、初始化中断处理程序等。这个结构看起来像这样:

```C
struct x86_init_ops {
	struct x86_init_resources       resources;
    struct x86_init_mpparse         mpparse;
    struct x86_init_irqs            irqs;
    struct x86_init_oem             oem;
    struct x86_init_paging          paging;
    struct x86_init_timers          timers;
    struct x86_init_iommu           iommu;
    struct x86_init_pci             pci;
};
```

注意 `timers` 字段的类型是 `x86_init_timers`。顾名思义，该字段与时间管理和定时器有关。`x86_init_timers` 包含四个字段，它们都是返回 [void](https://en.wikipedia.org/wiki/Void_type)（注：没有返回值）的函数指针:

* `setup_percpu_clockkev`- 为启动 CPU 的每个 CPU 设置时钟事件设备;
* `tsc_pre_init` - 在 [TSC](https://en.wikipedia.org/wiki/Time_Stamp_Counter) 初始化之前调用的平台函数 [^tsc_pre_init];
* `timer_init` - 初始化平台定时器;
* `wallclock_init` - 初始化墙上时钟设备。

[^tsc_pre_init]: 译注：6.11.0-rc5 源码中没有发现 `tsc_pre_init` 字段。

所以，我们已经知道，此情况下，`wallclock_init` 会执行墙上时钟设备的初始化。在 `x86_init` 结构体的定义中，我们会看到 `wallclock_init` 指向 `x86_init_noop`[^x86_init_noop]:

[^x86_init_noop]: 译注：6.11.0-rc5 中 `wallclock_init` 指向了 `x86_wallclock_init`。

```C
struct x86_init_ops x86_init __initdata = {
	...
	...
	...
	.timers = {
		.wallclock_init		    = x86_init_noop,
	},
	...
	...
	...
}
```

其中 `x86_init_noop` 函数什么都没做:

```C
void __cpuinit x86_init_noop(void) { }
```

对于标准 PC 硬件。实际上，`wallclock_init`函数是在 [Intel MID](https://en.wikipedia.org/wiki/Mobile_Internet_device#Intel_MID_platforms) 平台上使用的。`x86_init.timers.wallclock_init` 的初始化在 [arch/x86/platform/intel-mid/intel-mid.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/platform/intel-mid/intel-mid.c) 源代码文件的 `x86_intel_mid_early_setup` 函数中[^x86_intel_mid_early_setup]：

[^x86_intel_mid_early_setup]: 译注：其中 `wallclock_init` 的赋值语句在 6.11.0-rc5 中去掉了。

```C
void __init x86_intel_mid_early_setup(void)
{
	...
	...
	...
	x86_init.timers.wallclock_init = intel_mid_rtc_init;
	...
	...
	...
}
```

`intel_mid_rtc_init` 函数的实现在 [arch/x86/platform/intel-mid/intel_mid_vrtc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/platform/intel-mid/intel_mid_vrtc.c) 源代码文件中，看起来非常简单。首先，该函数解析[简单固件接口](https://en.wikipedia.org/wiki/Simple_Firmware_Interface) [^sfi] M-Real-Time-Clock 表，将获取此类设备到 `sfi_mrtc_array` 数组并初始化 `set_time` 和 `get_time` 函数:

[^sfi]: 译注：SFI 支持在 Linux 5.12 被移除

```C
void __init intel_mid_rtc_init(void)
{
	unsigned long vrtc_paddr;

	sfi_table_parse(SFI_SIG_MRTC, NULL, NULL, sfi_parse_mrtc);

	vrtc_paddr = sfi_mrtc_array[0].phys_addr;
	if (!sfi_mrtc_num || !vrtc_paddr)
		return;

	vrtc_virt_base = (void __iomem *)set_fixmap_offset_nocache(FIX_LNW_VRTC,
								vrtc_paddr);

    x86_platform.get_wallclock = vrtc_get_time;
	x86_platform.set_wallclock = vrtc_set_mmss;
}
```

这就是全部，此后，基于 `Intel MID` 的设备将能够从硬件时钟中获取时间。如前所述，标准 PC [x86_64](https://en.wikipedia.org/wiki/X86-64) 架构不支持 `x86_init_noop`，在调用该函数期间什么都不做。我们刚刚看到了 [Intel MID](https://en.wikipedia.org/wiki/Mobile_Internet_device#Intel_MID_platforms) 架构 [实时时钟](https://en.wikipedia.org/wiki/Real-time_clock) 的初始化，现在是时候回到通用 `x86_64` 架构，并看看那里的时间管理相关的东西。

## 熟悉 jiffies

回到 `setup_arch` 函数(该函数位于 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c#L842) 源代码文件中)，会看到下一个被调用的时间管理相关的函数:

```C
register_refined_jiffies(CLOCK_TICK_RATE);
```

在我们看这个函数的实现之前，我们必须了解 [jiffy](https://en.wikipedia.org/wiki/Jiffy_%28time%29)。具体参阅维基百科 [^jeffy]:

```
Jiffy 是一个非正式术语，指任何未指定的一小段时间 
```

[^jeffy]: 译注：一般是进程运行时时间片的单位，我觉得可以翻译成 **须臾**

该定义与 Linux 内核中的 `jiffy` 非常相似。有一个名为 `jiffies` 的全局变量，它保存了自系统启动以来发生的时钟数。Linux 内核将该变量设置为 0:

```C
extern unsigned long volatile __jiffy_data jiffies;
```

初始化过程中。在定时器中断处理时，该全局变量每次都会增加。除此之外，在 `jiffies` 变量附近，我们可以看到类似变量的定义：

```C
extern u64 jiffies_64;
```

实际上，Linux 内核中只使用了其中一个变量，这取决于处理器类型。对于 [x86_64](https://en.wikipedia.org/wiki/X86-64)，它将使用 `u64`，对于 [x86](https://en.wikipedia.org/wiki/X86)，它是 `unsigned long`。我们可以在 [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vmlinux.lds.S) 链接脚本中看到:

```
#ifdef CONFIG_X86_32
...
jiffies = jiffies_64;
...
#else
...
jiffies_64 = jiffies;
...
#endif
```
在 `x86_32` 的情况下，`jiffies` 将是 `jiffies_64` 变量的较低的32位。从原理上讲，可以想象如下所示：

```
                    jiffies_64
+-----------------------------------------------------+
|                       |                             |
|                       |                             |
|                       |       jiffies on `x86_32`   |
|                       |                             |
|                       |                             |
+-----------------------------------------------------+
63                     31                             0
```

现在我们了解了一些关于 `jiffies` 的理论，可以回到我们的函数了。我们的函数 `register_refined_jiffies` 没有特定体系结构的实现。该函数位于通用的内核代码 [kernel/time/jiffies.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/jiffies.c) 文件中。`register_refined_jiffies` 的要点是注册jiffy 的 `clocksource`。在查看 `register_refined_jiffies` 函数的实现之前，我们必须知道 `clocksource` 是什么。可以在评论中看到:

```
`clocksource` 是一个自由运行计数器的硬件抽象。
```

我不知道你是怎么想的，但是这个描述并没有很好地理解 `clocksource` 的概念。让我们试着理解它是什么，但我们不会更深入，因为这个主题将在单独的部分中更详细地描述。`clocksource` 的要点是计时抽象，或者用非常简单的话说：它为内核提供了一个时间值。我们已经知道 `jiffies` 接口表示自系统启动以来发生的时钟数。在 Linux 内核中，它由一个全局变量表示，每个时钟中断都会增加一个计数器。Linux 内核可以使用 `jiffies` 进行时间测量。那么，为什么我们需要独立的时间上下文，比如 `clocksource`？实际上，不同的硬件设备提供不同的时间源，其功能也各不相同。更精确的时间间隔测量技术的可用性依赖于更高级的硬件。

例如，`x86` 在芯片上有一个 64 位计数器，称为[时间戳计数器](https://en.wikipedia.org/wiki/Time_Stamp_Counter)，其频率可以与处理器频率相等。或者[高精度事件定时器](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)，它由一个至少 ` 10mhz` 频率的 `64位` 计数器组成。两个不同的定时器，它们都是针对 `x86` 的。如果从其他体系结构添加定时器，只会使问题变得更加复杂。Linux 内核提供了 `clocksource`(时钟源) 的概念来解决这个问题。

时钟源的概念由 Linux 内核中的 `clocksource` 结构表示。该结构定义在 [include/linux/clocksource.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clocksource.h) 头文件中，其中包含了几个描述时间计数器的字段。例如，它包含 - `name` 字段是计数器的名称，`flags` 字段描述了计数器的不同属性，指向 `suspend` 和 `resume` 函数的指针等等。

让我们看一下 jiffies 的 `clocksource` 结构，它定义在 [kernel/time/jiffies.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/jiffies.c) 源代码文件中:

```C
static struct clocksource clocksource_jiffies = {
	.name		= "jiffies",
	.rating		= 1,
	.read		= jiffies_read,
	.mask		= 0xffffffff,
	.mult		= NSEC_PER_JIFFY << JIFFIES_SHIFT,
	.shift		= JIFFIES_SHIFT,
	.max_cycles	= 10,
};
```

我们可以在这里看到默认名称的定义 - `jiffies`。接下来是 `rating` 字段，它允许指定硬件上可用的时钟源管理代码，选择最佳注册时钟源。`rating` 可能有以下值:

* `1-99` - 仅用于启动和测试;
* `100-199` - 实用但不理想。
* `200-299` - 正确且可用的时钟源。
* `300-399` - 相当快且准确的时钟源。
* `400-499` - 理想的时钟源，在可用的地方必须使用;

例如，[时间戳计数器](https://en.wikipedia.org/wiki/Time_Stamp_Counter)的 `rating` 是 300，但[高精度事件定时器](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)的评级是 250。下一个字段是 `read`，它是一个指针，指向允许它读取 clocksource 的周期值的函数，或者换句话说，它只是返回 `cycle_t` 类型的 `jiffies` 变量:

```C
static cycle_t jiffies_read(struct clocksource *cs)
{
        return (cycle_t) jiffies;
}
```

`cycle_t` 只是64位无符号类型:

```C
typedef u64 cycle_t;
```

下一个字段是 `mask` 值，它确保非 `64位` 计数器的计数器值之间的减法不需要特殊的溢出逻辑。在我们的例子中，掩码是 `0xffffffff`，它是32位。这意味着 `jiffy` 在 42 秒后会返回 0:

```python
>>> 0xffffffff
4294967295
# 42 nanoseconds
>>> 42 * pow(10, -9)
4.2000000000000006e-08
# 43 nanoseconds
>>> 43 * pow(10, -9)
4.3e-08
```

接下来的两个字段 `mult` 和 `shift` 用于将时钟源的周期转换为每个周期的纳秒。当内核调用 `clocksource.clock` 时。函数，这个函数返回一个以机器时间单位表示的值，用我们刚才看到的 `cycle_t` 数据类型表示。要将这个返回值转换为[纳秒](https://en.wikipedia.org/wiki/Nanosecond)，我们需要这两个字段: `mult` 和`shift`。`clocksource` 提供了`clocksource_cyc2ns` 函数，它将使用以下表达式为我们完成这项工作:

```C
((u64) cycles * mult) >> shift;
```
我们可以看到 `mult` 字段是相等的:

```C
NSEC_PER_JIFFY << JIFFIES_SHIFT

#define NSEC_PER_JIFFY  ((NSEC_PER_SEC+HZ/2)/HZ)
#define NSEC_PER_SEC    1000000000L
```

默认情况下，`shift` 是

```C
#if HZ < 34
  #define JIFFIES_SHIFT   6
#elif HZ < 67
  #define JIFFIES_SHIFT   7
#else
  #define JIFFIES_SHIFT   8
#endif
```

`jiffies` 时钟源使用 `NSEC_PER_JIFFY` 乘法器转换来指定纳秒/周期比。请注意，`JIFFIES_SHIFT` 和 `NSEC_PER_JIFFY` 的值取决于 `HZ` 的值。`HZ` 表示系统定时器的频率。该宏定义在 [include/asm-generic/param.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/asm-generic/param.h) 中，并依赖于 `CONFIG_HZ` 内核配置选项。`HZ` 的值因支持的体系结构而异，但对于 `x86`，它的定义如下:

```C
#define HZ		CONFIG_HZ
```

其中 `CONFIG_HZ` 可以是以下值之一:

![HZ](images/HZ.png)

这意味着在我们的例子中，定时器中断频率是 250 HZ，或者每秒发生 250 次，或者每个 4ms 发生一个定时器中断。

在 `clocksource_jiffies` 结构体定义中，我们可以看到的最后一个字段是 - `max_cycles`，它保存了可以安全地相乘而不会导致溢出的最大周期值。

好的，我们刚刚看到了 `clocksource_jiffies` 结构体的定义，我们也了解了一些关于 `jiffies` 和 `clocksource` 的知识，现在是时候回到我们函数的实现了。在这一部分的开始时，我们停在了下面的调用:

```C
register_refined_jiffies(CLOCK_TICK_RATE);
```

在 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c#L842) 源代码文件中的函数。

正如我已经写的，`register_refined_jiffies` 函数的主要目的是注册 `refined_jiffies` 时钟源。我们已经知道 `clocksource_jiffies` 结构表示标准的 `jiffies` 时钟源。现在，如果查看[kernel/time/jiffies.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/jiffies.c) 源代码文件，你会发现另一个时钟源定义:

```C
struct clocksource refined_jiffies;
```

`refined_jiffies` 和 `clocksource_jiffies` 之间有一个区别:基于 `jiffies` 的标准时钟源是最小公分母时钟源，应该在所有系统上都能工作。我们已经知道，`jiffies` 全局变量会在每次定时器中断期间增加。这意味着基于 `jiffies` 的标准时钟源具有与定时器中断频率相同的分辨率。由此我们可以理解，基于 `jiffies` 的标准时钟源可能会受到不准确性的影响。`refined_jiffies` 使用 `CLOCK_TICK_RATE` 作为 `jiffies` 移位的基准。

让我们看一下这个函数的实现。首先，我们可以看到基于 `clocksource_jiffies` 结构的 `refined_jiffies` 时钟源:

```C
int register_refined_jiffies(long cycles_per_second)
{
	u64 nsec_per_tick, shift_hz;
	long cycles_per_tick;

	refined_jiffies = clocksource_jiffies;
	refined_jiffies.name = "refined-jiffies";
	refined_jiffies.rating++;
	...
	...
	...
```

在这里，我们可以看到，我们将 `refined_jiffies` 的名称更新为 `refined-jiffies`，并增加了此结构的 rating。如你所知，`clocksource_jiffies` 的 rating 为 1，所以我们的 `refined_jiffies`时钟源的 rating 为 2。这意味着 `refined_jiffies` 将是时钟源管理代码的最佳选择。

下一步，我们需要计算每个滴答的时钟周期数:

```C
cycles_per_tick = (cycles_per_second + HZ/2)/HZ;
```

请注意，我们使用 `NSEC_PER_SEC` 宏作为标准 `jiffies` 乘法器的基础。这里我们使用的是 `register_refined_jiffies` 函数的第一个参数 `cycles_per_second`。我们已经将 `CLOCK_TICK_RATE` 宏传递给了 `register_refined_jiffies` 函数。该宏定义在 [arch/x86/include/asm/timex.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/timex.h) 头文件中，并扩展为

```C
#define CLOCK_TICK_RATE         PIT_TICK_RATE
```

其中 `PIT_TICK_RATE` 宏扩展到[Intel 8253](可编程间隔定时器) 的频率:

```C
#define PIT_TICK_RATE 1193182ul
```

之后，我们为 `register_refined_jiffies` 计算 `shift_hz`，它将赋值为 `hz << 8`，或者换句话说，系统定时器的频率。我们将 `cycles_per_second` 或可编程间隔定时器的频率左移到`8`，以获得额外的精度:

```C
shift_hz = (u64)cycles_per_second << 8;
shift_hz += cycles_per_tick/2;
do_div(shift_hz, cycles_per_tick);
```

在下一步中，我们通过将 `NSEC_PER_SEC` 在 `8` 上左移来计算每个时钟周期的秒数，就像我们对 `shift_hz` 所做的那样，并进行与之前相同的计算:

```C
nsec_per_tick = (u64)NSEC_PER_SEC << 8;
nsec_per_tick += (u32)shift_hz/2;
do_div(nsec_per_tick, (u32)shift_hz);
```

```C
refined_jiffies.mult = ((u32)nsec_per_tick) << JIFFIES_SHIFT;
```

在 `register_refined_jiffies` 函数的末尾，我们使用在 [include/linux/clocksource.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clocksource.h) 头文件中定义的 `__clocksource_register` 函数注册新的时钟源并返回:

```C
__clocksource_register(&refined_jiffies);
return 0;
```

时钟源管理代码提供了时钟源注册和选择的 API。我们可以看到，时钟源是通过在内核初始化期间或从内核模块中调用 `__clocksource_register` 函数来注册的。在注册期间，时钟源管理代码将使用 `clocksource.rating` 来选择系统中可用的最佳时钟源。当我们为 `jiffies` 初始化 `clocksource` 结构时，已经看到了。

## 使用 the jiffies

在上一段中，我们看到了两个基于`jiffies`的时钟源的初始化:

* 基于标准 `jiffies` 的时钟源;
* 基于改进 `jiffies` 的时钟源;

如果你不理解这里的计算，也不用担心。它们一开始看起来很吓人。很快，我们就会一步一步地学习这些东西。因此，我们刚刚看到了基于 `jiffies` 的时钟源的初始化，我们也知道 Linux 内核有全局变量 `jiffies`，它保存了自内核开始工作以来发生的时标数量。现在，让我们看看如何使用它。要使用 `jiffies`，我们只需通过名称或调用 `get_jiffies_64` 函数来使用 `jiffies` 全局变量。这个函数定义在 [kernel/time/jiffies.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/jiffies.c) 源代码文件中，只返回完整的 `jiffies` 的64位值:

```C
u64 get_jiffies_64(void)
{
	unsigned long seq;
	u64 ret;

	do {
		seq = read_seqbegin(&jiffies_lock);
		ret = jiffies_64;
	} while (read_seqretry(&jiffies_lock, seq));
	return ret;
}
EXPORT_SYMBOL(get_jiffies_64);
```

请注意，例如 `get_jiffies_64` 函数并没有被实现为 `jiffies_read`:

```C
static cycle_t jiffies_read(struct clocksource *cs)
{
	return (cycle_t) jiffies;
}
```

我们可以看到 `get_jiffies_64` 的实现更复杂。`jiffies_64` 变量的读取是使用[seqlocks](https://en.wikipedia.org/wiki/Seqlock) 实现的。实际上，这是为那些不能原子读取完整 64 位值的机器而做的。

如果我们可以访问 `jiffies` 或 `jiffies_64` 变量，我们可以将其转换为 `human` 时间单位。为了获得一秒钟的时间，我们可以使用以下表达式:

```C
jiffies / HZ
```

如果我们知道这个，我们可以得到任何时间单位。例如:

```C
/* Thirty seconds from now */
jiffies + 30*HZ

/* Two minutes from now */
jiffies + 120*HZ

/* One millisecond from now */
jiffies + HZ / 1000
```

以上。

结论
--------------------------------------------------------------------------------

到此为止，第一部分涵盖了 Linux 内核中与时间和时间管理相关的概念。我们首先遇到了 `jiffies` 和 `clocksource` 两个概念及其初始化。在下一部分中，我们将继续深入这个有趣的主题，如前所述，我们将尝试理解 Linux 内核中时间管理概念的内部原理。

如果你有问题或者建议，可随时在推特上联系我 [0xAX](https://twitter.com/0xAX), 给我发送 [email](mailto:anotherworldofworld@gmail.com) 或者直接创建 [issue](https://github.com/0xAX/linux-insides/issues/new).

**请注意，英语不是我的母语，给您带来的不便我感到非常抱歉。如果你发现任何错误，请提交 PR 到 [linux-insides](https://github.com/0xAX/linux-insides).**

链接
--------------------------------------------------------------------------------

* [系统调用](https://en.wikipedia.org/wiki/System_call)
* [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)
* [锁验证器](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)
* [cgroups](https://en.wikipedia.org/wiki/Cgroups)
* [bss](https://en.wikipedia.org/wiki/.bss)
* [initrd](https://en.wikipedia.org/wiki/Initrd)
* [Intel MID](https://en.wikipedia.org/wiki/Mobile_Internet_device#Intel_MID_platforms)
* [TSC](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [void](https://en.wikipedia.org/wiki/Void_type)
* [简单固件接口](https://en.wikipedia.org/wiki/Simple_Firmware_Interface)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [实时时钟](https://en.wikipedia.org/wiki/Real-time_clock)
* [Jiffy](https://en.wikipedia.org/wiki/Jiffy_%28time%29)
* [高精度事件定时器](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)
* [纳秒](https://en.wikipedia.org/wiki/Nanosecond)
* [Intel 8253](https://en.wikipedia.org/wiki/Intel_8253)
* [seqlocks](https://en.wikipedia.org/wiki/Seqlock)
* [cloksource 文档](https://www.kernel.org/doc/Documentation/timers/timekeeping.txt)
* [前一章](https://0xax.gitbook.io/linux-insides/summary/syscall)
