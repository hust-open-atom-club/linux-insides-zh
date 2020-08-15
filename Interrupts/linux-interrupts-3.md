中断和中断处理. Part 3.
================================================================================

异常处理
--------------------------------------------------------------------------------

这是第三部分 [chapter](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) Linux内核中有关中断和异常处理 在前面的内容中 [part](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html) 我们停止了 `setup_arch` 函数 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blame/master/arch/x86/kernel/setup.c) 源代码文件.

我们已经知道该函数执行特定于体系结构的东西的初始化。 在我们的例子中，`setup_arch`函数执行与[x86_64](https://en.wikipedia.org/wiki/X86-64) architecture相关的初始化工作。 `setup_arch`是一个大功能，在上一部分中，我们停止了以下两个异常的两个异常处理程序的设置：
* `#DB` - 调试异常，将控制从中断的进程转移到调试处理程序；
* `#BP` - 由`int`指令引起的断点异常。

这些异常允许x86_64体系结构具有早期异常处理功能，以便于通过[kgdb](https://en.wikipedia.org/wiki/KGDB) 进行调试
正如您记得的，我们在`early_trap_init`函数中设置了这些异常处理程序：

```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
        load_idt(&idt_descr);
}
```

来自 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c). 我们已经在上一部分中看到了`set_intr_gate_ist`和`set_system_intr_gate_ist`函数的实现，现在我们将看看这两个异常处理程序的实现。

调试和断点异常
--------------------------------------------------------------------------------

Ok，我们在`early_trap_init`函数中为`#DB`和`#BP`异常设置了异常处理程序，现在是时候考虑它们的实现了。但是，在执行此操作之前，我们首先看一下这些异常的详细信息。

第一个异常-`＃DB`或`debug`异常是在发生调试事件时发生的。例如-尝试更改[debug register](http://en.wikipedia.org/wiki/X86_debug_register) 的内容。debug register是从 [Intel 80386](http://en.wikipedia.org/wiki/Intel_80386) 处理器开始在`x86`处理器中提供的特殊寄存器，从CPU扩展的名可以知道，这些寄存器中的正在调试这些功能。

这些寄存器允许在代码上设置断点，并读取或写入数据以对其进行跟踪。debug register只能在特权模式下访问，以任何其他特权级别执行时尝试读取或写入调试寄存器都会导致[general protection fault](https://en.wikipedia.org/wiki/General_protection_fault) 异常。这就是为什么我们对`#DB`异常使用了`set_intr_gate_ist`，而不对`set_system_intr_gate_ist`使用。

`#DB`异常的记录编号为1（我们将其作为X86_TRAP_DB传递），并且正如我们在规范中可能会看到的那样，该异常没有错误代码：
```
+-----------------------------------------------------+
|Vector|Mnemonic|Description         |Type |Error Code|
+-----------------------------------------------------+
|1     | #DB    |Reserved            |F/T  |NO        |
+-----------------------------------------------------+
```

第二个异常是处理器执行[int 3](http://en.wikipedia.org/wiki/INT_%28x86_instruction%29#INT_3) 指令时发生的`#BP`或`breakpointv`异常。 与`DB`异常不同，`＃BP`异常可能在用户空间中发生。 我们可以将其添加到代码中的任何位置，让我们看一下简单的程序：
```C
// breakpoint.c
#include <stdio.h>

int main() {
    int i;
    while (i < 6){
	    printf("i equal to: %d\n", i);
	    __asm__("int3");
		++i;
    }
}
```

如果我们编译并运行该程序，我们将看到以下输出：

```
$ gcc breakpoint.c -o breakpoint
i equal to: 0
Trace/breakpoint trap
```

但是，如果将其与gdb一起运行，我们将看到断点并可以继续执行程序：

```
$ gdb breakpoint
...
...
...
(gdb) run
Starting program: /home/alex/breakpoints 
i equal to: 0

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400585 in main ()
=> 0x0000000000400585 <main+31>:	83 45 fc 01	add    DWORD PTR [rbp-0x4],0x1
(gdb) c
Continuing.
i equal to: 1

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400585 in main ()
=> 0x0000000000400585 <main+31>:	83 45 fc 01	add    DWORD PTR [rbp-0x4],0x1
(gdb) c
Continuing.
i equal to: 2

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400585 in main ()
=> 0x0000000000400585 <main+31>:	83 45 fc 01	add    DWORD PTR [rbp-0x4],0x1
...
...
...
```

从这一刻起，我们对这两个异常有所了解，我们可以把关注点转移到它们的处理程序。

异常处理程序之前的准备
--------------------------------------------------------------------------------

正如您之前可能注意到的那样，`set_intr_gate_ist`和`set_system_intr_gate_ist`函数在其第二个参数中使用异常处理程序的地址。 否则，我们的两个异常处理程序将是：
* `debug`;
* `int3`.

你在C代码中找不到这些功能。这些所有功能都可以在内核的`*.c/*.h`文件中找到，这些功能的定义位于[arch/x86/include/asm/traps.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/traps.h)内核头文件：

```C
asmlinkage void debug(void);
```

and

```C
asmlinkage void int3(void);
```

您可能会在这些函数的定义中注意到`asmlinkage`指令。 该指令是[gcc](http://en.wikipedia.org/wiki/GNU_Compiler_Collection) 的特殊说明符。 实际上，对于从汇编中调用的C函数，我们需要显式声明函数调用约定。在我们的例子中，如果函数使用`asmlinkage`描述符创建，则`gcc`将编译该函数以从堆栈中检索参数。
So, both handlers are defined in the [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) assembly source code file with the `idtentry` macro:

因此，这两个处理程序都在[arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) 汇编源代码文件中定义`idtentry`宏：
```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

and

```assembly
idtentry int3 do_int3 has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

每个异常处理程序可以由两部分组成。 第一部分是通用部分，所有异常处理程序都相同。 异常处理程序应将[general purpose registers](https://en.wikipedia.org/wiki/Processor_register) 保存在堆栈上，如果异常来自用户空间，则应切换到内核堆栈，并将控制权转移到异常的第二部分 处理程序。 异常处理程序的第二部分完成某些工作取决于某些异常。 例如，页面错误异常处理程序应找到给定地址的虚拟页面，无效的操作码异常处理程序应发送`SIGILL` [signal](https://en.wikipedia.org/wiki/Unix_signal) 等。
正如我们所见，异常处理程序从[arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/entry_64.S) 汇编源代码文件，因此让我们看一下该宏的实现。 我们可以会看到，`idtentry`宏接受五个参数：

*`sym`用`globl name`定义全局符号，它将作为异常处理程序的入口；
*`do_sym`符号名称，代表异常处理程序的辅助条目；
*`has_error_code`有关异常错误代码的存在的信息。

最后两个参数是可选的：

*`paranoid`-向我们展示了如何检查当前模式（稍后将详细解释）；
*`shift_ist`-显示我们是在“中断堆栈表”上运行的异常。

`idtentry`宏的定义如下：

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
...
...
...
END(\sym)
.endm
```

Before we will consider internals of the `idtentry` macro, we should to know state of stack when an exception occurs. As we may read in the [Intel® 64 and IA-32 Architectures Software Developer’s Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html), the state of stack when an exception occurs is following:

在考虑`identry`宏的内部之前，我们应该知道发生异常时的堆栈状态。 正如我们可能会在[Intel®64 and IA-32 Architectures Software Developer's Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html) ，则发生异常时的堆栈状态如下：
```
    +------------+
+40 | %SS        |
+32 | %RSP       |
+24 | %RFLAGS    |
+16 | %CS        |
 +8 | %RIP       |
  0 | ERROR CODE | <-- %RSP
    +------------+
```

现在我们可以开始考虑`idtmacro`的实现了。 `#DB`和`BP`异常处理程序都定义为：
```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
idtentry int3 do_int3 has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

如果我们看一下这些定义，我们可能知道编译器将生成两个带有`debug`和`int3`名称的例程，并且这两个异常处理程序在经过一些准备后将调用`do_debug`和`do_int3`辅助处理程序。 第三个参数定义了错误代码的存在，并且我们可以看到我们的两个异常都没有它们。 如上图所示，如果有异常，处理器会将错误代码压入堆栈。 在我们的例子中，`debug`和`int3`异常没有错误代码。 这可能会带来一些困难，因为对于提供错误代码的异常和未提供错误代码的异常，堆栈的外观会有所不同。 这就是为什么`idtentry`宏的实现始于在异常未提供的情况下将伪造的错误代码放入堆栈的原因：
```assembly
.ifeq \has_error_code
    pushq	$-1
.endif
```

这不仅是伪造的错误代码。此外，“-1”还代表无效的系统调用号码，因此系统调用重启逻辑将不会被触发。

`idtentry`宏`shift_ist`和`paranoid`的最后两个参数允许您知道是否从`Interrupt Stack Table`运行在堆栈上的异常处理程序。您可能已经知道系统中的每个内核线程都有自己的堆栈。除了这些堆栈外，还有一些专用堆栈与系统中的每个处理器相关联。这些堆栈之一是-异常堆栈。 [x86_64](https://en.wikipedia.org/wiki/X86-64) 架构提供了称为`中断堆栈表`的特殊功能。此功能允许针对指定事件（例如原子异常（如double fault）等）切换到新堆栈。因此，使用shift_ist参数可以让我们知道是否需要为异常处理程序打开IST堆栈。

第二个参数`paranoid`定义了一种方法，该方法可以帮助我们知道我们是来自用户空间还是来自异常处理程序。确定这一点的最简单方法是通过`CS`段寄存器中的`CPL`或`Current Privilege Level`。如果等于`3`，则来自用户空间；如果为零，则来自内核空间：

```
testl $3,CS(%rsp)
jnz userspace
...
...
...

// 我们来自内核空间
```

但是不幸的是，这种方法不能100％的保证。如内核文档中所述：
> if we are in an NMI/MCE/DEBUG/whatever super-atomic entry context,
> which might have triggered right after a normal entry wrote CS to the
> stack but before we executed SWAPGS, then the only safe way to check
> for GS is the slower method: the RDMSR.


换句话说，例如，`NMI`可能发生在[swapgs](http://www.felixcloutier.com/x86/SWAPGS.html) 指令的关键部分内。 这样，我们应该检查`MSR_GS_BASE` [模型专用寄存器](https://en.wikipedia.org/wiki/Model-specific_register) 的值，该值存储指向每个cpu区域开始的指针。 因此，要检查我们是否来自用户空间，我们应该检查`MSR_GS_BASE`模型特定寄存器的值，如果它是负数，则来自内核空间，否则来自用户空间：
```assembly
movl $MSR_GS_BASE,%ecx
rdmsr
testl %edx,%edx
js 1f
```

在前两行代码中，我们将模型专用寄存器`MSR_GS_BASE`的值读入edx:eax对。 我们不能从用户空间为gs设置负值。 但是从另一面我们知道，物理内存的直接映射是从虚拟地址`0xffff880000000000`开始的。 这样，`MSR_GS_BASE`将包含从`0xffff880000000000`到`0xffffc7ffffffffff`的地址。 执行完`rdmsr`指令后，`％edx`寄存器中的最小可能值为-`0xffff8800`，即无符号4个字节的`-30720`。 这就是指向`每个CPU`区域开始的内核空间`gs`包含负值的原因。
将伪错误代码压入堆栈后，我们应该使用以下命令为通用寄存器分配空间：

```assembly
ALLOC_PT_GPREGS_ON_STACK
```

在[arch / x86 / entry / calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/calling.h) 头文件中定义的宏。 该宏仅在堆栈上分配15 * 8字节空间以保留通用寄存器：

```assembly
.macro ALLOC_PT_GPREGS_ON_STACK addskip=0
    addq	$-(15*8+\addskip), %rsp
.endm
```


因此，在执行`ALLOC_PT_GPREGS_ON_STACK`之后，堆栈将如下所示：

```
     +------------+
+160 | %SS        |
+152 | %RSP       |
+144 | %RFLAGS    |
+136 | %CS        |
+128 | %RIP       |
+120 | ERROR CODE |
     |------------|
+112 |            |
+104 |            |
 +96 |            |
 +88 |            |
 +80 |            |
 +72 |            |
 +64 |            |
 +56 |            |
 +48 |            |
 +40 |            |
 +32 |            |
 +24 |            |
 +16 |            |
  +8 |            |
  +0 |            | <- %RSP
     +------------+
```


在为通用寄存器分配空间之后，我们进行一些检查以了解异常是否来自用户空间，如果是，则应移回中断的进程堆栈或保留在异常堆栈上：


```assembly
.if \paranoid
    .if \paranoid == 1
	    testb	$3, CS(%rsp)
	    jnz	1f
	.endif
	call	paranoid_entry
.else
	call	error_entry
.endif
```



让我们考虑一下所有情况

用户空间中发生异常
--------------------------------------------------------------------------------


首先，让我们考虑一个异常具有像我们的`debug`和`int3`异常这样的`paranoid = 1`的情况。 在这种情况下，如果来自用户空间，否则我们将从CS段寄存器中检查选择器，并跳转到`1f`标签上，否则将以其他方式调用`paranoid_entry`。
Let's consider first case when we came from userspace to an exception handler. As described above we should jump at `1` label. The `1` label starts from the call of the

```assembly
call	error_entry
```

该例程将所有通用寄存器保存在堆栈中先前分配的区域中:
```assembly
SAVE_C_REGS 8
SAVE_EXTRA_REGS 8
```




这两个宏都在[arch/x86/entry/calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/calling.h) 头文件中定义并移动 通用寄存器的值到堆栈中的某个位置，例如：
```assembly
.macro SAVE_EXTRA_REGS offset=0
	movq %r15, 0*8+\offset(%rsp)
	movq %r14, 1*8+\offset(%rsp)
	movq %r13, 2*8+\offset(%rsp)
	movq %r12, 3*8+\offset(%rsp)
	movq %rbp, 4*8+\offset(%rsp)
	movq %rbx, 5*8+\offset(%rsp)
.endm
```

执行`SAVE_C_REGS`和`SAVE_EXTRA_REGS`之后，堆栈将如下所示:

```
     +------------+
+160 | %SS        |
+152 | %RSP       |
+144 | %RFLAGS    |
+136 | %CS        |
+128 | %RIP       |
+120 | ERROR CODE |
     |------------|
+112 | %RDI       |
+104 | %RSI       |
 +96 | %RDX       |
 +88 | %RCX       |
 +80 | %RAX       |
 +72 | %R8        |
 +64 | %R9        |
 +56 | %R10       |
 +48 | %R11       |
 +40 | %RBX       |
 +32 | %RBP       |
 +24 | %R12       |
 +16 | %R13       |
  +8 | %R14       |
  +0 | %R15       | <- %RSP
     +------------+
```

在内核将通用寄存器保存在堆栈中之后，应该使用以下命令再次检查来自用户空间：
```assembly
testb	$3, CS+8(%rsp)
jz	.Lerror_kernelspace
```

因为如果报告文档中描述的`％RIP`被截断，我们可能有潜在的错误。 无论如何，在两种情况下，都将执行[SWAPGS](http://www.felixcloutier.com/x86/SWAPGS.html) 指令，并且将交换` MSR_KERNEL_GS_BASE`和` MSR_GS_BASE`中的值。 从这一刻开始，`％gs`寄存器将指向内核结构的基址。 因此，调用了`SWAPGS`指令，这是`error_entry`路由的要点。

现在我们可以回到`idtentry`宏。 调用`error_entry`之后，我们可能会看到以下汇编代码：
```assembly
movq	%rsp, %rdi
call	sync_regs
```

在这里，我们将堆栈指针`％rdi`寄存器的基地址放入其中，这将是`sync_regs`的第一个参数(根据[x86_64 ABI](https://www.uclibc.org/docs/psABI-x86_64.pdf) ) 函数并调用[arch / x86 / kernel / traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c) 源代码中定义的函数 文件：

```C
asmlinkage __visible notrace struct pt_regs *sync_regs(struct pt_regs *eregs)
{
	struct pt_regs *regs = task_pt_regs(current);
	*regs = *eregs;
	return regs;
}
```

此函数采用在[arch/x86/include/asm/processor.h]中定义的`task_ptr_regs`宏的结果(https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/ include / asm / processor.h)头文件，将其存储在堆栈指针中并返回。 宏`task_ptr_regs`扩展为`thread.sp0`的地址，该地址表示指向普通内核堆栈的指针：
```C
#define task_pt_regs(tsk)       ((struct pt_regs *)(tsk)->thread.sp0 - 1)
```


正如来自用户空间一样，这意味着异常处理程序将在实际流程上下文中运行。 从`sync_regs`获取堆栈指针后，我们切换堆栈：

```assembly
movq	%rax, %rsp
```

异常处理程序将调用辅助处理程序之前的最后两个步骤是：

1.传递指向`pt_regs`结构的指针，该结构包含保留的通用寄存器到`％rdi`寄存器：

```assembly
movq	%rsp, %rdi
```


因为它将作为辅助异常处理程序的第一个参数传递。


2.将错误代码传递到`％rsi`寄存器，因为它将是异常处理程序的第二个参数，并在堆栈上将其设置为-1，其目的与我们之前相同 防止重新启动系统调用 ：

```
.if \has_error_code
	movq	ORIG_RAX(%rsp), %rsi
	movq	$-1, ORIG_RAX(%rsp)
.else
	xorl	%esi, %esi
.endif
```


另外，如果异常不提供错误代码，可能会看到我们将上面的`％esi`寄存器清零了。

最后，我们只调用辅助异常处理程序：
```assembly
call	\do_sym
```

which:

```C
dotraplinkage void do_debug(struct pt_regs *regs, long error_code);
```

将用于`debug`异常和：

```C
dotraplinkage void notrace do_int3(struct pt_regs *regs, long error_code);
```



将用于`int 3`例外。 在本部分中，我们将看不到辅助处理程序的实现，因为它们非常具体，但是在下一部分中将看到其中的一些。

我们只是考虑了在用户空间中发生异常的第一种情况。 我们考虑最后两个。

内核空间中发生了偏执> 0的异常
--------------------------------------------------------------------------------

在这种情况下，内核空间中发生了异常，并且为该异常使用`paranoid = 1`定义了`idtentry`宏。 `paranoid`的值意味着我们应该使用在本部分开头看到的更慢的方式来检查我们是否真的来自内核空间。 `paranoid_entry`路由使我们知道这一点：

```assembly
ENTRY(paranoid_entry)
	cld
	SAVE_C_REGS 8
	SAVE_EXTRA_REGS 8
	movl	$1, %ebx
	movl	$MSR_GS_BASE, %ecx
	rdmsr
	testl	%edx, %edx
	js	1f
	SWAPGS
	xorl	%ebx, %ebx
1:	ret
END(paranoid_entry)
```

如您所见，此功能代表了我们之前介绍的功能。 我们使用第二（慢）方法来获取有关被中断任务的先前状态的信息。 当我们检查并在来自用户空间的情况下执行`SWAPGS`时，我们应该做与之前相同的操作：我们需要将指针指向一个结构，该结构将通用寄存器保存到`％rdi`（ 将是辅助处理程序的第一个参数），如果异常将其提供给％rsi（将是辅助处理程序的第二个参数），则放置错误代码：

```assembly
movq	%rsp, %rdi

.if \has_error_code
	movq	ORIG_RAX(%rsp), %rsi
	movq	$-1, ORIG_RAX(%rsp)
.else
	xorl	%esi, %esi
.endif
```

调用异常的辅助处理程序之前的最后一步是清理新的`IST`堆栈帧：
```assembly
.if \shift_ist != -1
	subq	$EXCEPTION_STKSZ, CPU_TSS_IST(\shift_ist)
.endif
```


您可能还记得我们将`shift_ist`作为`iddentry`宏的参数传递了。 在这里，我们检查其值，如果其值不等于-1，则通过`shift_ist` 索引从`中断堆栈表`中获取指向堆栈的指针并进行设置。

在第二种方法的结尾，我们只是像以前一样调用辅助异常处理程序：
```assembly
call	\do_sym
```

最后一种方法与前面两种方法都相似，但是`paranoid = 0`发生了例外，我们可以使用快速方法确定我们的来源。
从异常处理程序退出
--------------------------------------------------------------------------------



在辅助处理程序完成工作之后，我们将返回到`idtentry`宏，下一步将跳转到`error_exit`：
```assembly
jmp	error_exit
```

在相同的[arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) 汇编源代码中定义的`error_exit`函数 文件，此功能的主要目标是从用户空间或内核空间知道我们的位置，并根据此位置执行`SWPAGS`。 将寄存器恢复到先前的状态，并执行`iret`指令将控制权转移到中断的任务。

That's all.

总结完毕
--------------------------------------------------------------------------------

第三部分到此结束，有关Linux内核中的中断和中断处理。 在上一部分中，我们看到了使用`#DB`和`#BP`对[中断描述符表](https://en.wikipedia.org/wiki/Interrupt_descriptor_table) 的初始化，并开始进行控制之前的准备工作 将被转移到异常处理程序和这部分中某些中断处理程序的实现。 在下一部分中，我们将继续深入探讨该主题，然后通过`setup_arch`函数进行下一步，并尝试了解处理相关内容的中断。
如果您有任何疑问或建议，请在[twitter](https://twitter.com/0xAX) 上给我写评论或ping我。
**请注意，英语不是我的母语，对于由此带来的不便，我深表歉意。如果发现任何错误，请将PR发送给[linux-insides]（https://github.com/0xAX/linux-insides）。**
以下链接
--------------------------------------------------------------------------------

* [Debug registers](http://en.wikipedia.org/wiki/X86_debug_register)
* [Intel 80385](http://en.wikipedia.org/wiki/Intel_80386)
* [INT 3](http://en.wikipedia.org/wiki/INT_%28x86_instruction%29#INT_3)
* [gcc](http://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [TSS](http://en.wikipedia.org/wiki/Task_state_segment)
* [GNU assembly .error directive](https://sourceware.org/binutils/docs/as/Error.html#Error)
* [dwarf2](http://en.wikipedia.org/wiki/DWARF)
* [CFI directives](https://sourceware.org/binutils/docs/as/CFI-directives.html)
* [IRQ](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [system call](http://en.wikipedia.org/wiki/System_call)
* [swapgs](http://www.felixcloutier.com/x86/SWAPGS.html)
* [SIGTRAP](https://en.wikipedia.org/wiki/Unix_signal#SIGTRAP)
* [Per-CPU variables](http://0xax.gitbooks.io/linux-insides/content/Concepts/per-cpu.html)
* [kgdb](https://en.wikipedia.org/wiki/KGDB)
* [ACPI](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)
* [Previous part](http://0xax.gitbooks.io/linux-insides/content/interrupts/index.html)
