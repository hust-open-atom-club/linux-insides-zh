中断和中断处理 Part 1.
================================================================================

Introduction
--------------------------------------------------------------------------------

这是 [linux 内核揭密](http://0xax.gitbooks.io/linux-insides/content/) 新章节的第一部分。我们已经在这本书前面的[章节](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)中走过了漫长的道路。开始于内核初始化的[第一步](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html)，结束于[启动](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-10.html)第一个 `init` 程序 。我们见证了一系列与各种内核子系统相关的初始化步骤，但是我们并没有深入这些子系统。在这一章节中，我们将会试着去了解这些内核子系统是如何工作和实现的。就像你在这章标题中看到的，第一个子系统是[中断](http://en.wikipedia.org/wiki/Interrupt)。

什么是中断？
--------------------------------------------------------------------------------

我们已经在这本书的很多地方听到过 `中断（interrupts）` 这个词，也看到过很多关于中断的例子。在这一节中我们将会从下面的主题开始：

* 什么是 `中断` ？
* 什么是 `中断处理`？

我们将会继续深入探讨 `中断` 的细节和 Linux 内核如何处理他们。

所以，首先什么是中断？中断就是当软件或者硬件需要使用 CPU 时引发的 `事件（event）` 。比如，当我们在键盘上按下一个键的时候，我们下一步期望做什么？操作系统和电脑应该怎么做？做一个简单的假设，每一个物理硬件都有一根连接 CPU 的中断线，设备可以通过它对 CPU 发起中断信号。但是中断并不是直接通知给 CPU。在老机器上中断通知给 [PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller) ，它是一个顺序处理各种设备的各种中断请求的芯片。在新机器上，则是[高级程序中断控制器（Advanced Programmable Interrupt Controller）](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)，即我们熟知的 `APIC`。一个 APIC 包括两个独立的设备：

* `Local APIC`
* `I/O APIC`

第一个设备 -  `Local APIC ` 存在于每个CPU核心中，Local APIC 负责处理 CPU-specific 的中断配置。Local APIC 常被用于管理来自 APIC 时钟（APIC-timer），热敏元件和其他与 I/O 设备连接的设备的中断。

第二个设备 -  `I/O APIC` 提供了多核处理器的中断管理。它被用来在所有的 CPU 核心中分发外部中断。更多关于 local 和 I/O APIC 的内容将会在这一节的下面讲到。就如你所知道的，中断可以在任何时间发生。当一个中断发生时，操作系统必须立刻处理它。但是 `处理一个中断` 是什么意思呢？当一个中断发生时，操作系统必须确保下面的步骤：

* 内核必须暂停执行当前进程(取代当前的任务)；
* 内核必须搜索中断处理程序并且转交控制权(执行中断处理程序)；
* 中断处理程序结束之后，被中断的进程能够恢复执行。

当然，在这个中断处理程序中会涉及到很多错综复杂的过程。但是上面 3 条是这个程序的基本骨架。

每个中断处理程序的地址都保存在一个特殊的位置，这个位置被称为 `中断描述符表（Interrupt Descriptor Table）` 或者 `IDT`。处理器使用一个唯一的数字来识别中断和异常的类型，这个数字被称为 `中断标识码（vector number）`。一个中断标识码就是一个 `IDT` 的标识。中断标识码范围是有限的，从 `0` 到 `255`。你可以在 Linux 内核源码中找到下面的中断标识码范围检查代码：

```C
BUG_ON((unsigned)n > 0xFF);
```

你可以在 Linux 内核源码中关于中断设置的地方找到这个检查(例如：`set_intr_gate`, `void set_system_intr_gate` 在 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h)中)。最开始从 `0` 到 `31` 的 32 个中断标识码被处理器保留，用作处理架构定义的异常和中断。你可以在 Linux 内核初始化程序的第二部分 - [早期中断和异常处理](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html)中找到这个表和关于这些中断标识码的描述。从 `32` 到 `255` 的中断标识码设计为用户定义中断并且不被系统保留。这些中断通常分配给外部 I/O 设备，使这些设备可以发送中断给处理器。

现在，我们来讨论中断的类型。笼统地来讲，我们可以把中断分为两个主要类型：

* 外部或者硬件引起的中断；
* 软件引起的中断。

第一种类型 - 外部中断，由 `Local APIC` 或者与 `Local APIC` 连接的处理器针脚接收。第二种类型 - 软件引起的中断，由处理器特殊的情况引起(有时使用特殊架构的指令)。一个常见的关于特殊情况的例子就是 `除零`。另一个例子就是使用 `系统调用（syscall）` 退出程序。

就如之前提到过的，中断可以在任何时间因为超出代码和 CPU 控制的原因而发生。另一方面，异常和程序执行 `同步（synchronous）` ，并且可以被分为 3 类：

* `故障（Faults）`
* `陷入（Traps）`
* `终止（Aborts）`

`故障` 是在执行一个“不完善的”指令（可以在之后被修正）之前被报告的异常。如果发生了，它允许被中断的程序继续执行。

接下来的 `陷入` 是一个在执行了 `陷入` 指令后立刻被报告的异常。陷入同样允许被中断的程序继续执行，就像 `故障` 一样。

最后的 `终止` 是一个从不报告引起异常的精确指令的异常，并且不允许被中断的程序继续执行。

我们已经从前面的[部分](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html)知道，中断可以分为 `可屏蔽的（maskable）` 和 `不可屏蔽的（non-maskable）`。可屏蔽的中断可以被阻塞，使用 `x86_64` 的指令 - `sti` 和 `cli`。我们可以在 Linux 内核代码中找到他们：

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

现在我们了解了一些关于各种类型的中断和异常的内容，是时候转到更实用的部分了。我们从 `中断描述符表` 开始。就如之前所提到的，`IDT` 保存了中断和异常处理程序的入口指针。`IDT` 是一个类似于 `全局描述符表（Global Descriptor Table）`的结构，我们在[内核启动程序](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html)的第二部分已经介绍过。但是他们确实有一些不同，`IDT` 的表项被称为 `门（gates）`，而不是 `描述符（descriptors）`。


* 中断门（Interrupt gates）
* 任务门（Task gates）
* 陷阱门（Trap gates）

在 `x86` 架构中，只有 [long mode](http://en.wikipedia.org/wiki/Long_mode) 中断门和陷阱中断门可以在 `x86_64` 中引用。就像 `全局描述符表`，`中断描述符表` 在 `x86` 上是一个 8 字节数组门，而在 `x86_64` 上是一个16字节数组门。让我们回忆在[内核启动程序](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html)的第二部分，`全局描述符表` 必须包含 `NULL` 描述符作为它的第一个元素。与 `全局描述符表` 不一样的是，`中断描述符表` 的第一个元素可以是一个门。它并不是强制要求的。比如，你可能还记得我们只是在早期的章节中过渡到[保护模式](http://en.wikipedia.org/wiki/Protected_mode)时用 `NULL` 门加载过中断描述符表：

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

在 [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pm.c)中。`中断描述符表` 可以在线性地址空间和基址的任何地方被加载，只要在 `x86` 上以8字节对齐，在 `x86_64` 上以16字节对齐。`IDT` 的基址存储在一个特殊的寄存器 - IDTR。在 `x86` 上有两个指令 - 协调工作来修改 `IDTR` 寄存器： 

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

To form an index into the IDT, the processor scales the exception or interrupt vector by sixteen. The processor handles the occurrence of exceptions and interrupts just like it handles calls of a procedure when it sees the `call` instruction. A processor uses an unique number or `vector number` of the interrupt or the exception as the index to find the necessary `Interrupt Descriptor Table` entry. Now let's take a closer look at an `IDT` entry.

As we can see, `IDT` entry on the diagram consists of the following fields:

* `0-15` bits  - offset from the segment selector which is used by the processor as the base address of the entry point of the interrupt handler;
* `16-31` bits - base address of the segment select which contains the entry point of the interrupt handler;
* `IST` - a new special mechanism in the `x86_64`, will see it later;
* `DPL` - Descriptor Privilege Level;
* `P` - Segment Present flag;
* `48-63` bits - second part of the handler base address;
* `64-95` bits - third part of the base address of the handler;
* `96-127` bits - and the last bits are reserved by the CPU.

And the last `Type` field describes the type of the `IDT` entry. There are three different kinds of handlers for interrupts:

* Interrupt gate
* Trap gate
* Task gate

The `IST` or `Interrupt Stack Table` is a new mechanism in the `x86_64`. It is used as an alternative to the the legacy stack-switch mechanism. Previously The `x86` architecture provided a mechanism to automatically switch stack frames in response to an interrupt. The `IST` is a modified version of the `x86` Stack switching mode. This mechanism unconditionally switches stacks when it is enabled and can be enabled for any interrupt in the `IDT` entry related with the certain interrupt (we will soon see it). From this we can understand that `IST` is not necessary for all interrupts. Some interrupts can continue to use the legacy stack switching mode. The `IST` mechanism provides up to seven `IST` pointers in the [Task State Segment](http://en.wikipedia.org/wiki/Task_state_segment) or `TSS` which is the special structure which contains information about a process. The `TSS` is used for stack switching during the execution of an interrupt or exception handler in the Linux kernel. Each pointer is referenced by an interrupt gate from the `IDT`.

The `Interrupt Descriptor Table` represented by the array of the `gate_desc` structures:

```C
extern gate_desc idt_table[];
```

where `gate_desc` is:

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

and `gate_struct64` defined as:

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

Each active thread has a large stack in the Linux kernel for the `x86_64` architecture. The stack size is defined as `THREAD_SIZE` and is equal to:

```C
#define PAGE_SHIFT      12
#define PAGE_SIZE       (_AC(1,UL) << PAGE_SHIFT)
...
...
...
#define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

The `PAGE_SIZE` is `4096`-bytes and the `THREAD_SIZE_ORDER` depends on the `KASAN_STACK_ORDER`. As we can see, the `KASAN_STACK` depends on the `CONFIG_KASAN` kernel configuration parameter and is defined as:

```C
#ifdef CONFIG_KASAN
    #define KASAN_STACK_ORDER 1
#else
    #define KASAN_STACK_ORDER 0
#endif
```

`KASan` is a runtime memory [debugger](http://lwn.net/Articles/618180/). So... the `THREAD_SIZE` will be `16384` bytes if `CONFIG_KASAN` is disabled or `32768` if this kernel configuration option is enabled. These stacks contain useful data as long as a thread is alive or in a zombie state. While the thread is in user-space, the kernel stack is empty except for the `thread_info` structure (details about this structure are available in the fourth [part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html) of the Linux kernel initialization process) at the bottom of the stack. The active or zombie threads aren't the only threads with their own stack. There also exist specialized stacks that are associated with each available CPU. These stacks are active when the kernel is executing on that CPU. When the user-space is executing on the CPU, these stacks do not contain any useful information. Each CPU has a few special per-cpu stacks as well. The first is the `interrupt stack` used for the external hardware interrupts. Its size is determined as follows:

```C
#define IRQ_STACK_ORDER (2 + KASAN_STACK_ORDER)
#define IRQ_STACK_SIZE (PAGE_SIZE << IRQ_STACK_ORDER)
```

or `16384` bytes. The per-cpu interrupt stack represented by the `irq_stack_union` union in the Linux kernel for `x86_64`:

```C
union irq_stack_union {
	char irq_stack[IRQ_STACK_SIZE];

    struct {
		char gs_base[40];
		unsigned long stack_canary;
	};
};
```

The first `irq_stack` field is a 16 kilobytes array. Also you can see that `irq_stack_union` contains a structure with the two fields:

* `gs_base` - The `gs` register always points to the bottom of the `irqstack` union. On the `x86_64`, the `gs` register is shared by per-cpu area and stack canary (more about `per-cpu` variables you can read in the special [part](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)).  All per-cpu symbols are zero based and the `gs` points to the base of the per-cpu area. You already know that [segmented memory model](http://en.wikipedia.org/wiki/Memory_segmentation) is abolished in the long mode, but we can set the base address for the two segment registers - `fs` and `gs` with the [Model specific registers](http://en.wikipedia.org/wiki/Model-specific_register) and these registers can be still be used as address registers. If you remember the first [part](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html) of the Linux kernel initialization process, you can remember that we have set the `gs` register:

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr
```

where `initial_gs` points to the `irq_stack_union`:

```assembly
GLOBAL(initial_gs)
.quad	INIT_PER_CPU_VAR(irq_stack_union)
```

* `stack_canary` - [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries) for the interrupt stack is a `stack protector`
to verify that the stack hasn't been overwritten. Note that `gs_base` is a 40 bytes array. `GCC` requires that stack canary will be on the fixed offset from the base of the `gs` and its value must be `40` for the `x86_64` and `20` for the `x86`.

The `irq_stack_union` is the first datum in the `percpu` area, we can see it in the `System.map`:

```
0000000000000000 D __per_cpu_start
0000000000000000 D irq_stack_union
0000000000004000 d exception_stacks
0000000000009000 D gdt_page
...
...
...
```

We can see its definition in the code:

```C
DECLARE_PER_CPU_FIRST(union irq_stack_union, irq_stack_union) __visible;
```

Now, it's time to look at the initialization of the `irq_stack_union`. Besides the `irq_stack_union` definition, we can see the definition of the following per-cpu variables in the [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/processor.h):

```C
DECLARE_PER_CPU(char *, irq_stack_ptr);
DECLARE_PER_CPU(unsigned int, irq_count);
```

The first is the `irq_stack_ptr`. From the variable's name, it is obvious that this is a pointer to the top of the stack. The second - `irq_count` is used to check if a CPU is already on an interrupt stack or not. Initialization of the `irq_stack_ptr` is located in the `setup_per_cpu_areas` function in [arch/x86/kernel/setup_percpu.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup_percpu.c):

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

Here we go over all the CPUs one-by-one and setup `irq_stack_ptr`. This turns out to be equal to the top of the interrupt stack minus `64`. Why `64`?TODO  [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpu/common.c) source code file is following:

```C
void load_percpu_segment(int cpu)
{
        ...
        ...
        ...
        loadsegment(gs, 0);
        wrmsrl(MSR_GS_BASE, (unsigned long)per_cpu(irq_stack_union.gs_base, cpu));
}
```

and as we already know the `gs` register points to the bottom of the interrupt stack:

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr

	GLOBAL(initial_gs)
	.quad	INIT_PER_CPU_VAR(irq_stack_union)
```

Here we can see the `wrmsr` instruction which loads the data from `edx:eax` into the [Model specific register](http://en.wikipedia.org/wiki/Model-specific_register) pointed by the `ecx` register. In our case the model specific register is `MSR_GS_BASE` which contains the base address of the memory segment pointed by the `gs` register. `edx:eax` points to the address of the `initial_gs` which is the base address of our `irq_stack_union`.

We already know that `x86_64` has a feature called `Interrupt Stack Table` or `IST` and this feature provides the ability to switch to a new stack for events non-maskable interrupt, double fault and etc... There can be up to seven `IST` entries per-cpu. Some of them are:

* `DOUBLEFAULT_STACK`
* `NMI_STACK`
* `DEBUG_STACK`
* `MCE_STACK`

or

```C
#define DOUBLEFAULT_STACK 1
#define NMI_STACK 2
#define DEBUG_STACK 3
#define MCE_STACK 4
```

All interrupt-gate descriptors which switch to a new stack with the `IST` are initialized with the `set_intr_gate_ist` function. For example:

```C
set_intr_gate_ist(X86_TRAP_NMI, &nmi, NMI_STACK);
...
...
...
set_intr_gate_ist(X86_TRAP_DF, &double_fault, DOUBLEFAULT_STACK);
```

where `&nmi` and `&double_fault` are addresses of the entries to the given interrupt handlers:

```C
asmlinkage void nmi(void);
asmlinkage void double_fault(void);
```

defined in the [arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/entry_64.S)

```assembly
idtentry double_fault do_double_fault has_error_code=1 paranoid=2
...
...
...
ENTRY(nmi)
...
...
...
END(nmi)
```

When an interrupt or an exception occurs, the new `ss` selector is forced to `NULL` and the `ss` selector’s `rpl` field is set to the new `cpl`. The old `ss`, `rsp`, register flags, `cs`, `rip` are pushed onto the new stack. In 64-bit mode, the size of interrupt stack-frame pushes is fixed at 8-bytes, so we will get the following stack:

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

If the `IST` field in the interrupt gate is not `0`, we read the `IST` pointer into `rsp`. If the interrupt vector number has an error code associated with it, we then push the error code onto the stack. If the interrupt vector number has no error code, we go ahead and push the dummy error code on to the stack. We need to do this to ensure stack consistency. Next we load the segment-selector field from the gate descriptor into the CS register and must verify that the target code-segment is a 64-bit mode code segment by the checking bit `21` i.e. the `L` bit in the `Global Descriptor Table`. Finally we load the offset field from the gate descriptor into `rip` which will be the entry-point of the interrupt handler. After this the interrupt handler begins to execute. After an interrupt handler finishes its execution, it must return control to the interrupted process with the `iret` instruction. The `iret` instruction unconditionally pops the stack pointer (`ss:rsp`) to restore the stack of the interrupted process and does not depend on the `cpl` change.

That's all.

Conclusion
--------------------------------------------------------------------------------

It is the end of the first part about interrupts and interrupt handling in the Linux kernel. We saw some theory and the first steps of the initialization of stuff related to interrupts and exceptions. In the next part we will continue to dive into interrupts and interrupts handling - into the more practical aspects of it.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me a PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
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
* [Previous chapter](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)
