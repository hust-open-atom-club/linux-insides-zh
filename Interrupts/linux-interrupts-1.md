中断和中断处理 Part 1.
================================================================================

Introduction
--------------------------------------------------------------------------------

这是 [linux 内核揭秘](http://xinqiu.gitbooks.io/linux-insides-cn/content/) 这本书最新章节的第一部分。我们已经在这本书前面的[章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/index.html)中走过了漫长的道路。从内核初始化的[第一步](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-1.html)开始，结束于第一个 `init` 程序的[启动](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-10.html)。我们见证了一系列与各种内核子系统相关的初始化步骤，但是我们并没有深入这些子系统。在这一章中，我们将会试着去了解这些内核子系统是如何工作和实现的。就像你在这章标题中看到的，第一个子系统是[中断（interrupts）](http://en.wikipedia.org/wiki/Interrupt)。

什么是中断？
--------------------------------------------------------------------------------

我们已经在这本书的很多地方听到过 `中断（interrupts）` 这个词，也看到过很多关于中断的例子。在这一章中我们将会从下面的主题开始：

* 什么是 `中断（interrupts）` ？
* 什么是 `中断处理（interrupt handlers）` ？

我们将会继续深入探讨 `中断` 的细节和 Linux 内核如何处理这些中断。

所以，首先什么是中断？中断就是当软件或者硬件需要使用 CPU 时引发的 `事件（event）`。比如，当我们在键盘上按下一个键的时候，我们下一步期望做什么？操作系统和电脑应该怎么做？做一个简单的假设，每一个物理硬件都有一根连接 CPU 的中断线，设备可以通过它对 CPU 发起中断信号。但是中断信号并不是直接发送给 CPU。在老机器上中断信号发送给 [PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller) ，它是一个顺序处理各种设备的各种中断请求的芯片。在新机器上，则是[高级程序中断控制器（Advanced Programmable Interrupt Controller）](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)做这件事情，即我们熟知的 `APIC`。一个 APIC 包括两个独立的设备：

* `Local APIC`
* `I/O APIC`

第一个设备 -  `Local APIC ` 存在于每个CPU核心中，Local APIC 负责处理特定于 CPU 的中断配置。Local APIC 常被用于管理来自 APIC 时钟（APIC-timer）、热敏元件和其他与 I/O 设备连接的设备的中断。

第二个设备 -  `I/O APIC` 提供了多核处理器的中断管理。它被用来在所有的 CPU 核心中分发外部中断。更多关于 local 和 I/O APIC 的内容将会在这一节的下面讲到。就如你所知道的，中断可以在任何时间发生。当一个中断发生时，操作系统必须立刻处理它。但是 `处理一个中断` 是什么意思呢？当一个中断发生时，操作系统必须确保下面的步骤顺序：

* 内核必须暂停执行当前进程(取代当前的任务)；
* 内核必须搜索中断处理程序并且转交控制权(执行中断处理程序)；
* 中断处理程序结束之后，被中断的进程能够恢复执行。

当然，在这个中断处理程序中会涉及到很多错综复杂的过程。但是上面 3 条是这个程序的基本骨架。

每个中断处理程序的地址都保存在一个特殊的位置，这个位置被称为 `中断描述符表（Interrupt Descriptor Table）` 或者 `IDT`。处理器使用一个唯一的数字来识别中断和异常的类型，这个数字被称为 `中断标识码（vector number）`。一个中断标识码就是一个 `IDT` 的标识。中断标识码范围是有限的，从 `0` 到 `255`。你可以在 Linux 内核源码中找到下面的中断标识码范围检查代码：

```C
BUG_ON((unsigned)n > 0xFF);
```

你可以在 Linux 内核源码中关于中断设置的地方找到这个定义(例如：`set_intr_gate`, `void set_system_intr_gate` 在 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h)中)。从 `0` 到 `31` 的 32 个中断标识码被处理器保留，用作处理架构定义的异常和中断。你可以在 Linux 内核初始化程序的第二部分 - [早期中断和异常处理](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-2.html)中找到这个表和关于这些中断标识码的描述。从 `32` 到 `255` 的中断标识码设计为用户定义中断并且不被系统保留。这些中断通常分配给外部 I/O 设备，使这些设备可以发送中断给处理器。

现在，我们来讨论中断的类型。笼统地来讲，我们可以把中断分为两个主要类型：

* 外部或者硬件引起的中断；
* 软件引起的中断。

第一种类型 - 外部中断，由 `Local APIC` 或者与 `Local APIC` 连接的处理器针脚接收。第二种类型 - 软件引起的中断，由处理器自身的特殊情况引起(有时使用特殊架构的指令)。一个常见的关于特殊情况的例子就是 `除零`。另一个例子就是使用 `系统调用（syscall）` 退出程序。

就如之前提到过的，中断可以在任何时间因为超出代码和 CPU 控制的原因而发生。另一方面，异常和程序执行 `同步（synchronous）` ，并且可以被分为 3 类：

* `故障（Faults）`
* `陷入（Traps）`
* `终止（Aborts）`

`故障` 是在执行一个“不完善的”指令（可以在之后被修正）之前被报告的异常。如果发生了，它允许被中断的程序继续执行。

接下来的 `陷入` 是一个在执行了 `陷入` 指令后立刻被报告的异常。陷入同样允许被中断的程序继续执行，就像 `故障` 一样。

最后的 `终止` 是一个从不报告引起异常的精确指令的异常，并且不允许被中断的程序继续执行。

我们已经从前面的[部分](http://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-3.html)知道，中断可以分为 `可屏蔽的（maskable）` 和 `不可屏蔽的（non-maskable）`。可屏蔽的中断可以被阻塞，使用 `x86_64` 的指令 - `sti` 和 `cli`。我们可以在 Linux 内核代码中找到他们：

```C
static inline void native_irq_disable(void)
{
        asm volatile("cli": : :"memory");
}
```

and

```C
static inline void native_irq_enable(void)
{
        asm volatile("sti": : :"memory");
}
```

这两个指令修改了在中断寄存器中的 `IF` 标识位。 `sti` 指令设置 `IF` 标识，`cli` 指令清除这个标识。不可屏蔽的中断总是被报告。通常，任何硬件上的失败都映射为不可屏蔽中断。

如果多个异常或者中断同时发生，处理器以事先设定好的中断优先级处理他们。我们可以定义下面表中的从最低到最高的优先级：

```
+----------------------------------------------------------------+
|              |                                                 |
|   Priority   | Description                                     |
|              |                                                 |
+--------------+-------------------------------------------------+
|              | Hardware Reset and Machine Checks               |
|     1        | - RESET                                         |
|              | - Machine Check                                 |
+--------------+-------------------------------------------------+
|              | Trap on Task Switch                             |
|     2        | - T flag in TSS is set                          |
|              |                                                 |
+--------------+-------------------------------------------------+
|              | External Hardware Interventions                 |
|              | - FLUSH                                         |
|     3        | - STOPCLK                                       |
|              | - SMI                                           |
|              | - INIT                                          |
+--------------+-------------------------------------------------+
|              | Traps on the Previous Instruction               |
|     4        | - Breakpoints                                   |
|              | - Debug Trap Exceptions                         |
+--------------+-------------------------------------------------+
|     5        | Nonmaskable Interrupts                          |
+--------------+-------------------------------------------------+
|     6        | Maskable Hardware Interrupts                    |
+--------------+-------------------------------------------------+
|     7        | Code Breakpoint Fault                           |
+--------------+-------------------------------------------------+
|     8        | Faults from Fetching Next Instruction           |
|              | Code-Segment Limit Violation                    |
|              | Code Page Fault                                 |
+--------------+-------------------------------------------------+
|              | Faults from Decoding the Next Instruction       |
|              | Instruction length > 15 bytes                   |
|     9        | Invalid Opcode                                  |
|              | Coprocessor Not Available                       |
|              |                                                 |
+--------------+-------------------------------------------------+
|     10       | Faults on Executing an Instruction              |
|              | Overflow                                        |
|              | Bound error                                     |
|              | Invalid TSS                                     |
|              | Segment Not Present                             |
|              | Stack fault                                     |
|              | General Protection                              |
|              | Data Page Fault                                 |
|              | Alignment Check                                 |
|              | x87 FPU Floating-point exception                |
|              | SIMD floating-point exception                   |
|              | Virtualization exception                        |
+--------------+-------------------------------------------------+
```

现在我们了解了一些关于各种类型的中断和异常的内容，是时候转到更实用的部分了。我们从 `中断描述符表（IDT）` 开始。就如之前所提到的，`IDT` 保存了中断和异常处理程序的入口指针。`IDT` 是一个类似于 `全局描述符表（Global Descriptor Table）`的结构，我们在[内核启动程序](http://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-2.html)的第二部分已经介绍过。但是他们确实有一些不同，`IDT` 的表项被称为 `门（gates）`，而不是 `描述符（descriptors）`。它可以包含下面的一种：

* 中断门（Interrupt gates）
* 任务门（Task gates）
* 陷阱门（Trap gates）

在 `x86` 架构中，只有 [long mode](http://en.wikipedia.org/wiki/Long_mode) 中断门和陷阱门可以在 `x86_64` 中引用。就像 `全局描述符表`，`中断描述符表` 在 `x86` 上是一个 8 字节数组门，而在 `x86_64` 上是一个 16 字节数组门。让我们回忆在[内核启动程序](http://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-2.html)的第二部分，`全局描述符表` 必须包含 `NULL` 描述符作为它的第一个元素。与 `全局描述符表` 不一样的是，`中断描述符表` 的第一个元素可以是一个门。它并不是强制要求的。比如，你可能还记得我们只是在早期的章节中过渡到[保护模式](http://en.wikipedia.org/wiki/Protected_mode)时用 `NULL` 门加载过中断描述符表：

```C
/*
 * Set up the IDT
 */
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

在 [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pm.c)中。`中断描述符表` 可以在线性地址空间和基址的任何地方被加载，只要在 `x86` 上以 8 字节对齐，在 `x86_64` 上以 16 字节对齐。`IDT` 的基址存储在一个特殊的寄存器 - IDTR。在 `x86` 上有两个指令 - 协调工作来修改 `IDTR` 寄存器： 

* `LIDT`
* `SIDT`

第一个指令 `LIDT` 用来加载 `IDT` 的基址，即在 `IDTR` 的指定操作数。第二个指令 `SIDT` 用来在指定操作数中读取和存储 `IDTR` 的内容。在 `x86` 上 `IDTR` 寄存器是 48 位，包含了下面的信息：

```
+-----------------------------------+----------------------+
|                                   |                      |
|     Base address of the IDT       |   Limit of the IDT   |
|                                   |                      |
+-----------------------------------+----------------------+
47                                16 15                    0
```

让我们看看 `setup_idt` 的实现，我们准备了一个 `null_idt`，并且使用 `lidt` 指令把它加载到 `IDTR` 寄存器。注意，`null_idt` 是 `gdt_ptr` 类型，后者定义如下：

```C
struct gdt_ptr {
        u16 len;
        u32 ptr;
} __attribute__((packed));
```

这里我们可以看看 `IDTR` 结构的定义，就像我们在示意图中看到的一样，由 2 字节和 4 字节（共 48 位）的两个域组成。现在，让我们看看 `IDT` 入口结构体，它是一个在 `x86` 中被称为门的 16 字节数组。它拥有下面的结构：

```
127                                                                             96
+-------------------------------------------------------------------------------+
|                                                                               |
|                                Reserved                                       |
|                                                                               |
+--------------------------------------------------------------------------------
95                                                                              64
+-------------------------------------------------------------------------------+
|                                                                               |
|                               Offset 63..32                                   |
|                                                                               |
+-------------------------------------------------------------------------------+
63                               48 47      46  44   42    39             34    32
+-------------------------------------------------------------------------------+
|                                  |       |  D  |   |     |      |   |   |     |
|       Offset 31..16              |   P   |  P  | 0 |Type |0 0 0 | 0 | 0 | IST |
|                                  |       |  L  |   |     |      |   |   |     |
 -------------------------------------------------------------------------------+
31                                   16 15                                      0
+-------------------------------------------------------------------------------+
|                                      |                                        |
|          Segment Selector            |                 Offset 15..0           |
|                                      |                                        |
+-------------------------------------------------------------------------------+
```

为了把索引格式化成 IDT 的格式，处理器把异常和中断向量分为 16 个级别。处理器处理异常和中断的发生就像它看到 `call` 指令时处理一个程序调用一样。处理器使用中断或异常的唯一的数字或 `中断标识码` 作为索引来寻找对应的 `中断描述符表` 的条目。现在让我们更近距离地看看 `IDT` 条目。

就像我们所看到的一样，在表中的 `IDT` 条目由下面的域组成：

* `0-15` bits  - 段选择器偏移，处理器用它作为中断处理程序的入口指针基址；
* `16-31` bits - 段选择器基址，包含中断处理程序入口指针；
* `IST` - 在 `x86_64` 上的一个新的机制，下面我们会介绍它；
* `DPL` - 描述符特权级；
* `P` - 段存在标志；
* `48-63` bits - 中断处理程序基址的第二部分；
* `64-95` bits - 中断处理程序基址的第三部分；
* `96-127` bits - CPU 保留位.

`Type` 域描述了 `IDT` 条目的类型。有三种不同的中断处理程序：

* 中断门（Interrupt gate）
* 陷入门（Trap gate）
* 任务门（Task gate）

`IST` 或者说是 `Interrupt Stack Table` 是 `x86_64` 中的新机制，它用来代替传统的栈切换机制。之前的 `x86` 架构提供的机制可以在响应中断时自动切换栈帧。`IST` 是 `x86` 栈切换模式的一个修改版，在它使能之后可以无条件地切换栈，并且可以被任何与确定中断（我们将在下面介绍它）关联的 `IDT` 条目中的中断使能。从这里可以看出，`IST` 并不是所有的中断必须的，一些中断可以继续使用传统的栈切换模式。`IST` 机制在[任务状态段（Task State Segment）](http://en.wikipedia.org/wiki/Task_state_segment)或者 `TSS` 中提供了 7 个 `IST` 指针。`TSS` 是一个包含进程信息的特殊结构，用来在执行中断或者处理 Linux 内核异常的时候做栈切换。每一个指针都被 `IDT` 中的中断门引用。

`中断描述符表` 使用 `gate_desc` 的数组描述：

```C
extern gate_desc idt_table[];
```

`gate_desc` 定义如下：

```C
#ifdef CONFIG_X86_64
...
...
...
typedef struct gate_struct64 gate_desc;
...
...
...
#endif
```

`gate_struct64` 定义如下：

```C
struct gate_struct64 {
        u16 offset_low;
        u16 segment;
        unsigned ist : 3, zero0 : 5, type : 5, dpl : 2, p : 1;
        u16 offset_middle;
        u32 offset_high;
        u32 zero1;
} __attribute__((packed));
```

在 `x86_64` 架构中，每一个活动的线程在 Linux 内核中都有一个很大的栈。这个栈的大小由 `THREAD_SIZE` 定义，而且与下面的定义相等：

```C
#define PAGE_SHIFT      12
#define PAGE_SIZE       (_AC(1,UL) << PAGE_SHIFT)
...
...
...
#define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

`PAGE_SIZE` 是 `4096` 字节，`THREAD_SIZE_ORDER` 的值依赖于 `KASAN_STACK_ORDER`。就像我们看到的，`KASAN_STACK` 依赖于 `CONFIG_KASAN` 内核配置参数，它定义如下：

```C
#ifdef CONFIG_KASAN
    #define KASAN_STACK_ORDER 1
#else
    #define KASAN_STACK_ORDER 0
#endif
```

`KASan` 是一个运行时内存[调试器](http://lwn.net/Articles/618180/)。所以，如果 `CONFIG_KASAN` 被禁用，`THREAD_SIZE` 是 `16384` ；如果内核配置选项打开，`THREAD_SIZE` 的值是 `32768`。这块栈空间保存着有用的数据，只要线程是活动状态或者僵尸状态。但是当线程在用户空间的时候，这个内核栈是空的，除非 `thread_info` 结构（关于这个结构的详细信息在 Linux 内核初始程序的第四[部分](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-4.html)）在这个栈空间的底部。活动的或者僵尸线程并不是在他们栈中的唯一的线程，与每一个 CPU 关联的特殊栈也存在于这个空间。当内核在这个 CPU 上执行代码的时候，这些栈处于活动状态；当在这个 CPU 上执行用户空间代码时，这些栈不包含任何有用的信息。每一个 CPU 也有一个特殊的 per-cpu 栈。首先是给外部中断使用的 `中断栈（interrupt stack）`。它的大小定义如下： 

```C
#define IRQ_STACK_ORDER (2 + KASAN_STACK_ORDER)
#define IRQ_STACK_SIZE (PAGE_SIZE << IRQ_STACK_ORDER)
```

或者是 `16384` 字节。Per-cpu 的中断栈在 `x86_64` 架构中使用 `irq_stack_union` 联合描述:

```C
union irq_stack_union {
	char irq_stack[IRQ_STACK_SIZE];

    struct {
		char gs_base[40];
		unsigned long stack_canary;
	};
};
```

第一个 `irq_stack` 域是一个 16KB 的数组。然后你可以看到 `irq_stack_union` 联合包含了一个结构体，这个结构体有两个域：

* `gs_base` - 总是指向 `irqstack` 联合底部的 `gs` 寄存器。在 `x86_64` 中， per-cpu（更多关于 `per-cpu` 变量的信息可以阅读特定的[章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html)） 和 stack canary 共享 `gs` 寄存器。所有的 per-cpu 标志初始值为零，并且 `gs` 指向 per-cpu 区域的开始。你已经知道[段内存模式](http://en.wikipedia.org/wiki/Memory_segmentation)已经废除很长时间了，但是我们可以使用[特殊模块寄存器（Model specific registers）](http://en.wikipedia.org/wiki/Model-specific_register)给这两个段寄存器 - `fs` 和 `gs` 设置基址，并且这些寄存器仍然可以被用作地址寄存器。如果你记得 Linux 内核初始程序的第一[部分](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-1.html)，你会记起我们设置了 `gs` 寄存器：

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr
```

`initial_gs` 指向 `irq_stack_union`:

```assembly
GLOBAL(initial_gs)
.quad	INIT_PER_CPU_VAR(irq_stack_union)
```

* `stack_canary` - [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries) 对于中断栈来说是一个用来验证栈是否已经被修改的 `栈保护者（stack protector）`。`gs_base` 是一个 40 字节的数组，`GCC` 要求 stack canary 在被修正过的偏移量上，并且 `gs` 的值在 `x86_64` 架构上必须是 `40`，在 `x86` 架构上必须是 `20`。 

`irq_stack_union` 是 `percpu` 的第一个数据, 我们可以在 `System.map`中看到它：

```
0000000000000000 D __per_cpu_start
0000000000000000 D irq_stack_union
0000000000004000 d exception_stacks
0000000000009000 D gdt_page
...
...
...
```

我们可以看到它在代码中的定义:

```C
DECLARE_PER_CPU_FIRST(union irq_stack_union, irq_stack_union) __visible;
```

现在，是时候来看 `irq_stack_union` 的初始化过程了。除了 `irq_stack_union` 的定义，我们可以在[arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/processor.h)中查看下面的 per-cpu 变量

```C
DECLARE_PER_CPU(char *, irq_stack_ptr);
DECLARE_PER_CPU(unsigned int, irq_count);
```

第一个就是 `irq_stack_ptr`。从这个变量的名字中可以知道，它显然是一个指向这个栈顶的指针。第二个 `irq_count` 用来检查 CPU 是否已经在中断栈。`irq_stack_ptr` 的初始化在[arch/x86/kernel/setup_percpu.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup_percpu.c)的 `setup_per_cpu_areas` 函数中：

```C
void __init setup_per_cpu_areas(void)
{
...
...
#ifdef CONFIG_X86_64
for_each_possible_cpu(cpu) {
    ...
    ...
    ...
    per_cpu(irq_stack_ptr, cpu) =
            per_cpu(irq_stack_union.irq_stack, cpu) +
            IRQ_STACK_SIZE - 64;
    ...
    ...
    ...
#endif
...
...
}
```

现在，我们一个一个查看所有 CPU，并且设置 `irq_stack_ptr`。事实证明它等于中断栈的顶减去 `64`。为什么是 `64`？TODO [[arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpu/common.c)] 代码如下：

```C
void load_percpu_segment(int cpu)
{
        ...
        ...
        ...
        __loadsegment_simple(gs, 0);
        wrmsrl(MSR_GS_BASE, cpu_kernelmode_gs_base(cpu));
        ...
        load_stack_canary_segment();
}
```

正如我们所知的一样，`gs` 寄存器指向中断栈的栈底：

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr

SYM_DATA(initial_gs,
.quad INIT_PER_CPU_VAR(fixed_percpu_data))
```

现在我们可以看到 `wrmsr` 指令，这个指令从 `edx:eax` 加载数据到 被 `ecx` 指向的[MSR寄存器]((http://en.wikipedia.org/wiki/Model-specific_register))。在这里MSR寄存器是 `MSR_GS_BASE`，它保存了被 `gs` 寄存器指向的内存段的基址。`edx:eax` 指向 `initial_gs` ，的地址，它就是 `fixed_percpu_data`	的基址。

我们还知道，`x86_64` 有一个叫 `中断栈表（Interrupt Stack Table）` 或者 `IST` 的组件，当发生不可屏蔽中断、双重错误等等的时候，这个组件提供了切换到新栈的功能。这可以到达7个 `IST` per-cpu 入口。其中一些如下;


* `DOUBLEFAULT_STACK`
* `NMI_STACK`
* `DEBUG_STACK`
* `MCE_STACK`

或者

```C
#define DOUBLEFAULT_STACK 1
#define NMI_STACK 2
#define DEBUG_STACK 3
#define MCE_STACK 4
```

所有被 `IST` 切换到新栈的中断门描述符都由 `set_intr_gate_ist` 函数初始化。例如:

```C
static const __initconst struct idt_data def_idts[] = {
    ...
	INTG(X86_TRAP_NMI,		nmi),
    ...
	INTG(X86_TRAP_DF,		double_fault),
```

其中 `&nmi` 和 `&double_fault` 在以下位置创建入口点：


[arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S)中

```assembly
idtentry double_fault			do_double_fault			has_error_code=1 paranoid=2 read_cr2=1
...
...
...
SYM_CODE_START(nmi)
...
...
...
SYM_CODE_END(nmi)
SYM_CODE_END(nmi)
```
在以下位置给出了中断处理程序的声明 [arch/x86/include/asm/traps.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/traps.h):
```C
asmlinkage void nmi(void);
asmlinkage void double_fault(void);
```

当一个中断或者异常发生时，新的 `ss` 选择器被强制置为 `NULL`，并且 `ss` 选择器的 `rpl` 域被设置为新的 `cpl`。旧的 `ss`、`rsp`、寄存器标志、`cs`、`rip` 被压入新栈。在 64 位模型下，中断栈帧大小固定为 8 字节，所以我们可以得到下面的栈:

```
+---------------+
|               |
|      SS       | 40
|      RSP      | 32
|     RFLAGS    | 24
|      CS       | 16
|      RIP      | 8
|   Error code  | 0
|               |
+---------------+
```

如果在中断门中 `IST` 域不是 `0`，我们把 `IST` 读到 `rsp` 中。如果它关联了一个中断向量错误码，我们再把这个错误码压入栈。如果中断向量没有错误码，就继续并且把虚拟错误码压入栈。我们必须做以上的步骤以确保栈一致性。接下来我们从门描述符中加载段选择器域到 CS 寄存器中，并且通过验证第 `21` 位的值来验证目标代码是一个 64 位代码段，例如 `L` 位在 `全局描述符表（Global Descriptor Table）`。最后我们从门描述符中加载偏移域到 `rip` 中，`rip` 是中断处理函数的入口指针。然后中断函数开始执行，在中断函数执行结束后，它必须通过 `iret` 指令把控制权交还给被中断进程。`iret` 指令无条件地弹出栈指针（`ss:rsp`）来恢复被中断的进程，并且不会依赖于 `cpl` 改变。

这就是中断的所有过程。

总结
--------------------------------------------------------------------------------

关于 Linux 内核的中断和中断处理的第一部分至此结束。我们初步了解了一些理论和与中断和异常相关的初始化条件。在下一部分，我会接着深入了解中断和中断处理 - 更深入了解她真实的样子。

如果你有任何问题或建议，请给我发评论或者给我发 [Twitter](https://twitter.com/0xAX)。

**请注意英语并不是我的母语，我为任何表达不清楚的地方感到抱歉。如果你发现任何错误请发 PR 到 [linux-insides](https://github.com/MintCN/linux-insides-zh)。(译者注：翻译问题请发 PR 到 [linux-insides-cn](https://www.gitbook.com/book/xinqiu/linux-insides-cn))**


链接
--------------------------------------------------------------------------------

* [PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller)
* [Advanced Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [long mode](http://en.wikipedia.org/wiki/Long_mode)
* [kernel stacks](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)
* [Task State Segement](http://en.wikipedia.org/wiki/Task_state_segment)
* [segmented memory model](http://en.wikipedia.org/wiki/Memory_segmentation)
* [Model specific registers](http://en.wikipedia.org/wiki/Model-specific_register)
* [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries)
* [Previous chapter](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/index.html)
