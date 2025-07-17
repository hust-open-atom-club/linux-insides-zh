中断和中断处理 Part 3.
================================================================================

异常处理
--------------------------------------------------------------------------------
这是关于Linux内核中断和异常处理[章节](https://0xax.gitbook.io/linux-insides/summary/interrupts)的第三部分。在[上一小节](/Interrupts/linux-interrupts-2.md)中，我们结束于[arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blame/master/arch/x86/kernel/setup.c)中的 `setup_arch` 函数。
我们已经知道这个函数执行架构相关的初始化操作。在我们的例子中，`setup_arch`函数执行[x86_64](https://en.wikipedia.org/wiki/X86-64)架构相关的初始化工作，`setup_arch`函数是一个体量庞大的函数，在前一节中，我们结束于为以下两个异常设置处理程序位置。

* `#DB` - 调试异常，控制权从中断进程转移到调试处理程序；
* `#BP` - 断点异常，由`int 3`指令触发。


这些异常使得`x86_64`架构能够实现早期异常处理，以便通过 [kgdb](https://en.wikipedia.org/wiki/KGDB) 进行调试。
As you can remember we set these exceptions handlers in the `early_trap_init` function:
如您所知，我们在`early_trap_init`函数中设置了那些异常处理程序：

```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
        load_idt(&idt_descr);
}
```
在上一节中，我们已经在[arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)看到了`set_intr_gate_ist` 和 `set_system_intr_gate_ist`两个函数的具体实现。现在我们深入这两个异常处理程序的实现。


调试和断点异常
--------------------------------------------------------------------------------

Ok，我们已经在`early_trap_init`函数中设置了`#DB`和`#BP`异常的异常处理程序，现在是时候考虑它们的具体实现了。但在这之前，我们先了解下这些异常的细节。

第一个异常- `#DB` 或 `debug` 异常触发于一个调试事件发生时。例如- 企图改变一个[调试寄存器](http://en.wikipedia.org/wiki/X86_debug_register)的值。调试寄存器是自[Intel 80386](http://en.wikipedia.org/wiki/Intel_80386) 处理器起引入`x86`处理器中的特殊寄存器。从这一CPU扩展的名称可以看出，他们的主要用途是调试。


这些寄存器允许在代码中设置断点，并通过读写数据实现跟踪功能。调试寄存器只能在特权模式下访问，如果在其他任何特权级别下尝试读取或写入这些寄存器，都会触发[一般保护错误](https://en.wikipedia.org/wiki/General_protection_fault)。正因如此，我们在处理 `#DB`异常时使用`set_intr_gate_ist`，而不是`set_system_intr_gate_ist`。


`#DB` 异常的向量号是 1 （我们以 `X86_TRAP_DB` 传递此参数）。并且根据手册描述，此异常没有错误码。

```
+-----------------------------------------------------+
|Vector|Mnemonic|Description         |Type |Error Code|
+-----------------------------------------------------+
|1     | #DB    |Reserved            |F/T  |NO        |
+-----------------------------------------------------+
```


第二个异常是 `#BP` 或 `breakpoint` 异常，当处理器执行 [int 3](http://en.wikipedia.org/wiki/INT_%28x86_instruction%29#INT_3) 指令时会触发此异常。不同于 `DB` 异常，`#BP` 异常可以发生在用户空间。我们可以在我们代码的任何地方添加它。例如，来看这个简单的程序：

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


如果我们编译运行这段代码，会看到如下输出：
```
$ gcc breakpoint.c -o breakpoint
$ ./breakpoint
i equal to: 0
Trace/breakpoint trap
```

但是如果我们使用 gdb 运行此代码，我们会看到我们的断点，并可以继续执行我们的程序：

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

至此，我们对这两种异常有了初步了解，接下来我们可以分析其异常处理程序的实现细节了。


异常处理程序之前的准备
--------------------------------------------------------------------------------

如你所知， `set_intr_gate_ist` 和 `set_system_intr_gate_ist` 函数的第二个参数是异常处理程序的地址。在我们的例子中，这两个异常处理程序将是：

* `debug`;
* `int3`.

你不会在C代码中找到这些函数。它们的声明仅存在于内核的 `*.c/*.h` 文件中，更确切地说，在
[arch/x86/include/asm/traps.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/traps.h) 内核头文件中：
```C
asmlinkage void debug(void);
```
和

```C
asmlinkage void int3(void);
```

你或许注意到这些函数中的 `asmlinkage` 指令。该指令是 [gcc](http://en.wikipedia.org/wiki/GNU_Compiler_Collection) 中的特殊限定符。对于从汇编代码调用的 `C` 函数，我们需要显式定义其调用约定。在我们的例子中，以 `asmlinkage` 描述符修饰的函数，`gcc` 将编译该函数使其从栈中获取参数。



因此，这两个异常处理函数都在 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) 汇编源文件中定义，使用的是 `idtentry` 宏：

```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```
和

```assembly
idtentry int3 do_int3 has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

每个异常处理程序都包含两个部分。第一部分是通用部分，这部分对所有异常处理函数来说都是相同的。一个异常处理程序应该保存 [通用寄存器](https://en.wikipedia.org/wiki/Processor_register) 的值到栈中，如果异常来自于用户空间，则切换为内核栈，并且将控制权转移到异常处理程序的第二部分。异常处理程序的第二部分根据不同的异常做出不同的处理。例如缺页错误异常处理程序应该为给定的地址查找虚拟页。无效操作符异常处理程序应该发送 `SIGILL` [信号](https://en.wikipedia.org/wiki/Unix_signal) 等等。


正如我们所见，一个异常处理程序开始于[arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S)汇编源文件中的 `idtentry` 宏定义。因此我们看看这个宏的实现。我们会看到，`idtentry` 宏有五个参数。


* `sym` - 使用`.globl name` 定义了一个全局符号，它将成为异常处理程序的入口点；
* `do_sym` - 表示异常处理程序的第二入口点的符号名称；
* `has_error_code` - 关于异常错误码的存在信息。

后两个参数是可选的。


* `paranoid` - 显示我们需要检查当前模式（稍后会详细解释）；
* `shift_ist` - 显示异常是否在 `Interrupt Stack Table` 运行。


`.idtentry` 宏的定义如下：

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
...
...
...
END(\sym)
.endm
```

在我们考虑 `idtentry` 宏的内部之前，我们应该了解异常发生时栈的状态。正如我们在 [Intel® 64 and IA-32 Architectures Software Developer’s Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html) 读到的，当异常发生时栈的状态如下所示：

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

Now we may start to consider implementation of the `idtmacro`. Both `#DB` and `BP` exception handlers are defined as:
现在我们可以开始考虑 `idtmacro` 宏的实现。`#DB` 和 `BP` 异常处理程序都定义为：

```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
idtentry int3 do_int3 has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

如果我们查看这些定义，就会发现编译器会生成两个名为 `debug` 和 `int3` 的例程，并且这两个异常处理程序在进行一些准备工作后，会调用 `do_debug` 和 `do_int3` 这两个二级处理函数。第三个参数用来指示是否存在错误代码，正如我们所见，这两种异常都没有错误代码。如上图所示，如果某个异常自带错误代码，处理器会将该错误代码压入栈中。而在我们的例子中，`debug` 与 `int3` 异常并不包含错误代码。这会带来一些麻烦，因为对比带有错误代码的异常和不带错误代码的异常，其栈结构是不同的。因此，`idtentry` 宏的实现一开始就会在异常不提供错误代码时，向栈中压入一个伪错误代码：

```assembly
.ifeq \has_error_code
    pushq	$-1
.endif
```

但它不仅仅只是伪错误代码，此外 `-1` 还表示的是无效的系统调用号，因此系统调用重启逻辑不会被触发。

`idtentry` 宏的最后两个参数 `shift_ist` 和 `paranoid` 用于判断异常处理程序是否运行在`中断栈表`的堆栈上 。你或许已经知道每一个内核线程在系统中都有它自己的堆栈。除了这些堆栈以外，在系统中还有还有和每个处理器相关联的专用堆栈。其中一个栈是—— 异常堆栈。[x86_64](https://en.wikipedia.org/wiki/X86-64) 架构提供了一个特殊的功能，叫做 - `中断栈表`。这个功能允许为指定的事件例如像`双重故障`等的原子异常切换到一个新的堆栈。因此，`shift_ist` 参数允许我们判断异常处理程序是否需要切换到 `IST` 堆栈上。


第二个参数 —— `paranoid` 定义了一个帮助我们判断我们是否从用户空间来到异常处理程序的方法。判断这个最简单的方法就是查看 `CS` 段寄存器中的 `CPL` 或 `当前特权级`。如果它等于 `3`，我们来自用户空间，如果为 `0`，我们来自内核空间：

```
testl $3,CS(%rsp)
jnz userspace
...
...
...
// we are from the kernel space
```

但是不幸的是，这种方法并不能100%保证。正如在内核文档中描述的：
> 如果我们在一个 NMI/MCE/DEBUG/其他 超级原子入口上下文中，
> 它可能会在正常入口写入 CS 到堆栈后立即触发，
> 但在执行 SWAPGS 之前，那么唯一安全的方法是较慢的方法：RDMSR。


换句话说，例如 `NMI` 可能会在 `swapgs` 指令的临界区中发生。在这种情况下，我们应该检查 `MSR_GS_BASE` [模型特定寄存器](https://en.wikipedia.org/wiki/Model-specific_register) 的值，如果它是负值，我们来自内核空间，否则我们来自用户空间：


```assembly
movl $MSR_GS_BASE,%ecx
rdmsr
testl %edx,%edx
js 1f
```

在这段代码的头两行里，我们将 `MSR_GS_BASE` 模型特定寄存器的值读到 `edx:eax` 对中。我们不能从用户空间设置负值到 `gs` 。但是另一方面，我们知道物理内存的直接映射开始于虚拟地址 `0xffff880000000000` 。因此，`MSR_GS_BASE` 寄存器中的地址必然在 `0xffff880000000000` 到 `0xffffc7ffffffffff` 范围内。在执行 `rdmsr` 指令后，`%edx` 寄存器中的最小可能值将是 `-30720` ，这是 `4` 字节无符号值。这就是为什么指向 `per-cpu` 区域的内核空间 `gs` 会包含负值。


在我们向栈中压入伪错误代码后，我们应该为通用寄存器分配空间：

```assembly
ALLOC_PT_GPREGS_ON_STACK
```


定义在 [arch/x86/entry/calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/calling.h) 文件中的这个宏会在栈中分配 15*8 字节的空间来保留通用寄存器。

```assembly
.macro ALLOC_PT_GPREGS_ON_STACK addskip=0
    addq	$-(15*8+\addskip), %rsp
.endm
```
因此在执行了 `ALLOC_PT_GPREGS_ON_STACK` 后，栈的结构如下：

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

在我们为通用寄存器分配了空间之后，我们做一些检查，以了解异常是否来自用户空间，如果是，我们应该回到被中断的进程栈，否则留在异常栈上：

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

让我们考虑所有这些情况。

异常发生在用户空间
--------------------------------------------------------------------------------


首先我们考虑一个异常有 `paranoid=1` 的情况，比如我们的 `debug` 和 `int3` 异常。在这种情况下，我们检查 `CS` 段寄存器中的选择器，如果我们来自用户空间，则跳转到 `1f` 标签，否则将调用 `paranoid_entry` 。


让我们考虑从用户空间来到异常处理程序的第一种情况。正如我们上面所描述的，我们应该跳转到 `1` 标签。`1` 标签从调用

```assembly
call	error_entry
```

保存所有通用寄存器到之前分配的栈空间的`error_entry` 函数开始：

```assembly
SAVE_C_REGS 8
SAVE_EXTRA_REGS 8
```

这两个宏都定义在 [arch/x86/entry/calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/calling.h) 头文件中，并且只是将通用寄存器的值移动到栈中的特定位置，例如：


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
执行完 `SAVE_C_REGS` 和 `SAVE_EXTRA_REGS` 后，栈的结构如下：

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

在内核在栈中保存了通用寄存器后，我们应该再次检查我们是否来自用户空间：

```assembly
testb	$3, CS+8(%rsp)
jz	.Lerror_kernelspace
```

根据文档描述，若 `%RIP` 被截断，可能会导致潜在的故障。但无论如何，在这两种情况下，都会执行 `SWAPGS` 指令，并且从 `MSR_KERNEL_GS_BASE` 和 `MSR_GS_BASE` 中交换值。从现在开始，`%gs` 寄存器将指向内核结构的基地址。因此，`SWAPGS` 指令被调用，这是 `error_entry` 函数的核心操作。

现在我们返回到 `idtentry` 宏。在调用了 `error_entry` 后 我们可能会看到以下汇编代码：

```assembly
movq	%rsp, %rdi
call	sync_regs
```


这里我们将栈指针 `%rdi` 寄存器的基地址作为 `sync_regs` 函数的第一个参数（根据[x86_64 ABI](https://www.uclibc.org/docs/psABI-x86_64.pdf)），并且调用这个定义在 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c) 源代码文件中的函数：
```C
asmlinkage __visible notrace struct pt_regs *sync_regs(struct pt_regs *eregs)
{
	struct pt_regs *regs = task_pt_regs(current);
	*regs = *eregs;
	return regs;
}
```
这个函数获取定义在 [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h)头文件中的 `task_ptr_regs` 宏的结果，将其存储在栈指针中，并返回它。`task_ptr_regs` 宏扩展为 `thread.sp0` 的地址，它表示指向正常内核栈的指针：

```C
#define task_pt_regs(tsk)       ((struct pt_regs *)(tsk)->thread.sp0 - 1)
```
因为我们来自用户空间，这意味着异常处理程序将在真实进程上下文中运行。从 `sync_regs` 中获取栈指针后，我们切换栈：

```assembly
movq	%rax, %rsp
```

在异常处理程序调用第二级处理程序之前，最后两步是：

1. 将包含保留通用寄存器的 `pt_regs` 结构的指针传递给 `%rdi` 寄存器：

```assembly
movq	%rsp, %rdi
```
它将作为第二级异常处理程序的第一个参数传递。

2. 将错误码传递给 `%rsi` 寄存器，它将作为异常处理程序的第二个参数传递，并在栈中将其设置为 `-1`，以防止系统调用的重启：

```
.if \has_error_code
	movq	ORIG_RAX(%rsp), %rsi
	movq	$-1, ORIG_RAX(%rsp)
.else
	xorl	%esi, %esi
.endif
```

另外如果一个异常没有提供错误码时，你或许会看到我们将 `%esi` 寄存器置零。
最后我们调用第二级异常处理程序：

```assembly
call	\do_sym
```

这

```C
dotraplinkage void do_debug(struct pt_regs *regs, long error_code);
```

是 `debug` 异常的第二级异常处理程序，这

```C
dotraplinkage void notrace do_int3(struct pt_regs *regs, long error_code);
```

是`int 3` 异常的。在这部分我们不会深入第二级处理程序的实现，因为它们高度特化，但我们将在接下来的某一部分中看到其中一些。


我们刚刚考虑了异常发生在用户空间的第一种情况。现在让我们考虑最后两种情况。

发生在内核空间并且 paranoid > 0 的异常
--------------------------------------------------------------------------------

在这种情况下，异常发生在内核空间，并且 `idtentry` 宏为这个异常定义了 `paranoid=1`。这个值的 `paranoid` 表示我们应该使用我们在这部分开始时看到的较慢的方法来检查我们是否真的来自内核空间。`paranoid_entry` 函数允许我们知道这一点：

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


正如你所见，我们使用第二种（慢）方法获取一个中断任务之前状态的信息。我们检查了这一点并在从用户空间来到异常处理程序时执行了 `SWAPGS`，我们应该做我们之前做的事情：我们需要将包含通用寄存器的结构的指针传递给 `%rdi`（它将成为第二级处理程序的第一个参数），并将错误码传递给 `%rsi`（它将成为第二级处理程序的第二个参数）：

```assembly
movq	%rsp, %rdi

.if \has_error_code
	movq	ORIG_RAX(%rsp), %rsi
	movq	$-1, ORIG_RAX(%rsp)
.else
	xorl	%esi, %esi
.endif
```
在异常的第二级处理程序被调用之前，最后一步是清理新的 `IST` 栈帧：

```assembly
.if \shift_ist != -1
	subq	$EXCEPTION_STKSZ, CPU_TSS_IST(\shift_ist)
.endif
```

你也许记得我们将 `shift_ist` 作为 `idtentry` 宏的参数传递。在这里我们检查它的值，如果它不等于 `-1`，我们通过 `shift_ist` 索引从 `Interrupt Stack Table` 中获取栈指针并设置它。

在第二种方式的最后，我们只调用第二级异常处理程序，就像我们之前做的一样：

```assembly
call	\do_sym
```

最后的方法与前面两种方法类似，但当 `paranoid=0` 的异常发生时，我们可以使用快速方法确定我们来自哪里。


从异常处理程序退出
--------------------------------------------------------------------------------

在第二个处理程序完成它的工作后，我们将返回到 `idtentry` 宏，并且下一个步将跳转到 `error_exit` 函数：

```assembly
jmp	error_exit
```

`error_exit` 函数定义在相同的 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) 汇编源文件中，这个函数的主要目的是知道我们来自哪里（来自用户空间或内核空间）并根据这个执行 `SWPAGS`。恢复寄存器到之前的状态并执行 `iret` 指令以将控制权转移到一个被中断的任务。

以上就是所有内容。

结论
--------------------------------------------------------------------------------


这是关于在 Linux 内核中断和中断处理的第三部分的结尾。在之前的部分中，我们看到了 [中断描述符表](https://en.wikipedia.org/wiki/Interrupt_descriptor_table) 的初始化过程，包括 `#DB` 和 `#BP` 的门描述符设置，并且深入分析了在控制权转移到异常处理程序前的准备工作及部分中断处理程序的实现细节。在接下来的部分中，我们将继续深入这个主题，并通过 `setup_arch` 函数继续前进，并尝试理解与中断处理相关的内容。


如果你有任何建议或疑问，请在我的 [twitter](https://twitter.com/0xAX) 页面中留言或抖一抖我。

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

链接
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
* [swapgs](https://www.felixcloutier.com/x86/SWAPGS.html)
* [SIGTRAP](https://en.wikipedia.org/wiki/Unix_signal#SIGTRAP)
* [Per-CPU variables](https://0xax.gitbook.io/linux-insides/summary/concepts/linux-cpu-1)
* [kgdb](https://en.wikipedia.org/wiki/KGDB)
* [ACPI](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)
* [Previous part](https://0xax.gitbook.io/linux-insides/summary/interrupts)
