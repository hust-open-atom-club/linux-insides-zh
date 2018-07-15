中断和中断处理 Part 2.
================================================================================

深入Linux内核中的中断和异常处理
--------------------------------------------------------------------------------

在 [上一章节](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html)中我们学习了中断和异常处理的一些理论知识，在本章节中，我们将深入了解Linux内核源代码中关于中断与异常处理的部分。之前的章节中主要从理论方面描述了Linux中断和异常处理的相关内容，而在本章节中，我们将直接深入Linux源代码来了解相关内容。像其他章节一样，我们将从启动早期的代码开始阅读。本章将不会像 [Linux内核启动过程](http://0xax.gitbooks.io/linux-insides/content/Booting/index.html)中那样从Linux内核启动的 [最开始](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L292)几行代码读起，而是从与中断与异常处理相关的最早期代码开始阅读，了解Linux内核源代码中所有与中断和异常处理相关的代码。

如果你读过本书的前面部分，你可能记得Linux内核中关于 `x86_64`架构的代码中与中断相关的最早期代码出现在 [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pm.c)文件中，该文件首次配置了 [中断描述符表](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)(IDT)。对IDT的配置在`go_to_protected_mode`函数中完成，该函数首先调用了 `setup_idt`函数配置了IDT，然后将处理器的工作模式切换为 [保护模式](http://en.wikipedia.org/wiki/Protected_mode):

```C
void go_to_protected_mode(void)
{
        ...
        setup_idt();
        ...
}
```

`setup_idt`函数在同一文件中定义，它仅仅是用 `NULL`填充了中断描述符表:

```C
static void setup_idt(void)
{
        static const struct gdt_ptr null_idt = {0, 0};
        asm volatile("lidtl %0" : : "m" (null_idt));
}
```

其中，`gdt_ptr`表示了一个48-bit的特殊功能寄存器 `GDTR`，其包含了全局描述符表 `Global Descriptor`的基地址:

```C
struct gdt_ptr {
        u16 len;
        u32 ptr;
} __attribute__((packed));
```

显然，在此处的 `gdt_prt`不是代表 `GDTR`寄存器而是代表 `IDTR`寄存器，因为我们将其设置到了中断描述符表中。之所以在Linux内核代码中没有`idt_ptr`结构体，是因为其与`gdt_prt`具有相同的结构而仅仅是名字不同，因此没必要定义两个重复的数据结构。可以看到，内核在此处并没有填充`Interrupt Descriptor Table`，这是因为此刻处理任何中断或异常还为时尚早，因此我们仅仅以`NULL`来填充`IDT`。

在设置完 [Interrupt descriptor table](http://en.wikipedia.org/wiki/Interrupt_descriptor_table), [Global Descriptor Table](http://en.wikipedia.org/wiki/GDT)和其他一些东西以后，内核开始进入保护模式，这部分代码在 [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pmjump.S)中实现，你可以在描述如何进入保护模式的 [章节](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html)中了解到更多细节。

在最早的章节中我们已经了解到进入保护模式的代码位于 `boot_params.hdr.code32_start`，你可以在 [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pm.c)的末尾看到内核将入口函数指针和启动参数 `boot_params`传递给了 `protected_mode_jump`函数:

```C
protected_mode_jump(boot_params.hdr.code32_start,
                            (u32)&boot_params + (ds() << 4));
```

定义在文件 [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pmjump.S)中的函数`protected_mode_jump`通过一种[8086](http://en.wikipedia.org/wiki/Intel_8086)的调用 [约定](http://en.wikipedia.org/wiki/X86_calling_conventions#List_of_x86_calling_conventions)，通过 `ax`和 `dx`两个寄存器来获取参数:

```assembly
GLOBAL(protected_mode_jump)
        ...
        ...
        ...
        .byte   0x66, 0xea              # ljmpl opcode
2:      .long   in_pm32                 # offset
        .word   __BOOT_CS               # segment
...
...
...
ENDPROC(protected_mode_jump)
```

其中 `in_pm32`包含了对32-bit入口的跳转语句:

```assembly
GLOBAL(in_pm32)
        ...
        ...
        jmpl    *%eax // %eax contains address of the `startup_32`
        ...
        ...
ENDPROC(in_pm32)
```

你可能还记得32-bit的入口地址位于汇编文件 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S)中，尽管它的名字包含 `_64`后缀。我们可以在 `arch/x86/boot/compressed`目录下看到两个相似的文件:

* `arch/x86/boot/compressed/head_32.S`.
* `arch/x86/boot/compressed/head_64.S`;

然而32-bit模式的入口位于第二个文件中，而第一个文件在 `x86_64`配置下不会参与编译。如 [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/Makefile):

```
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
...
...
```

代码中的 `head_*`取决于 `$(BITS)` 变量的值，而该值由"架构"决定。我们可以在 [arch/x86/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Makefile)找到相关信息:

```
ifeq ($(CONFIG_X86_32),y)
...
        BITS := 32
else
        BITS := 64
        ...
endif
```

现在我们从 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S)跳入了 `startup_32`函数，在这个函数中没有与中断处理相关的内容。`startup_32`函数包含了进入 [long mode](http://en.wikipedia.org/wiki/Long_mode)之前必须的准备工作，并直接进入了 `long mode`。 `long mode`的入口位于 `startup_64`函数中，在这个函数中完成了 [内核解压](http://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-5.html)的准备工作。内核解压的代码位于 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/misc.c)中的 `decompress_kernel`函数中。内核解压完成以后，程序跳入 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S)中的 `startup_64`函数。在这个函数中，我们开始构建 `identity-mapped pages`，并在之后检查 [NX](http://en.wikipedia.org/wiki/NX_bit)位，配置 `Extended Feature Enable Register`(见链接)，使用 `lgdt`指令更新早期的`Global Descriptor Table`，在此之后我们还需要使用如下代码来设置 `gs`寄存器:

```assembly
movl    $MSR_GS_BASE,%ecx
movl    initial_gs(%rip),%eax
movl    initial_gs+4(%rip),%edx
wrmsr
```

这段代码在之前的 [章节](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html)中也出现过。请注意代码最后的 `wrmsr`指令，这个指令将 `edx:eax`寄存器指定的地址中的数据写入到由 `ecx`寄存器指定的 [model specific register](http://en.wikipedia.org/wiki/Model-specific_register)中。由代码可以看到，`ecx`中的值是 `$MSR_GS_BASE`，该值在 [arch/x86/include/uapi/asm/msr-index.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/uapi/asm/msr-index.h)中定义:

```C
#define MSR_GS_BASE             0xc0000101
```

由此可见，`MSR_GS_BASE`定义了 `model specific register`的编号。由于 `cs`, `ds`, `es`,和 `ss`在64-bit模式中不再使用，这些寄存器中的值将会被忽略，但我们可以通过 `fs`和 `gs`寄存器来访问内存空间。`model specific register`提供了一种后门 `back door`来访问这些段寄存器，也让我们可以通过段寄存器 `fs`和 `gs`来访问64-bit的基地址。看起来这部分代码映射在 `GS.base`域中。再看到 `initial_gs`函数的定义:

```assembly
GLOBAL(initial_gs)
        .quad   INIT_PER_CPU_VAR(irq_stack_union)
```

这段代码将 `irq_stack_union`传递给 `INIT_PER_CPU_VAR`宏，后者只是给输入参数添加了 `init_per_cpu__`前缀而已。在此得出了符号 `init_per_cpu__irq_stack_union`。再看到 [链接脚本](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vmlinux.lds.S)，其中可以看到如下定义:

```
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(irq_stack_union);
```

这段代码告诉我们符号 `init_per_cpu__irq_stack_union`的地址将会是 `irq_stack_union + __per_cpu_load`。现在再来看看 `init_per_cpu__irq_stack_union`和 `__per_cpu_load`在哪里。`irq_stack_union`的定义出现在 [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h)中，其中的 `DECLARE_INIT_PER_CPU`宏展开后又调用了 `init_per_cpu_var`宏:

```C
DECLARE_INIT_PER_CPU(irq_stack_union);

#define DECLARE_INIT_PER_CPU(var) \
       extern typeof(per_cpu_var(var)) init_per_cpu_var(var)

#define init_per_cpu_var(var)  init_per_cpu__##var
```

将所有的宏展开之后我们可以得到与之前相同的名称 `init_per_cpu__irq_stack_union`，但此时它不再只是一个符号，而成了一个变量。请注意表达式 `typeof(per_cpu_var(var))`,在此时 `var`是 `irq_stack_union`，而 `per_cpu_var`宏在 [arch/x86/include/asm/percpu.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/percpu.h)中定义:

```C
#define PER_CPU_VAR(var)        %__percpu_seg:var
```

其中:

```C
#ifdef CONFIG_X86_64
    #define __percpu_seg gs
endif
```

因此，我们实际访问的是 `gs:irq_stack_union`，它的类型是 `irq_union`。到此为止，我们定义了上面所说的第一个变量并且知道了它的地址。再看到第二个符号 `__per_cpu_load`，该符号定义在 [include/asm-generic/sections.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/asm-generic-sections.h)，这个符号定义了一系列 `per-cpu`变量:

```C
extern char __per_cpu_load[], __per_cpu_start[], __per_cpu_end[];
```

同时，符号代表了这一系列变量的数据区域的基地址。因此我们知道了 `irq_stack_union`和 `__per_cpu_load`的地址，并且知道变量 `init_per_cpu__irq_stack_union`位于 `__per_cpu_load`。并且看到 [System.map](http://en.wikipedia.org/wiki/System.map):

```
...
...
...
ffffffff819ed000 D __init_begin
ffffffff819ed000 D __per_cpu_load
ffffffff819ed000 A init_per_cpu__irq_stack_union
...
...
...
```

现在我们终于知道了 `initial_gs`是什么，回到之前的代码中:

```assembly
movl    $MSR_GS_BASE,%ecx
movl    initial_gs(%rip),%eax
movl    initial_gs+4(%rip),%edx
wrmsr
```

此时我们通过 `MSR_GS_BASE`指定了一个平台相关寄存器，然后将 `initial_gs`的64-bit地址放到了 `edx:eax`段寄存器中，然后执行 `wrmsr`指令，将 `init_per_cpu__irq_stack_union`的基地址放入了 `gs`寄存器，而这个地址将是中断栈的栈底地址。在此之后我们将进入 `x86_64_start_kernel`函数的C语言代码中，此函数定义在 [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head64.c)。在这个函数中，我们将完成最后的准备工作，之后就要进入到与平台无关的通用内核代码。如果你读过前文的 [早期中断和异常处理](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html)章节，你可能记得其中之一的工作就是将中断服务程序入口地址填写到早期 `Interrupt Descriptor Table`中。

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
        set_intr_gate(i, early_idt_handlers[i]);

load_idt((const struct desc_ptr *)&idt_descr);
```

当我写 `早期中断和异常处理`章节时Linux内核版本是 `3.18`，而如今Linux内核版本已经生长到了 `4.1.0-rc6+`，并且 `Andy Lutomirski`提交了一个与 `early_idt_handlers`相关的修改 [patch](https://lkml.org/lkml/2015/6/2/106)，该修改即将并入内核代码主线中。**NOTE**在我写这一段时，这个 [patch](https://github.com/torvalds/linux/commit/425be5679fd292a3c36cb1fe423086708a99f11a)已经进入了Linux内核源代码中。现在这段代码变成了:

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
        set_intr_gate(i, early_idt_handler_array[i]);

load_idt((const struct desc_ptr *)&idt_descr);
```
如你所见，这段代码与之前相比唯一的区别在于中断服务程序入口点数组的名称现在改为了 `early_idt_handler_array`:

```C
extern const char early_idt_handler_array[NUM_EXCEPTION_VECTORS][EARLY_IDT_HANDLER_SIZE];
```

其中 `NUM_EXCEPTION_VECTORS` 和 `EARLY_IDT_HANDLER_SIZE` 的定义如下:

```C
#define NUM_EXCEPTION_VECTORS 32
#define EARLY_IDT_HANDLER_SIZE 9
```

因此，数组 `early_idt_handler_array` 存放着中断服务程序入口，其中每个入口占据9个字节。`early_idt_handlers` 定义在文件[arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S)中。`early_idt_handler_array` 也定义在这个文件中:

```assembly
ENTRY(early_idt_handler_array)
...
...
...
ENDPROC(early_idt_handler_common)
```

这里使用 `.rept NUM_EXCEPTION_VECTORS` 填充了 `early_idt_handler_array` ，其中也包含了 `early_make_pgtable` 的中断服务函数入口(关于该中断服务函数的实现请参考章节 [早期的中断和异常控制](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html))。现在我们完成了所有`x86-64`平台相关的代码，即将进入通用内核代码中。当然，我们之后还会在 `setup_arch` 函数中重新回到平台相关代码，但这已经是 `x86_64` 平台早期代码的最后部分。

为中断堆栈设置`Stack Canary`值
-------------------------------------------------------------------------------

正如之前阅读过的关于Linux内核初始化过程的[章节](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)，在[arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S)之后的下一步进入到了[init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)中的函数体最大的函数 `start_kernel` 中。这个函数将完成内核以[pid](https://en.wikipedia.org/wiki/Process_identifier) - `1`运行第一个`init`进程
之前的所有初始化工作。其中，与中断和异常处理相关的第一件事是调用 `boot_init_stack_canary` 函数。这个函数通过设置[canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries)值来防止中断栈溢出。前面我们已经看过了 `boot_init_stack_canary` 实现的一些细节，现在我们更进一步地认识它。你可以在[arch/x86/include/asm/stackprotector.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/stackprotector.h)中找到这个函数的实现，它的实现取决于 `CONFIG_CC_STACKPROTECTOR` 这个内核配置选项。如果该选项没有置位，那该函数将是一个空函数:

```C
#ifdef CONFIG_CC_STACKPROTECTOR
...
...
...
#else
static inline void boot_init_stack_canary(void)
{
}
#endif
```

如果设置了内核配置选项 `CONFIG_CC_STACKPROTECTOR` ，那么函数`boot_init_stack_canary` 一开始将检查联合体 `irq_stack_union` 的状态，这个联合体代表了[per-cpu](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)中断栈，其与 `stack_canary` 值中间有40个字节的 `offset` :

```C
#ifdef CONFIG_X86_64
        BUILD_BUG_ON(offsetof(union irq_stack_union, stack_canary) != 40);
#endif
```

如之前[章节](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html)所描述， `irq_stack_union` 联合体的定义如下:

```C
union irq_stack_union {
        char irq_stack[IRQ_STACK_SIZE];

    struct {
                char gs_base[40];
                unsigned long stack_canary;
        };
};
```

以上定义位于文件[arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h)。总所周知，[C语言](http://en.wikipedia.org/wiki/C_%28programming_language%29)中的[联合体](http://en.wikipedia.org/wiki/Union_type)是一种描述多个数据结构共用一片内存的数据结构。可以看到，第一个数据域 `gs_base` 大小为40 bytes，代表了 `irq_stack` 的栈底。因此，当我们使用 `BUILD_BUG_ON` 对该表达式进行检查时结果应为成功。(关于 `BUILD_BUG_ON` 宏的详细信息可见[Linux内核初始化过程章节](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html))。

紧接着我们使用随机数和[时戳计数器](http://en.wikipedia.org/wiki/Time_Stamp_Counter)计算新的 `canary` 值:

```C
get_random_bytes(&canary, sizeof(canary));
tsc = __native_read_tsc();
canary += tsc + (tsc << 32UL);
```
并且通过 `this_cpu_write` 宏将 `canary` 值写入了 `irq_stack_union` 中:

```C
this_cpu_write(irq_stack_union.stack_canary, canary);
```

关于  `this_cpu_*` 系列宏的更多信息参见[Linux kernel documentation](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/this_cpu_ops.txt)。

禁用/使能本地中断
--------------------------------------------------------------------------------
在 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 中，与中断和中断处理相关的操作中，设置的 `canary` 的下一步是调用 `local_irq_disable` 宏。

这个宏定义在头文件 [include/linux/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irqflags.h) 中，宏如其名，调用这个宏将禁用本地CPU的中断。我们来仔细了解一下这个宏的实现，首先，它依赖于内核配置选项 `CONFIG_TRACE_IRQFLAGS_SUPPORT` :

```C
#ifdef CONFIG_TRACE_IRQFLAGS_SUPPORT
...
#define local_irq_disable() \
         do { raw_local_irq_disable(); trace_hardirqs_off(); } while (0)
...
#else
...
#define local_irq_disable()     do { raw_local_irq_disable(); } while (0)
...
#endif
```

如你所见，两者唯一的区别在于当 `CONFIG_TRACE_IRQFLAGS_SUPPORT` 选项使能时， `local_irq_disable` 宏将同时调用 `trace_hardirqs_off` 函数。在Linux死锁检测模块[lockdep](http://lwn.net/Articles/321663/)中有一项功能 `irq-flags tracing` 可以追踪 `hardirq` 和 `softirq` 的状态。在这种情况下， `lockdep` 死锁检测模块可以提供系统中关于硬/软中断的开/关事件的相关信息。函数 `trace_hardirqs_off` 的定义位于[kernel/locking/lockdep.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/locking/lockdep.c):

```C
void trace_hardirqs_off(void)
{
         trace_hardirqs_off_caller(CALLER_ADDR0);
}
EXPORT_SYMBOL(trace_hardirqs_off);
```

可见它只是调用了 `trace_hardirqs_off_caller` 函数。 `trace_hardirqs_off_caller` 函数,该函数检查了当前进程的 `hardirqs_enabled` 域，如果本次 `local_irq_disable` 调用是冗余的话，便使 `redundant_hardirqs_off` 域的值增长，否则便使 `hardirqs_off_events` 域的值增加。这两个域或其它与死锁检测模块 `lockdep` 统计相关的域定义在文件[kernel/locking/lockdep_insides.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/locking/lockdep_insides.h)中的 `lockdep_stats` 结构体中:

```C
struct lockdep_stats {
...
...
...
int     softirqs_off_events;
int     redundant_softirqs_off;
...
...
...
}
```

如果你使能了 `CONFIG_DEBUG_LOCKDEP` 内核配置选项，`lockdep_stats_debug_show`函数会将所有的调试信息写入 `/proc/lockdep` 文件中:

```C
static void lockdep_stats_debug_show(struct seq_file *m)
{
#ifdef CONFIG_DEBUG_LOCKDEP
    unsigned long long hi1 = debug_atomic_read(hardirqs_on_events),
        hi2 = debug_atomic_read(hardirqs_off_events),
        hr1 = debug_atomic_read(redundant_hardirqs_on),
        ...
    ...
    ...
    seq_printf(m, " hardirq on events:             %11llu\n", hi1);
    seq_printf(m, " hardirq off events:            %11llu\n", hi2);
    seq_printf(m, " redundant hardirq ons:         %11llu\n", hr1);
#endif
}
```

你可以如下命令查看其内容:

```
$ sudo cat /proc/lockdep
 hardirq on events:             12838248974
 hardirq off events:            12838248979
 redundant hardirq ons:               67792
 redundant hardirq offs:         3836339146
 softirq on events:                38002159
 softirq off events:               38002187
 redundant softirq ons:                   0
 redundant softirq offs:                  0
```

现在我们总算了解了调试函数 `trace_hardirqs_off` 的一些信息，下文将有独立的章节介绍 `lockdep` 和 `trancing`。`local_disable_irq` 宏的实现中都包含了一个宏 `raw_local_irq_disable` ，这个定义在 [arch/x86/include/asm/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irqflags.h) 中，其展开后的样子是:

```C
static inline void native_irq_disable(void)
{
        asm volatile("cli": : :"memory");
}
```

你可能还记得， `cli` 指令将清除[IF](http://en.wikipedia.org/wiki/Interrupt_flag) 标志位，这个标志位控制着处理器是否响应中断或异常。与 `local_irq_disable` 相对的还有宏 `local_irq_enable` ，这个宏的实现与 `local_irq_disable` 很相似，也具有相同的调试机制，区别在于使用 `sti` 指令使能了中断:

```C
static inline void native_irq_enable(void)
{
        asm volatile("sti": : :"memory");
}
```

如今我们了解了 `local_irq_disable` 和 `local_irq_enable` 宏的实现机理。此处是首次调用 `local_irq_disable` 宏，我们还将在Linux内核源代码中多次看到它的倩影。现在我们位于 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 中的 `start_kernel` 函数，并且刚刚禁用了`本地`中断。为什么叫"本地"中断？为什么要禁用本地中断呢？早期版本的内核中提供了一个叫做 `cli` 的函数来禁用所有处理器的中断，该函数已经被[移除](https://lwn.net/Articles/291956/)，替代它的是 `local_irq_{enabled,disable}` 宏，用于禁用或使能当前处理器的中断。我们在调用 `local_irq_disable` 宏禁用中断以后，接着设置了变量值:

```C
early_boot_irqs_disabled = true;
```

变量 `early_boot_irqs_disabled` 定义在文件 [include/linux/kernel.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/kernel.h) 中:

```C
extern bool early_boot_irqs_disabled;
```

并在另外的地方使用。例如在 [kernel/smp.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/smp.c) 中的 `smp_call_function_many` 函数中，通过这个变量来检查当前是否由于中断禁用而处于死锁状态:

```C
WARN_ON_ONCE(cpu_online(this_cpu) && irqs_disabled()
                     && !oops_in_progress && !early_boot_irqs_disabled);
```

内核初始化过程中的早期 `trap` 初始化
--------------------------------------------------------------------------------

在 `local_disable_irq` 之后执行的函数是 `boot_cpu_init` 和 `page_address_init`，但这两个函数与中断和异常处理无关(更多与这两个函数有关的信息请阅读内核初始化过程[章节](http://0xax.gitbooks.io/linux-insides/content/Initialization/index.html))。接下来是 `setup_arch` 函数。你可能还有印象，这个函数定义在[arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel.setup.c) 文件中，并完成了很多[架构相关的初始化工作](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html)。在 `setup_arch` 函数中与中断相关的第一个函数是 `early_trap_init` 函数，该函数定义于 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c) ，其用许多对程序入口填充了中断描述符表 `Interrupt Descriptor Table` :

```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
#ifdef CONFIG_X86_32
        set_intr_gate(X86_TRAP_PF, page_fault);
#endif
        load_idt(&idt_descr);
}
```

这里出现了三个不同的函数调用

* `set_intr_gate_ist`
* `set_system_intr_gate_ist`
* `set_intr_gate`

这些函数都定义在 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/desc.h) 中，他们做的事情也差不多。第一个函数 `set_intr_gate_ist` 将一个新的中断门插入到`IDT`中，其实现如下:

```C
static inline void set_intr_gate_ist(int n, void *addr, unsigned ist)
{
        BUG_ON((unsigned)n > 0xFF);
        _set_gate(n, GATE_INTERRUPT, addr, 0, ist, __KERNEL_CS);
}
```

该函数首先检查了参数 `n` 即[中断向量编号](http://en.wikipedia.org/wiki/Interrupt_vector_table) 是否不大于 `0xff` 或 255。之前的 [章节] (http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html) 中提到过，中断的向量号必须处于 0 到 255 的闭区间。然后调用了 `_set_gate` 函数将中断门设置到了 `IDT` 表中:

```C
static inline void _set_gate(int gate, unsigned type, void *addr,
                             unsigned dpl, unsigned ist, unsigned seg)
{
        gate_desc s;

        pack_gate(&s, type, (unsigned long)addr, dpl, ist, seg);
        write_idt_entry(idt_table, gate, &s);
        write_trace_idt_entry(gate, &s);
}
```

首先，通过 `pack_gate` 函数填充了一个表示 `IDT` 入口项的 `gate_desc` 类型的结构体，参数包括基地址，限制范围，[中断栈表](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks), [特权等级](http://en.wikipedia.org/wiki/Privilege_level) 和中断类型。中断类型的取值如下:

* `GATE_INTERRUPT`
* `GATE_TRAP`
* `GATE_CALL`
* `GATE_TASK`

并设置了该 `IDT` 项的`present`位域:

```C
static inline void pack_gate(gate_desc *gate, unsigned type, unsigned long func,
                             unsigned dpl, unsigned ist, unsigned seg)
{
        gate->offset_low        = PTR_LOW(func);
        gate->segment           = __KERNEL_CS;
        gate->ist               = ist;
        gate->p                 = 1;
        gate->dpl               = dpl;
        gate->zero0             = 0;
        gate->zero1             = 0;
        gate->type              = type;
        gate->offset_middle     = PTR_MIDDLE(func);
        gate->offset_high       = PTR_HIGH(func);
}
```

然后，我们把这个中断门通过 `write_idt_entry` 宏填入了 `IDT` 中。这个宏展开后是 `native_write_idt_entry` ，其将中断门信息通过索引拷贝到了 `idt_table` 之中:

```C
#define write_idt_entry(dt, entry, g)           native_write_idt_entry(dt, entry, g)

static inline void native_write_idt_entry(gate_desc *idt, int entry, const gate_desc *gate)
{
        memcpy(&idt[entry], gate, sizeof(*gate));
}
```

其中 `idt_table` 是一个 `gate_desc` 类型的数组:

```C
extern gate_desc idt_table[];
```

函数 `set_intr_gate_ist` 的内容到此为止。第二个函数 `set_system_intr_gate_ist` 的实现仅有一个地方不同:

```C
static inline void set_system_intr_gate_ist(int n, void *addr, unsigned ist)
{
        BUG_ON((unsigned)n > 0xFF);
        _set_gate(n, GATE_INTERRUPT, addr, 0x3, ist, __KERNEL_CS);
}
```

注意 `_set_gate` 函数的第四个参数是 `0x3`，而在 `set_intr_gate_ist`函数中这个值是 `0x0`，这个参数代表的是 `DPL`或称为特权等级。其中，`0`代表最高特权等级而 `3`代表最低等级。现在我们了解了 `set_system_intr_gate_ist`, `set_intr_gate_ist`, `set_intr_gate`这三函数的作用并回到 `early_trap_init`函数中:

```C
set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
```

我们设置了 `#DB`和 `int3`两个 `IDT`入口项。这些函数输入相同的参数组:

* vector number of an interrupt;
* address of an interrupt handler;
* interrupt stack table index.

这就是 `early_trap_init`函数的全部内容，你将在下一章节中看到更多与中断和服务函数相关的内容。

总结
--------------------------------------------------------------------------------

现在已经到了Linux内核中断和中断服务部分的第二部分的结尾。我们在之前的章节中了解了中断与异常处理的相关理论，并在本部分中开始深入阅读中断和异常处理的代码。我们从Linux内核启动最早期的代码中与中断相关的代码开始。下一部分中我们将继续深入这个有趣的主题，并学习更多关于中断处理相关的内容。

如果你有任何建议或疑问，请在我的 [twitter](https://twitter.com/0xAX)页面中留言或抖一抖我。

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

链接
--------------------------------------------------------------------------------

* [IDT](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [List of x86 calling conventions](http://en.wikipedia.org/wiki/X86_calling_conventions#List_of_x86_calling_conventions)
* [8086](http://en.wikipedia.org/wiki/Intel_8086)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [NX](http://en.wikipedia.org/wiki/NX_bit)
* [Extended Feature Enable Register](http://en.wikipedia.org/wiki/Control_register#Additional_Control_registers_in_x86-64_series)
* [Model-specific register](http://en.wikipedia.org/wiki/Model-specific_register)
* [Process identifier](https://en.wikipedia.org/wiki/Process_identifier)
* [lockdep](http://lwn.net/Articles/321663/)
* [irqflags tracing](https://www.kernel.org/doc/Documentation/irqflags-tracing.txt)
* [IF](http://en.wikipedia.org/wiki/Interrupt_flag)
* [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries)
* [Union type](http://en.wikipedia.org/wiki/Union_type)
* [this_cpu_* operations](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/this_cpu_ops.txt)
* [vector number](http://en.wikipedia.org/wiki/Interrupt_vector_table)
* [Interrupt Stack Table](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)
* [Privilege level](http://en.wikipedia.org/wiki/Privilege_level)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/interrupts-1.html)
