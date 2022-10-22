Linux 系统内核调用 第二节
================================================================================

Linux 内核如何处理系统调用
--------------------------------------------------------------------------------

前一[小节](http://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-1.html) 作为本章节的第一部分描述了 Linux 内核[system call](https://en.wikipedia.org/wiki/System_call) 概念。
前一节中提到通常系统调用处于内核处于操作系统层面。 前一节内容从用户空间的角度介绍，并且 [write](http://man7.org/linux/man-pages/man2/write.2.html)系统调用实现的一部分内容没有讨论。 在这一小节继续关注系统调用，在深入 Linux 内核之前，从一些理论开始。

程序中一个用户程序并不直接使用系统调用。 我们并未这样写 `Hello World`程序代码：

```C
int main(int argc, char **argv)
{
	...
	...
	...
	sys_write(fd1, buf, strlen(buf));
	...
	...
}
```

我们可以使用与 [C standard library](https://en.wikipedia.org/wiki/GNU_C_Library) 帮助类似的方式:

```C
#include <unistd.h>

int main(int argc, char **argv)
{
	...
	...
	...
	write(fd1, buf, strlen(buf));
	...
	...
}
```

不管怎样，`write` 不是直接的系统调用也不是内核函数。 程序必须将通用目的寄存器按照正确的顺序存入正确的值，之后使用 `syscall` 指令实现真正的系统调用。 在这一节我们关注 Linux 内核中，处理器执行 `syscall` 指令时的细节。

系统调用表的初始化
--------------------------------------------------------------------------------

从前一节可知系统调用与中断非常相似。 深入的说，系统调用是软件中断的处理程序。 因此，当处理器执行程序的 `syscall` 指令时，指令引起异常导致将控制权转移至异常处理。 众所周知，所有的异常处理 (或者内核 [C](https://en.wikipedia.org/wiki/C_%28programming_language%29) 函数将响应异常) 是放在内核代码中的。 但是 Linux 内核如何查找对应系统调用的系统调用处理程序的地址？ Linux 内核由一个特殊的表：`system call table`。 系统调用表是Linux内核源码文件 [arch/x86/entry/syscall_64.c](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscall_64.c) 中定义的数组 `sys_call_table` 的对应。 其实现如下:

```C
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	[0 ... __NR_syscall_max] = &sys_ni_syscall,
    #include <asm/syscalls_64.h>
};
```

 `sys_call_table` 数组的大小为 `__NR_syscall_max + 1`，`__NR_syscall_max` 宏作为给定[架构](https://en.wikipedia.org/wiki/List_of_CPU_architectures)的系统调用最大数量。 这本书关于 [x86_64](https://en.wikipedia.org/wiki/X86-64) 架构, 因此 `__NR_syscall_max` 为 `547`，这也是本书编写时(当前 Linux 内核版本为 `5.0.0-rc7`) 的数字。 编译内核时可通过 [Kbuild](https://www.kernel.org/doc/Documentation/kbuild/makefiles.txt) 产生的头文件查看该宏 - include/generated/asm-offsets.h`:

```C
#define __NR_syscall_max 547
```

对于 `x86_64`，[arch/x86/entry/syscalls/syscall_64.tbl](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L331) 中也有相同的系统调用数量。 这里存在两个重要的话题;  `sys_call_table` 数组的类型及数组中元数的初始值。 首先，`sys_call_ptr_t` 为指向系统调用表的指针。 其是通过 [typedef](https://en.wikipedia.org/wiki/Typedef) 定义的函数指针的，返回值为空且无参数：

```C
typedef void (*sys_call_ptr_t)(void);
```

其次为 `sys_call_table` 数组中元素的初始化。 从上面的代码中可知，数组中所有元素包含指向 `sys_ni_syscall` 的系统调用处理器的指针。 `sys_ni_syscall` 函数为 “not-implemented” 系统调用。 首先，`sys_call_table` 的所有元素指向 “not-implemented” 系统调用。 这是正确的初始化方法，因为我们仅仅初始化指向系统调用处理器的指针的存储位置，稍后再做处理。 `sys_ni_syscall` 的结果比较简单，仅仅返回 [-errno](http://man7.org/linux/man-pages/man3/errno.3.html) 或者 `-ENOSYS` :

```C
asmlinkage long sys_ni_syscall(void)
{
	return -ENOSYS;
}
```

The `-ENOSYS` error tells us that:

```
ENOSYS          Function not implemented (POSIX.1)
```

在 `sys_call_table` 的初始化中同时也要注意 `...`。 我们可通过 [GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection) 编译器插件 - [Designated Initializers](https://gcc.gnu.org/onlinedocs/gcc/Designated-Inits.html) 使用它。 插件允许使用不固定的顺序初始化元素。 在数组结束处，我们引用 `asm/syscalls_64.h` 头文件在。 头文件由特殊的脚本 [arch/x86/entry/syscalls/syscalltbl.sh](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscalltbl.sh) 从 [syscall table](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl) 产生。  `asm/syscalls_64.h` 包括以下宏的定义:

```C
__SYSCALL_COMMON(0, sys_read, sys_read)
__SYSCALL_COMMON(1, sys_write, sys_write)
__SYSCALL_COMMON(2, sys_open, sys_open)
__SYSCALL_COMMON(3, sys_close, sys_close)
__SYSCALL_COMMON(5, sys_newfstat, sys_newfstat)
...
...
...
```

宏 `__SYSCALL_COMMON`  在相同的[源码](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscall_64.c)中定义，作为宏 `__SYSCALL_64` 的扩展:

```C
#define __SYSCALL_COMMON(nr, sym, compat) __SYSCALL_64(nr, sym, compat)
#define __SYSCALL_64(nr, sym, compat) [nr] = sym,
```

因而，到此为止，`sys_call_table` 为如下格式:

```C
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	[0 ... __NR_syscall_max] = &sys_ni_syscall,
	[0] = sys_read,
	[1] = sys_write,
	[2] = sys_open,
	...
	...
	...
};
```

之后所有指向 “non-implemented” 系统调用元素的内容为 `sys_ni_syscall` 函数的地址，该函数仅返回 `-ENOSYS`。 其他元素指向 `sys_syscall_name` 函数。

至此，我们完成了系统调用表的填充并且 Linux 内核了解每个系统调用处理器的位置。 但是 Linux 内核在处理用户空间程序的系统调用时并未立即调用 `sys_syscall_name` 函数。 记住关于中断及中断处理的[章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/index.html)。 当 Linux 内核获得处理中断的控制权，在调用中断处理程序前，必须做一些准备如保存用户空间寄存器、切换至新的堆栈及其他很多工作。 系统调用处理也是相同的情形。 第一件事是处理系统调用的准备，但是在 Linux 内核开始这些准备之前，系统调用的入口必须完成初始化，同时只有 Linux 内核知道如何执行这些准备。 在下一章节我们将关注 Linux 内核中关于系统调用入口的初始化过程。

系统调用入口初始化
--------------------------------------------------------------------------------

当系统中发生系统调用，开始处理调用的代码的第一个字节在什么地方? 阅读 Intel 的手册 - [64-ia-32-architectures-software-developer-vol-2b-manual](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html):

```
SYSCALL 引起操作系统系统调用处理器处于特权级 0，其通过加载 IA32_LSTAR MSR 至 RIP 完成。
```

这就是说我们需要将系统调用入口放置到 `IA32_LSTAR` [model specific register](https://en.wikipedia.org/wiki/Model-specific_register)。 这一操作在 Linux 内核初始过程时完成 若你已阅读关于 Linux 内核中断及中断处理章节的[第四节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/linux-interrupts-4.html)，Linux 内核调用在初始化过程中调用 `trap_init` 函数。 该函数在 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c) 源代码文件中定义，执行 `non-early` 异常处理（如除法错误，[协处理器](https://en.wikipedia.org/wiki/Coprocessor) 错误等 ）的初始化。 除了  `non-early` 异常处理的初始化外，函数调用  [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/blob/arch/x86/kernel/cpu/common.c) 中 `cpu_init` 函数，调用相同源码文件中的 `syscall_init` 完成 `per-cpu` 状态初始化。

该函数执行系统调用入口的初始化。 查看函数的实现，函数没有参数且首先填充两个特殊模块寄存器：

```C
wrmsrl(MSR_STAR,  ((u64)__USER32_CS)<<48  | ((u64)__KERNEL_CS)<<32);
wrmsrl(MSR_LSTAR, entry_SYSCALL_64);
```

第一个特殊模块集寄存器- `MSR_STAR` 的 `63:48` 为用户代码的代码段。 这些数据将加载至 `CS` 和  `SS` 段选择符，由提供将系统调用返回至相应特权级的用户代码功能的 `sysret` 指令使用。 同时从内核代码来看，当用户空间应用程序执行系统调用时，`MSR_STAR` 的 `47:32` 将作为 `CS` and `SS` 段选择寄存器的基地址。 第二行代码中我们将使用系统调用入口 `entry_SYSCALL_64` 填充 `MSR_LSTAR` 寄存器。 `entry_SYSCALL_64` 在 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) 汇编文件中定义，包含系统调用执行前的准备(上面已经提及这些准备)。 目前不关注 `entry_SYSCALL_64`，将在章节的后续讨论。

在设置系统调用的入口之后，需要以下特殊模式寄存器：

* `MSR_CSTAR` - target `rip` for the compability mode callers;
* `MSR_IA32_SYSENTER_CS` - target `cs` for the `sysenter` instruction;
* `MSR_IA32_SYSENTER_ESP` - target `esp` for the `sysenter` instruction;
* `MSR_IA32_SYSENTER_EIP` - target `eip` for the `sysenter` instruction.

这些特殊模式寄存器的值与内核配置选项 `CONFIG_IA32_EMULATION` 有关。 若开启该内核配置选项，将允许 64 字节内核运行 32 字节的程序。 第一个例子中，若 `CONFIG_IA32_EMULATION` 内合配置选项开启，将使用兼容模式的系统调用入口填充这些特殊模式寄存器：

```C
wrmsrl(MSR_CSTAR, entry_SYSCALL_compat);
```

对于内核代码段，将堆栈指针置零，`entry_SYSENTER_compat` 字的地址写入[指令指针](https://en.wikipedia.org/wiki/Program_counter):

```C
wrmsrl_safe(MSR_IA32_SYSENTER_CS, (u64)__KERNEL_CS);
wrmsrl_safe(MSR_IA32_SYSENTER_ESP, 0ULL);
wrmsrl_safe(MSR_IA32_SYSENTER_EIP, (u64)entry_SYSENTER_compat);
```

另一方面，若 `CONFIG_IA32_EMULATION` 内核配置选项未开启，将把 `ignore_sysret` 字写入 `MSR_CSTAR`:

```C
wrmsrl(MSR_CSTAR, ignore_sysret);
```

其在 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S) 汇编文件中定义，仅返回 `-ENOSYS` 错误代码:

```assembly
ENTRY(ignore_sysret)
	mov	$-ENOSYS, %eax
	sysret
END(ignore_sysret)
```

现在需要像之前代码一样，当 `CONFIG_IA32_EMULATION` 内核配置选项打开时，填充 `MSR_IA32_SYSENTER_CS`，`MSR_IA32_SYSENTER_ESP`，`MSR_IA32_SYSENTER_EIP` 特殊模式寄存器。 而在 `CONFIG_IA32_EMULATION` 配置选项未设置的情况下，将用零填充 `MSR_IA32_SYSENTER_ESP` 和 `MSR_IA32_SYSENTER_EIP`，同时将 [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table) 的无效段加载至 `MSR_IA32_SYSENTER_CS` 特殊模式寄存器:

```C
wrmsrl_safe(MSR_IA32_SYSENTER_CS, (u64)GDT_ENTRY_INVALID_SEG);
wrmsrl_safe(MSR_IA32_SYSENTER_ESP, 0ULL);
wrmsrl_safe(MSR_IA32_SYSENTER_EIP, 0ULL);
```

可以从描述 Linux 内核启动过程的[章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-2.html)阅读更多关于 `Global Descriptor Table` 的内容。

在 `syscall_init` 函数的尾段，通过写入 `MSR_SYSCALL_MASK` 特殊寄存器的标志位，将[标志寄存器](https://en.wikipedia.org/wiki/FLAGS_register)中的标志位屏蔽：

```C
wrmsrl(MSR_SYSCALL_MASK,
	   X86_EFLAGS_TF|X86_EFLAGS_DF|X86_EFLAGS_IF|
	   X86_EFLAGS_IOPL|X86_EFLAGS_AC|X86_EFLAGS_NT);
```

这些标志位将在 syscall 初始化时清除。 至此，`syscall_init` 函数结束 也意味着系统调用已经可用。 现在我们可以开始关注当用户程序执行 `syscall` 指令发生什么。

系统调用处理执行前的准备
--------------------------------------------------------------------------------

如之前写到，系统调用或中断处理在被 Linux 内核调用前需要一些准备。 宏 `idtentry` 完成异常处理被执行前的所需准备，宏 `interrupt` 完成中断处理被调用前的所需准备，`entry_SYSCALL_64` 完成系统调用执行前的所需准备。

 `entry_SYSCALL_64` 在 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S)  汇编文件中定义，从下面的宏开始:

```assembly
SWAPGS_UNSAFE_STACK
```

该宏在 [arch/x86/include/asm/irqflags.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/irqflags.h) 头文件中定义，扩展为 `swapgs` 指令:

```C
#define SWAPGS_UNSAFE_STACK	swapgs
```

此宏将交换 GS 段选择符及 `MSR_KERNEL_GS_BASE ` 特殊模式寄存器中的值。 换句话说，将其入内核堆栈。 之后使老的堆栈指针指向 `rsp_scratch` [per-cpu](http://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html) 变量，并设置堆栈指针指向当前处理器的栈顶：

```assembly
movq	%rsp, PER_CPU_VAR(rsp_scratch)
movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp
```

下一步中将堆栈段及老的堆栈指针入栈：

```assembly
pushq	$__USER_DS
pushq	PER_CPU_VAR(rsp_scratch)
```

这之后使能中断，因为入口中断被关闭，保存通用目的[寄存器](https://en.wikipedia.org/wiki/Processor_register) (除 `bp`，`bx` 及 `r12` 至 `r15`)、标志位、“non-implemented” 系统调用相关的 `-ENOSYS` 及代码段寄存器至堆栈:

```assembly
ENABLE_INTERRUPTS(CLBR_NONE)

pushq	%r11
pushq	$__USER_CS
pushq	%rcx
pushq	%rax
pushq	%rdi
pushq	%rsi
pushq	%rdx
pushq	%rcx
pushq	$-ENOSYS
pushq	%r8
pushq	%r9
pushq	%r10
pushq	%r11
sub	$(6*8), %rsp
```

当系统调用由用户空间程序引起时，通用目的寄存器状态如下:

* `rax` - 包含系统调用编号
* `rcx` - 包含回到用户空间返回地址contains return address to the user space;
* `r11` - 包含寄存器标志
* `rdi` - 包含 system call handler 的第一个参数
* `rsi` - 包含 system call handler 的第二个参数
* `rdx` - 包含 system call handler 的第三个参数
* `r10` - 包含 system call handler 的第四个参数
* `r8`  - 包含 system call handler 的第五个参数
* `r9`  - 包含 system call handler 的第六个参数

其他通用目的寄存器 (如 `rbp`，`rbx` 和  `r12` 至 `r15`)  在 [C ABI](http://www.x86-64.org/documentation/abi.pdf) 保留)。 将寄存器标志位入栈，之后是 “non-implemented” 系统调用的用户代码段，用户空间返回地址，系统调用编号，三个参数，dump 错误代码和堆栈中的其他信息。

下一步检查当前 `thread_info` 中的 `_TIF_WORK_SYSCALL_ENTRY`：

```assembly
testl	$_TIF_WORK_SYSCALL_ENTRY, ASM_THREAD_INFO(TI_flags, %rsp, SIZEOF_PTREGS)
jnz	tracesys
```

宏 `_TIF_WORK_SYSCALL_ENTRY` 在 [arch/x86/include/asm/thread_info.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/thread_info.h) 头文件中定义，提供一系列与系统调用跟踪有关的进程信息标志：

```C
#define _TIF_WORK_SYSCALL_ENTRY \
    (_TIF_SYSCALL_TRACE | _TIF_SYSCALL_EMU | _TIF_SYSCALL_AUDIT |   \
    _TIF_SECCOMP | _TIF_SINGLESTEP | _TIF_SYSCALL_TRACEPOINT |     \
    _TIF_NOHZ)
```

本章节中不讨论追踪/调试相关内容，这将在关于 Linux 内核调试及追踪相关独立章节中讨论。 在 `tracesys` 标签之后，下一标签为 `entry_SYSCALL_64_fastpath`。 在 `entry_SYSCALL_64_fastpath` 内检查头文件 [arch/x86/include/asm/unistd.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/unistd.h) 中定义的 `__SYSCALL_MASK` 

```C
# ifdef CONFIG_X86_X32_ABI
#  define __SYSCALL_MASK (~(__X32_SYSCALL_BIT))
# else
#  define __SYSCALL_MASK (~0)
# endif
```

其中 `__X32_SYSCALL_BIT` 为：

```C
#define __X32_SYSCALL_BIT	0x40000000
```

众所周知，`__SYSCALL_MASK` 与 `CONFIG_X86_X32_ABI` 内核配置选项相关，且其为 64 位内核中 32 位 [ABI](https://en.wikipedia.org/wiki/Application_binary_interface) 的掩码。

因此我们可以检查 `__SYSCALL_MASK`，若 `CONFIG_X86_X32_ABI` 未启用，我们会将 `rax` 寄存器的值与系统调用最大数量(`__NR_syscall_max`) 进行比较，而若 `CNOFIG_X86_X32_ABI` 有启用，我们会对 `eax` 寄存器与 `X32_SYSCALL_BIT` 进行掩码操作并接着做同样的比较：

```assembly
#if __SYSCALL_MASK == ~0
	cmpq	$__NR_syscall_max, %rax
#else
	andl	$__SYSCALL_MASK, %eax
	cmpl	$__NR_syscall_max, %eax
#endif
```

至此检查最后一道比较指令的结果，`ja` 指令在 `CF` 和 `ZF` 标志为 0 时执行:

```assembly
ja	1f
```

若其正确调用系统调用，第四个参数将从 `r10` 移动至 `rcx`，保持 [x86_64 C ABI](http://www.x86-64.org/documentation/abi.pdf) 开启，同时以系统调用的处理程序的地址为参数执行 `call` 指令：

```assembly
movq	%r10, %rcx
call	*sys_call_table(, %rax, 8)
```

注意，上文提到 `sys_call_table` 是一个数组。 `rax` 通用目的寄存器为系统调用的编号，且 `sys_call_table` 的每个元素为 8 字节。 因此使用 `*sys_call_table(, %rax, 8)` 符号找到指定系统调用处理在  `sys_call_table`  中的偏移。

就这样。 完成了所需的准备，系统调用处理将被相应的中断处理调用。 例如 Linux 内核代码中 `SYSCALL_DEFINE[N]` 宏定义的 `sys_read`，`sys_write` 和其他中断处理。

退出系统调用
--------------------------------------------------------------------------------

在系统调用处理完成任务后，将退回 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S)，正好在系统调用之后:

```assembly
call	*sys_call_table(, %rax, 8)
```

在从系统调用处理返回之后，下一步是将系统调用处理的返回值入栈。 系统调用将用户程序的返回结果放置在通用目的寄存器 `rax` 中，因此在系统调用处理完成其工作后，将寄存器的值入栈上 `RAX` 指定的位置：

```C
movq	%rax, RAX(%rsp)
```

之后调用在 [arch/x86/include/asm/irqflags.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/irqflags.h) 中定义的宏 `LOCKDEP_SYS_EXIT` :

```assembly
LOCKDEP_SYS_EXIT
```

宏的实现与 `CONFIG_DEBUG_LOCK_ALLOC` 内核配置选项相关，该配置允许在退出系统调用时调试锁。 再次强调，在该章节不关注，将在单独的章节讨论相关内容。 在 `entry_SYSCALL_64` 函数的最后，恢复除 `rxc` 和 `r11` 外所有通用寄存器，因为 `rcx` 寄存器为调用系统调用的应用程序的返回地址，`r11` 寄存器为老的 [flags register](https://en.wikipedia.org/wiki/FLAGS_register)。 在恢复所有通用寄存器之后，将在 `rcx` 中装入返回地址，`r11` 寄存器装入标志，`rsp` 装入老的堆栈指针:

```assembly
RESTORE_C_REGS_EXCEPT_RCX_R11

movq	RIP(%rsp), %rcx
movq	EFLAGS(%rsp), %r11
movq	RSP(%rsp), %rsp

USERGS_SYSRET64
```

最后仅仅调用宏 `USERGS_SYSRET64`，其扩展调用 `swapgs` 指令交换用户 `GS` 和内核 `GS`，`sysretq` 指令执行从系统调用处理退出。

```C
#define USERGS_SYSRET64				\
	swapgs;	           				\
	sysretq;
```

现在我们知道，当用户程序使用系统调用时发生的一切。 整个过程的步骤如下：

* 用户程序中的代码装入通用目的寄存器的值（系统调用编号和系统调用的参数）;
* 处理器从用户模式切换到内核模式 开始执行系统调用入口 - `entry_SYSCALL_64`;
* `entry_SYSCALL_64` 切换至内核堆栈，在堆栈中存通用目的寄存器，老的堆栈，代码段，标志位等;
* `entry_SYSCALL_64` 检查 `rax` 寄存器中的系统调用编号，系统调用编号正确时，在 `sys_call_table` 中查找系统调用处理并调用;
* 若系统调用编号不正确，跳至系统调用退出;
* 系统调用处理完成工作后，恢复通用寄存器，老的堆栈，标志位及返回地址，通过 `sysretq` 指令退出`entry_SYSCALL_64`。

结论
--------------------------------------------------------------------------------

这是 Linux 内核相关概念的第二节。 在[前一节](http://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/linux-syscall-1.html)我们从用户应用程序的角度讨论了这些概念的原理。 在这一节继续深入系统调用概念的相关内容，讨论了系统调用发生时 Linux 内核执行的内容。

若存在疑问及建议，在 twitter @[0xAX](https://twitter.com/0xAX)，通过 [email](anotherworldofworld@gmail.com) 或者创建 [issue](https://github.com/MintCN/linux-insides-zh/issues/new)。

**由于英语是我的第一语言由此造成的不便深感抱歉。若发现错误请提交 PR 至 [linux-insides](https://github.com/MintCN/linux-insides-zh)。**

Links
--------------------------------------------------------------------------------

* [system call](https://en.wikipedia.org/wiki/System_call)
* [write](http://man7.org/linux/man-pages/man2/write.2.html)
* [C standard library](https://en.wikipedia.org/wiki/GNU_C_Library)
* [list of cpu architectures](https://en.wikipedia.org/wiki/List_of_CPU_architectures)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [kbuild](https://www.kernel.org/doc/Documentation/kbuild/makefiles.txt)
* [typedef](https://en.wikipedia.org/wiki/Typedef)
* [errno](http://man7.org/linux/man-pages/man3/errno.3.html)
* [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [model specific register](https://en.wikipedia.org/wiki/Model-specific_register)
* [intel 2b manual](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [coprocessor](https://en.wikipedia.org/wiki/Coprocessor)
* [instruction pointer](https://en.wikipedia.org/wiki/Program_counter)
* [flags register](https://en.wikipedia.org/wiki/FLAGS_register)
* [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)
* [per-cpu](http://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html)
* [general purpose registers](https://en.wikipedia.org/wiki/Processor_register)
* [ABI](https://en.wikipedia.org/wiki/Application_binary_interface)
* [x86_64 C ABI](http://www.x86-64.org/documentation/abi.pdf)
* [previous chapter](http://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/linux-syscall-1.html)
