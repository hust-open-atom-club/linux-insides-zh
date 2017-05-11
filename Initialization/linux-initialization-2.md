内核初始化 第二部分
================================================================================

初期中断和异常处理
--------------------------------------------------------------------------------

在上一个 [部分](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-1.html) 我们谈到了初期中断初始化。目前我们已经处于解压缩后的Linux内核中了，还有了用于初期启动的基本的 [分页](https://en.wikipedia.org/wiki/Page_table) 机制。我们的目标是在内核的主体代码执行前做好准备工作。

我们已经在 [本章](https://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/index.html) 的 [第一部分](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-1.html) 做了一些工作，在这一部分中我们会继续分析关于中断和异常处理部分的代码。

我们在上一部分谈到了下面这个循环：

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
	set_intr_gate(i, early_idt_handler_array[i]);
```

这段代码位于 [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c)。在分析这段代码之前，我们先来了解一些关于中断和中断处理程序的知识。

理论
--------------------------------------------------------------------------------

中断是一种由软件或硬件产生的、向CPU发出的事件。例如，如果用户按下了键盘上的一个按键时，就会产生中断。此时CPU将会暂停当前的任务，并且将控制流转到特殊的程序中—— [中断处理程序(Interrupt Handler)](https://en.wikipedia.org/wiki/Interrupt_handler)。一个中断处理程序会对中断进行处理，然后将控制权交还给之前暂停的任务中。中断分为三类：

* 软件中断 - 当一个软件可以向CPU发出信号，表明它需要系统内核的相关功能时产生。这些中断通常用于系统调用；
* 硬件中断 - 当一个硬件有任何事件发生时产生，例如键盘的按键被按下；
* 异常 - 当CPU检测到错误时产生，例如发生了除零错误或者访问了一个不存在的内存页。

每一个中断和异常都可以由一个数来表示，这个数叫做 `向量号` ，它可以取从 `0` 到 `255` 中的任何一个数。通常在实践中前 `32` 个向量号用来表示异常，`32` 到 `255` 用来表示用户定义的中断。可以看到在上面的代码中，`NUM_EXCEPTION_VECTORS` 就定义为：

```C
#define NUM_EXCEPTION_VECTORS 32
```

CPU会从 [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) 或者 CPU 引脚接收中断，并使用中断向量号作为 `中断描述符表` 的索引。下面的表中列出了 `0-31` 号异常：

```
----------------------------------------------------------------------------------------------
|Vector|Mnemonic|Description         |Type |Error Code|Source                   |
----------------------------------------------------------------------------------------------
|0     | #DE    |Divide Error        |Fault|NO        |DIV and IDIV                          |
|---------------------------------------------------------------------------------------------
|1     | #DB    |Reserved            |F/T  |NO        |                                      |
|---------------------------------------------------------------------------------------------
|2     | ---    |NMI                 |INT  |NO        |external NMI                          |
|---------------------------------------------------------------------------------------------
|3     | #BP    |Breakpoint          |Trap |NO        |INT 3                                 |
|---------------------------------------------------------------------------------------------
|4     | #OF    |Overflow            |Trap |NO        |INTO  instruction                     |
|---------------------------------------------------------------------------------------------
|5     | #BR    |Bound Range Exceeded|Fault|NO        |BOUND instruction                     |
|---------------------------------------------------------------------------------------------
|6     | #UD    |Invalid Opcode      |Fault|NO        |UD2 instruction                       |
|---------------------------------------------------------------------------------------------
|7     | #NM    |Device Not Available|Fault|NO        |Floating point or [F]WAIT             |
|---------------------------------------------------------------------------------------------
|8     | #DF    |Double Fault        |Abort|YES       |Ant instrctions which can generate NMI|
|---------------------------------------------------------------------------------------------
|9     | ---    |Reserved            |Fault|NO        |                                      |
|---------------------------------------------------------------------------------------------
|10    | #TS    |Invalid TSS         |Fault|YES       |Task switch or TSS access             |
|---------------------------------------------------------------------------------------------
|11    | #NP    |Segment Not Present |Fault|NO        |Accessing segment register            |
|---------------------------------------------------------------------------------------------
|12    | #SS    |Stack-Segment Fault |Fault|YES       |Stack operations                      |
|---------------------------------------------------------------------------------------------
|13    | #GP    |General Protection  |Fault|YES       |Memory reference                      |
|---------------------------------------------------------------------------------------------
|14    | #PF    |Page fault          |Fault|YES       |Memory reference                      |
|---------------------------------------------------------------------------------------------
|15    | ---    |Reserved            |     |NO        |                                      |
|---------------------------------------------------------------------------------------------
|16    | #MF    |x87 FPU fp error    |Fault|NO        |Floating point or [F]Wait             |
|---------------------------------------------------------------------------------------------
|17    | #AC    |Alignment Check     |Fault|YES       |Data reference                        |
|---------------------------------------------------------------------------------------------
|18    | #MC    |Machine Check       |Abort|NO        |                                      |
|---------------------------------------------------------------------------------------------
|19    | #XM    |SIMD fp exception   |Fault|NO        |SSE[2,3] instructions                 |
|---------------------------------------------------------------------------------------------
|20    | #VE    |Virtualization exc. |Fault|NO        |EPT violations                        |
|---------------------------------------------------------------------------------------------
|21-31 | ---    |Reserved            |INT  |NO        |External interrupts                   |
----------------------------------------------------------------------------------------------
```

为了能够对中断进行处理，CPU使用了一种特殊的结构 - 中断描述符表（IDT）。IDT 是一个由描述符组成的数组，其中每个描述符都为8个字节，与全局描述附表一致；不过不同的是，我们把IDT中的每一项叫做 `门(gate)` 。为了获得某一项描述符的起始地址，CPU 会把向量号乘以8，在64位模式中则会乘以16。在前面我们已经见过，CPU使用一个特殊的 `GDTR` 寄存器来存放全局描述符表的地址，中断描述符表也有一个类似的寄存器 `IDTR` ，同时还有用于将基地址加载入这个寄存器的指令 `lidt` 。

64位模式下 IDT 的每一项的结构如下：

```
127                                                                             96
 --------------------------------------------------------------------------------
|                                                                               |
|                                Reserved                                       |
|                                                                               |
 --------------------------------------------------------------------------------
95                                                                              64
 --------------------------------------------------------------------------------
|                                                                               |
|                               Offset 63..32                                   |
|                                                                               |
 --------------------------------------------------------------------------------
63                               48 47      46  44   42    39             34    32
 --------------------------------------------------------------------------------
|                                  |       |  D  |   |     |      |   |   |     |
|       Offset 31..16              |   P   |  P  | 0 |Type |0 0 0 | 0 | 0 | IST |
|                                  |       |  L  |   |     |      |   |   |     |
 --------------------------------------------------------------------------------
31                                   15 16                                      0
 --------------------------------------------------------------------------------
|                                      |                                        |
|          Segment Selector            |                 Offset 15..0           |
|                                      |                                        |
 --------------------------------------------------------------------------------
```

其中:

* `Offset` - 代表了到中断处理程序入口点的偏移；
* `DPL` -    描述符特权级别；
* `P` -      Segment Present 标志;
* `Segment selector` - 在GDT或LDT中的代码段选择子；
* `IST` -    用来为中断处理提供一个新的栈。

最后的 `Type` 域描述了这一项的类型，中断处理程序共分为三种：

* 任务描述符
* 中断描述符
* 陷阱描述符

中断和陷阱描述符包含了一个指向中断处理程序的远 (far) 指针，二者唯一的不同在于CPU处理 `IF` 标志的方式。如果是由中断门进入中断处理程序的，CPU 会清除 `IF` 标志位，这样当当前中断处理程序执行时，CPU 不会对其他的中断进行处理；只有当当前的中断处理程序返回时，CPU 才在 `iret` 指令执行时重新设置 `IF` 标志位。

中断门的其他位为保留位，必须为0。下面我们来看一下 CPU 是如何处理中断的：

* CPU 会在栈上保存标志寄存器、`cs`段寄存器和程序计数器IP；
* 如果中断是由错误码引起的（比如 `#PF`）， CPU会在栈上保存错误码；
* 在中断处理程序执行完毕后，由`iret`指令返回。

OK，接下来我们继续分析代码。

设置并加载 IDT
--------------------------------------------------------------------------------

我们分析到了如下代码：

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
	set_intr_gate(i, early_idt_handler_array[i]);
```

这里循环内部调用了 `set_intr_gate` ，它接受两个参数：

* 中断号，即 `向量号`；
* 中断处理程序的地址。

同时，这个函数还会将中断门插入至 `IDT` 表中，代码中的 `&idt_descr` 数组即为 `IDT`。 首先让我们来看一下 `early_idt_handler_array` 数组，它定义在 [arch/x86/include/asm/segment.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/segment.h) 头文件中，包含了前32个异常处理程序的地址：

```C
#define EARLY_IDT_HANDLER_SIZE   9
#define NUM_EXCEPTION_VECTORS	32

extern const char early_idt_handler_array[NUM_EXCEPTION_VECTORS][EARLY_IDT_HANDLER_SIZE];
```

`early_idt_handler_array` 是一个大小为 `288` 字节的数组，每一项为 `9` 个字节，其中2个字节的备用指令用于向栈中压入默认错误码（如果异常本身没有提供错误码的话），2个字节的指令用于向栈中压入向量号，剩余5个字节用于跳转到异常处理程序。

在上面的代码中，我们只通过一个循环向 `IDT` 中填入了前32项内容，这是因为在整个初期设置阶段，中断是禁用的。`early_idt_handler_array` 数组中的每一项指向的都是同一个通用中断处理程序，定义在 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) 。我们先暂时跳过这个数组的内容，看一下 `set_intr_gate` 的定义。

`set_intr_gate` 宏定义在 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h)：

```C
#define set_intr_gate(n, addr)                         \
         do {                                                            \
                 BUG_ON((unsigned)n > 0xFF);                             \
                 _set_gate(n, GATE_INTERRUPT, (void *)addr, 0, 0,        \
                           __KERNEL_CS);                                 \
                 _trace_set_gate(n, GATE_INTERRUPT, (void *)trace_##addr,\
                                 0, 0, __KERNEL_CS);                     \
         } while (0)
```

首先 `BUG_ON` 宏确保了传入的中断向量号不会大于255，因为我们最多只有 `256` 个中断。然后它调用了 `_set_gate` 函数，它会将中断门写入 `IDT`：

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

在 `_set_gate` 函数的开始，它调用了 `pack_gate` 函数。这个函数会使用给定的参数填充 `gate_desc` 结构：

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
在这个函数里，我们把从主循环中得到的中断处理程序入口点地址拆成三个部分，填入门描述符中。下面的三个宏就用来做这个拆分工作：

```C
#define PTR_LOW(x) ((unsigned long long)(x) & 0xFFFF)
#define PTR_MIDDLE(x) (((unsigned long long)(x) >> 16) & 0xFFFF)
#define PTR_HIGH(x) ((unsigned long long)(x) >> 32)
```

调用 `PTR_LOW` 可以得到 x 的低 `2` 个字节，调用 `PTR_MIDDLE` 可以得到 x 的中间 `2` 个字节，调用 `PTR_HIGH` 则能够得到 x 的高 `4` 个字节。接下来我们来位中断处理程序设置段选择子，即内核代码段 `__KERNEL_CS`。然后将 `Interrupt Stack Table` 和 `描述符特权等级` （最高特权等级）设置为0，以及在最后设置 `GAT_INTERRUPT` 类型。

现在我们已经设置好了IDT中的一项，那么通过调用 `native_write_idt_entry` 函数来把复制到 `IDT`：

```C
static inline void native_write_idt_entry(gate_desc *idt, int entry, const gate_desc *gate)
{
        memcpy(&idt[entry], gate, sizeof(*gate));
}
```

主循环结束后，`idt_table` 就已经设置完毕了，其为一个 `gate_desc` 数组。然后我们就可以通过下面的代码加载 `中断描述符表`：

```C
load_idt((const struct desc_ptr *)&idt_descr);
```

其中，`idt_descr` 为：

```C
struct desc_ptr idt_descr = { NR_VECTORS * 16 - 1, (unsigned long) idt_table };
```

`load_idt` 函数只是执行了一下 `lidt` 指令：

```C
asm volatile("lidt %0"::"m" (*dtr));
```

你可能已经注意到了，在代码中还有对 `_trace_*` 函数的调用。这些函数会用跟 `_set_gate` 同样的方法对 `IDT` 门进行设置，但仅有一处不同：这些函数并不设置 `idt_table` ，而是 `trace_idt_table` ，用于设置追踪点（tracepoint，我们将会在其他章节介绍这一部分）。

好了，至此我们已经了解到，通过设置并加载 `中断描述符表` ，能够让CPU在发生中断时做出相应的动作。下面让我们来看一下如何编写中断处理程序。

初期中断处理程序
--------------------------------------------------------------------------------

在上面的代码中，我们用 `early_idt_handler_array` 的地址来填充了 `IDT` ，这个 `early_idt_handler_array` 定义在 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)：

```assembly
	.globl early_idt_handler_array
early_idt_handlers:
	i = 0
	.rept NUM_EXCEPTION_VECTORS
	.if (EXCEPTION_ERRCODE_MASK >> i) & 1
	pushq $0
	.endif
	pushq $i
	jmp early_idt_handler_common
	i = i + 1
	.fill early_idt_handler_array + i*EARLY_IDT_HANDLER_SIZE - ., 1, 0xcc
	.endr
```

这段代码自动生成为前 `32` 个异常生成了中断处理程序。首先，为了统一栈的布局，如果一个异常没有返回错误码，那么我们就手动在栈中压入一个 `0`。然后再在栈中压入中断向量号，最后跳转至通用的中断处理程序 `early_idt_handler_common` 。我们可以通过 `objdump` 命令的输出一探究竟：

```
$ objdump -D vmlinux
...
...
...
ffffffff81fe5000 <early_idt_handler_array>:
ffffffff81fe5000:       6a 00                   pushq  $0x0
ffffffff81fe5002:       6a 00                   pushq  $0x0
ffffffff81fe5004:       e9 17 01 00 00          jmpq   ffffffff81fe5120 <early_idt_handler_common>
ffffffff81fe5009:       6a 00                   pushq  $0x0
ffffffff81fe500b:       6a 01                   pushq  $0x1
ffffffff81fe500d:       e9 0e 01 00 00          jmpq   ffffffff81fe5120 <early_idt_handler_common>
ffffffff81fe5012:       6a 00                   pushq  $0x0
ffffffff81fe5014:       6a 02                   pushq  $0x2
...
...
...
```

由于在中断发生时，CPU 会在栈上压入标志寄存器、`CS` 段寄存器和 `RIP` 寄存器的内容。因此在 `early_idt_handler` 执行前，栈的布局如下：

```
|--------------------|
| %rflags            |
| %cs                |
| %rip               |
| rsp --> error code |
|--------------------|
```

下面我们来看一下 `early_idt_handler_common` 的实现。它也定义在 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S#L343) 文件中。首先它会检查当前中断是否为 [不可屏蔽中断(NMI)](http://en.wikipedia.org/wiki/Non-maskable_interrupt)，如果是则简单地忽略它们：

```assembly
	cmpl $2,(%rsp)
	je .Lis_nmi
```

其中 `is_nmi` 为:

```assembly
is_nmi:
	addq $16,%rsp
	INTERRUPT_RETURN
```

这段程序首先从栈顶弹出错误码和中断向量号，然后通过调用 `INTERRUPT_RETURN` ，即 `iretq` 指令直接返回。

如果当前中断不是 `NMI` ，则首先检查 `early_recursion_flag` 以避免在 `early_idt_handler_common` 程序中递归地产生中断。如果一切都没问题，就先在栈上保存通用寄存器，为了防止中断返回时寄存器的内容错乱：

```assembly
	pushq %rax
	pushq %rcx
	pushq %rdx
	pushq %rsi
	pushq %rdi
	pushq %r8
	pushq %r9
	pushq %r10
	pushq %r11
```

然后我们检查栈上的段选择子：

```assembly
	cmpl $__KERNEL_CS,96(%rsp)
	jne 11f
```

段选择子必须为内核代码段，如果不是则跳转到标签 `11` ，输出 `PANIC` 信息并打印栈的内容。然后我们来检查向量号，如果是 `#PF` 即 [缺页中断（Page Fault）](https://en.wikipedia.org/wiki/Page_fault)，那么就把 `cr2` 寄存器中的值赋值给 `rdi` ，然后调用 `early_make_pgtable` （详见后文）：

```assembly
	cmpl $14,72(%rsp)
	jnz 10f
	GET_CR2_INTO(%rdi)
	call early_make_pgtable
	andl %eax,%eax
	jz 20f
```

如果向量号不是 `#PF` ，那么就恢复通用寄存器：
```assembly
	popq %r11
	popq %r10
	popq %r9
	popq %r8
	popq %rdi
	popq %rsi
	popq %rdx
	popq %rcx
	popq %rax
```

并调用 `iret` 从中断处理程序返回。

第一个中断处理程序到这里就结束了。由于它只是一个初期中段处理程序，因此只处理缺页中断。下面让我们首先来看一下缺页中断处理程序，其他中断的处理程序我们之后再进行分析。

缺页中断处理程序
--------------------------------------------------------------------------------

在上一节中我们第一次见到了初期中断处理程序，它检查了缺页中断的中断号，并调用了 `early_make_pgtable` 来建立新的页表。在这里我们需要提供 `#PF` 中断处理程序，以便为之后将内核加载至 `4G` 地址以上，并且能访问位于4G以上的 `boot_params` 结构体。

`early_make_pgtable` 的实现在 [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c)，它接受一个参数：从 `cr2` 寄存器得到的地址，这个地址引发了内存中断。下面让我们来看一下：

```C
int __init early_make_pgtable(unsigned long address)
{
	unsigned long physaddr = address - __PAGE_OFFSET;
	unsigned long i;
	pgdval_t pgd, *pgd_p;
	pudval_t pud, *pud_p;
	pmdval_t pmd, *pmd_p;
	...
	...
	...
}
```

首先它定义了一些 `*val_t` 类型的变量。这些类型均为：

```C
typedef unsigned long   pgdval_t;
```

此外，我们还会遇见 `*_t` (不带val)的类型，比如 `pgd_t` ……这些类型都定义在 [arch/x86/include/asm/pgtable_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/pgtable_types.h)，形式如下：

```C
typedef struct { pgdval_t pgd; } pgd_t;
```

例如，

```C
extern pgd_t early_level4_pgt[PTRS_PER_PGD];
```

在这里 `early_level4_pgt` 代表了初期顶层页表目录，它是一个 `pdg_t` 类型的数组，其中的 `pgd` 指向了下一级页表。

在确认不是非法地址后，我们取得页表中包含引起 `#PF` 中断的地址的那一项，将其赋值给 `pgd` 变量：

```C
pgd_p = &early_level4_pgt[pgd_index(address)].pgd;
pgd = *pgd_p;
```

接下来我们检查一下 `pgd` ，如果它包含了正确的全局页表项的话，我们就把这一项的物理地址处理后赋值给 `pud_p` ：


```C
pud_p = (pudval_t *)((pgd & PTE_PFN_MASK) + __START_KERNEL_map - phys_base);
```

其中 `PTE_PFN_MASK` 是一个宏：

```C
#define PTE_PFN_MASK            ((pteval_t)PHYSICAL_PAGE_MASK)
```

展开后将为：

```C
(~(PAGE_SIZE-1)) & ((1 << 46) - 1)
```

或者写为：

```
0b1111111111111111111111111111111111111111111111
```

它是一个46bit大小的页帧屏蔽值。

如果 `pgd` 没有包含有效的地址，我们就检查 `next_early_pgt` 与 `EARLY_DYNAMIC_PAGE_TABLES`（即 `64` ）的大小。`EARLY_DYNAMIC_PAGE_TABLES` 它是一个固定大小的缓冲区，用来在需要的时候建立新的页表。如果 `next_early_pgt` 比 `EARLY_DYNAMIC_PAGE_TABLES` 大，我们就用一个上层页目录指针指向当前的动态页表，并将它的物理地址与 `_KERPG_TABLE` 访问权限一起写入全局页目录表：

```C
if (next_early_pgt >= EARLY_DYNAMIC_PAGE_TABLES) {
	reset_early_page_tables();
    goto again;
}
	
pud_p = (pudval_t *)early_dynamic_pgts[next_early_pgt++];
for (i = 0; i < PTRS_PER_PUD; i++)
	pud_p[i] = 0;
*pgd_p = (pgdval_t)pud_p - __START_KERNEL_map + phys_base + _KERNPG_TABLE;
```

然后我们来修正上层页目录的地址：

```C
pud_p += pud_index(address);
pud = *pud_p;
```

下面我们对中层页目录重复上面同样的操作。最后我们利用 In the end we fix address of the page middle directory which contains maps kernel text+data virtual addresses:

```C
pmd = (physaddr & PMD_MASK) + early_pmd_flags;
pmd_p[pmd_index(address)] = pmd;
```

到此缺页中断处理程序就完成了它所有的工作，此时 `early_level4_pgt` 就包含了指向合法地址的项。

小结
--------------------------------------------------------------------------------

本书的第二部分到此结束了。

如果你有任何问题或建议，请在twitter上联系我 [0xAX](https://twitter.com/0xAX)，或者通过[邮件](anotherworldofworld@gmail.com)与我沟通，还可以新开[issue](https://github.com/MintCN/linux-insides-zh/issues/new)。

接下来我们将会看到进入内核入口点 `start_kernel` 函数之前剩下所有的准备工作。

相关链接
--------------------------------------------------------------------------------

* [GNU assembly .rept](https://sourceware.org/binutils/docs-2.23/as/Rept.html)
* [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [Page table](https://en.wikipedia.org/wiki/Page_table)
* [Interrupt handler](https://en.wikipedia.org/wiki/Interrupt_handler)
* [Page Fault](https://en.wikipedia.org/wiki/Page_fault),
* [Previous part](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-1.html)
