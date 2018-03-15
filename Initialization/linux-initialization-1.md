内核初始化 第一部分
================================================================================

踏入内核代码的第一步（TODO: Need proofreading）
--------------------------------------------------------------------------------

[上一章](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-5.html)是[引导过程](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/index.html)的最后一部分。从现在开始，我们将深入探究 Linux 内核的初始化过程。在解压缩完 Linux 内核镜像、并把它妥善地放入内存后，内核就开始工作了。我们在第一章中介绍了 Linux 内核引导程序，它的任务就是为执行内核代码做准备。而在本章中，我们将探究内核代码，看一看内核的初始化过程——即在启动 [PID](https://en.wikipedia.org/wiki/Process_identifier) 为 `1` 的 `init` 进程前，内核所做的大量工作。

本章的内容很多，介绍了在内核启动前的所有准备工作。[arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) 文件中定义了内核入口点，我们会从这里开始，逐步地深入下去。在 `start_kernel` 函数（定义在 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c#L489)） 执行之前，我们会看到很多的初期的初始化过程，例如初期页表初始化、切换到一个新的内核空间描述符等等。

在[上一章](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/index.html)的[最后一节](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-5.html)中，我们跟踪到了 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 文件中的 [jmp](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S) 指令：

```assembly
jmp	*%rax
```

此时 `rax` 寄存器中保存的就是 Linux 内核入口点，通过调用 `decompress_kernel` （[arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c)） 函数后获得。由此可见，内核引导程序的最后一行代码是一句指向内核入口点的跳转指令。既然已经知道了内核入口点定义在哪，我们就可以继续探究 Linux 内核在引导结束后做了些什么。


内核执行的第一步
--------------------------------------------------------------------------------

OK，在调用了 `decompress_kernel` 函数后，`rax` 寄存器中保存了解压缩后的内核镜像的地址，并且跳转了过去。解压缩后的内核镜像的入口点定义在 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)，这个文件的开头几行如下：

```assembly
	__HEAD
	.code64
	.globl startup_64
startup_64:
	...
	...
	...
```

我们可以看到 `startup_64` 过程定义在了 `__HEAD` 区段下。 `__HEAD` 只是一个宏，它将展开为可执行的 `.head.text` 区段：

```C
#define __HEAD		.section	".head.text","ax"
```

我们可以在 [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S#L93) 链接器脚本文件中看到这个区段的定义：

```
.text : AT(ADDR(.text) - LOAD_OFFSET) {
	_text = .;
	...
	...
	...
} :text = 0x9090
```

除了对 `.text` 区段的定义，我们还能从这个脚本文件中得知内核的默认物理地址与虚拟地址。`_text` 是一个地址计数器，对于 [x86_64](https://en.wikipedia.org/wiki/X86-64) 来说，它定义为：

```
. = __START_KERNEL;
```

`__START_KERNEL` 宏的定义在 [arch/x86/include/asm/page_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_types.h) 头文件中，它由内核映射的虚拟基址与基物理起始点相加得到：

```C
#define _START_KERNEL	(__START_KERNEL_map + __PHYSICAL_START)

#define __PHYSICAL_START  ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN)
```

换句话说：

* Linux 内核的物理基址 - `0x1000000`;
* Linux 内核的虚拟基址 - `0xffffffff81000000`.

现在我们知道了 `startup_64` 过程的默认物理地址与虚拟地址，但是真正的地址必须要通过下面的代码计算得到：

```assembly
	leaq	_text(%rip), %rbp
	subq	$_text - __START_KERNEL_map, %rbp
```

没错，虽然定义为 `0x1000000`，但是仍然有可能变化，例如启用 [kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization#Linux) 的时候。所以我们当前的目标是计算 `0x1000000` 与实际加载地址的差。这里我们首先将RIP相对地址（`rip-relative`）放入 `rbp` 寄存器，并且从中减去 `$_text - __START_KERNEL_map` 。我们已经知道， `_text` 在编译后的默认虚拟地址为 `0xffffffff81000000`， 物理地址为 `0x1000000`。`__START_KERNEL_map` 宏将展开为 `0xffffffff80000000`，因此对于对于第二行汇编代码，我们将得到如下的表达式：

```
rbp = 0x1000000 - (0xffffffff81000000 - 0xffffffff80000000)
```

在计算过后，`rbp` 的值将为 `0`，代表了实际加载地址与编译后的默认地址之间的差值。在我们这个例子中，`0` 代表了 Linux 内核被加载到了默认地址，并且没有启用 [kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization#Linux) 。

在得到了 `startup_64` 的地址后，我们需要检查这个地址是否已经正确对齐。下面的代码将进行这项工作：

```assembly
	testl	$~PMD_PAGE_MASK, %ebp
	jnz	bad_address
```

在这里我们将 `rbp` 寄存器的低32位与 `PMD_PAGE_MASK` 进行比较。`PMD_PAGE_MASK` 代表中层页目录（`Page middle directory`）屏蔽位（相关信息请阅读 [paging](http://xinqiu.gitbooks.io/linux-insides-cn/content/Theory/linux-theory-1.html) 一节），它的定义如下：

```C
#define PMD_PAGE_MASK           (~(PMD_PAGE_SIZE-1))

#define PMD_PAGE_SIZE           (_AC(1, UL) << PMD_SHIFT)
#define PMD_SHIFT       21
```

可以很容易得出 `PMD_PAGE_SIZE` 为 `2MB` 。在这里我们使用标准公式来检查对齐问题，如果 `text` 的地址没有对齐到 `2MB`，则跳转到 `bad_address`。

在此之后，我们通过检查高 `18` 位来防止这个地址过大：

```assembly
	leaq	_text(%rip), %rax
	shrq	$MAX_PHYSMEM_BITS, %rax
	jnz	bad_address
```

这个地址必须不超过 `46` 个比特，即小于2的46次方：

```C
#define MAX_PHYSMEM_BITS       46
```

OK，至此我们完成了一些初步的检查，可以继续进行后续的工作了。


修正页表基地址
--------------------------------------------------------------------------------

在开始设置 Identity 分页之前，我们需要首先修正下面的地址：

```assembly
	addq	%rbp, early_level4_pgt + (L4_START_KERNEL*8)(%rip)
	addq	%rbp, level3_kernel_pgt + (510*8)(%rip)
	addq	%rbp, level3_kernel_pgt + (511*8)(%rip)
	addq	%rbp, level2_fixmap_pgt + (506*8)(%rip)
```

如果 `startup_64` 的值不为默认的 `0x1000000` 的话， 则包括 `early_level4_pgt`、`level3_kernel_pgt` 在内的很多地址都会不正确。`rbp`寄存器中包含的是相对地址，因此我们把它与 `early_level4_pgt`、`level3_kernel_pgt` 以及  `level2_fixmap_pgt` 中特定的项相加。首先我们来看一下它们的定义：

```assembly
NEXT_PAGE(early_level4_pgt)
	.fill	511,8,0
	.quad	level3_kernel_pgt - __START_KERNEL_map + _PAGE_TABLE

NEXT_PAGE(level3_kernel_pgt)
	.fill	L3_START_KERNEL,8,0
	.quad	level2_kernel_pgt - __START_KERNEL_map + _KERNPG_TABLE
	.quad	level2_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE

NEXT_PAGE(level2_kernel_pgt)
	PMDS(0, __PAGE_KERNEL_LARGE_EXEC,
		KERNEL_IMAGE_SIZE/PMD_SIZE)

NEXT_PAGE(level2_fixmap_pgt)
	.fill	506,8,0
	.quad	level1_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE
	.fill	5,8,0

NEXT_PAGE(level1_fixmap_pgt)
	.fill	512,8,0
```

看起来很难理解，实则不然。首先我们来看一下 `early_level4_pgt`。它的前 (4096 - 8) 个字节全为 `0`，即它的前 `511` 个项均不使用，之后的一项是 `level3_kernel_pgt - __START_KERNEL_map + _PAGE_TABLE`。我们知道 `__START_KERNEL_map` 是内核的虚拟基地址，因此减去 `__START_KERNEL_map` 后就得到了 `level3_kernel_pgt` 的物理地址。现在我们来看一下 `_PAGE_TABLE`，它是页表项的访问权限：

```C
#define _PAGE_TABLE     (_PAGE_PRESENT | _PAGE_RW | _PAGE_USER | \
                         _PAGE_ACCESSED | _PAGE_DIRTY)
```

更多信息请阅读 [分页](http://xinqiu.gitbooks.io/linux-insides-cn/content/Theory/linux-theory-1.html) 部分.

`level3_kernel_pgt` 中保存的两项用来映射内核空间，在它的前 `510`（即 `L3_START_KERNEL`）项均为 `0`。这里的 `L3_START_KERNEL` 保存的是在上层页目录（Page Upper Directory）中包含`__START_KERNEL_map` 地址的那一条索引，它等于 `510`。后面一项 `level2_kernel_pgt - __START_KERNEL_map + _KERNPG_TABLE` 中的 `level2_kernel_pgt` 比较容易理解，它是一条页表项，包含了指向中层页目录的指针，它用来映射内核空间，并且具有如下的访问权限：

```C
#define _KERNPG_TABLE   (_PAGE_PRESENT | _PAGE_RW | _PAGE_ACCESSED | \
                         _PAGE_DIRTY)
```

`level2_fixmap_pgt` 是一系列虚拟地址，它们可以在内核空间中指向任意的物理地址。它们由`level2_fixmap_pgt`作为入口点、`10`MB 大小的空间用来为 [vsyscalls](https://lwn.net/Articles/446528/) 做映射。`level2_kernel_pgt` 则调用了`PDMS` 宏，在 `__START_KERNEL_map` 地址处为内核的 `.text` 创建了 `512`MB 大小的空间（这 `512` MB空间的后面是模块内存空间）。

现在，在看过了这些符号的定义之后，让我们回到本节开始时介绍的那几行代码。`rbp` 寄存器包含了实际地址与 `startup_64` 地址之差，其中 `startup_64` 的地址是在内核[链接](https://en.wikipedia.org/wiki/Linker_%28computing%29)时获得的。因此我们只需要把它与各个页表项的基地址相加，就能够得到正确的地址了。在这里这些操作如下：

```assembly
	addq	%rbp, early_level4_pgt + (L4_START_KERNEL*8)(%rip)
	addq	%rbp, level3_kernel_pgt + (510*8)(%rip)
	addq	%rbp, level3_kernel_pgt + (511*8)(%rip)
	addq	%rbp, level2_fixmap_pgt + (506*8)(%rip)
```

换句话说，`early_level4_pgt` 的最后一项就是 `level3_kernel_pgt`，`level3_kernel_pgt` 的最后两项分别是 `level2_kernel_pgt` 和 `level2_fixmap_pgt`， `level2_fixmap_pgt` 的第507项就是 `level1_fixmap_pgt` 页目录。

在这之后我们就得到了：

```
early_level4_pgt[511] -> level3_kernel_pgt[0]
level3_kernel_pgt[510] -> level2_kernel_pgt[0]
level3_kernel_pgt[511] -> level2_fixmap_pgt[0]
level2_kernel_pgt[0]   -> 512 MB kernel mapping
level2_fixmap_pgt[507] -> level1_fixmap_pgt
```

需要注意的是，我们并不修正 `early_level4_pgt` 以及其他页目录的基地址，我们会在构造、填充这些页目录结构的时候修正。我们修正了页表基地址后，就可以开始构造这些页目录了。

Identity Map Paging
--------------------------------------------------------------------------------

现在我们可以进入到对初期页表进行 Identity 映射的初始化过程了。在 Identity 映射分页中，虚拟地址会被映射到地址相同的物理地址上，即 `1 : 1`。下面我们来看一下细节。首先我们找到 `_text` 与 `_early_level4_pgt` 的 RIP 相对地址，并把他们放入 `rdi` 与 `rbx` 寄存器中。

```assembly
	leaq	_text(%rip), %rdi
	leaq	early_level4_pgt(%rip), %rbx
```

在此之后我们使用 `rax` 保存 `_text` 的地址。同时，在全局页目录表中有一条记录中存放的是 `_text` 的地址。为了得到这条索引，我们把 `_text` 的地址右移 `PGDIR_SHIFT` 位。

```assembly
	movq	%rdi, %rax
	shrq	$PGDIR_SHIFT, %rax

	leaq	(4096 + _KERNPG_TABLE)(%rbx), %rdx
	movq	%rdx, 0(%rbx,%rax,8)
	movq	%rdx, 8(%rbx,%rax,8)
```

其中 `PGDIR_SHIFT` 为 `39`。`PGDIR_SHIFT`表示的是在虚拟地址下的全局页目录位的屏蔽值（mask）。下面的宏定义了所有类型的页目录的屏蔽值：


```C
#define PGDIR_SHIFT     39
#define PUD_SHIFT       30
#define PMD_SHIFT       21
```

此后我们就将 `level3_kernel_pgt` 的地址放进 `rdx` 中，并将它的访问权限设置为 `_KERNPG_TABLE`（见上），然后将 `level3_kernel_pgt` 填入 `early_level4_pgt` 的两项中。

然后我们给 `rdx` 寄存器加上 `4096`（即 `early_level4_pgt` 的大小），并把 `rdi` 寄存器的值（即 `_text` 的物理地址）赋值给 `rax` 寄存器。之后我们把上层页目录中的两个项写入 `level3_kernel_pgt`：

```assembly
	addq	$4096, %rdx
	movq	%rdi, %rax
	shrq	$PUD_SHIFT, %rax
	andl	$(PTRS_PER_PUD-1), %eax
	movq	%rdx, 4096(%rbx,%rax,8)
	incl	%eax
	andl	$(PTRS_PER_PUD-1), %eax
	movq	%rdx, 4096(%rbx,%rax,8)
```

下一步我们把中层页目录表项的地址写入 `level2_kernel_pgt`，然后修正内核的 text 和 data 的虚拟地址：

```assembly
	leaq	level2_kernel_pgt(%rip), %rdi
	leaq	4096(%rdi), %r8
1:	testq	$1, 0(%rdi)
	jz	2f
	addq	%rbp, 0(%rdi)
2:	addq	$8, %rdi
	cmp	%r8, %rdi
	jne	1b
```

这里首先把 `level2_kernel_pgt` 的地址赋值给 `rdi`，并把页表项的地址赋值给 `r8` 寄存器。下一步我们来检查 `level2_kernel_pgt` 中的存在位，如果其为0，就把 `rdi` 加上8以便指向下一个页。然后我们将其与 `r8`（即页表项的地址）作比较，不相等的话就跳转回前面的标签 `1` ，反之则继续运行。

接下来我们使用 `rbp` （即 `_text` 的物理地址）来修正 `phys_base` 物理地址。将 `early_level4_pgt` 的物理地址与 `rbp` 相加，然后跳转至标签 `1`：

```assembly
	addq	%rbp, phys_base(%rip)
	movq	$(early_level4_pgt - __START_KERNEL_map), %rax
	jmp 1f
```

其中 `phys_base` 与 `level2_kernel_pgt` 第一项相同，为 `512` MB的内核映射。

跳转至内核入口点之前的最后准备
--------------------------------------------------------------------------------

此后我们就跳转至标签`1`来开启 `PAE` 和 `PGE` （Paging Global Extension），并且将`phys_base`的物理地址（见上）放入 `rax` 就寄存器，同时将其放入 `cr3` 寄存器：

```assembly
1:
	movl	$(X86_CR4_PAE | X86_CR4_PGE), %ecx
	movq	%rcx, %cr4

	addq	phys_base(%rip), %rax
	movq	%rax, %cr3
```

接下来我们检查CPU是否支持 [NX](http://en.wikipedia.org/wiki/NX_bit) 位：


```assembly
	movl	$0x80000001, %eax
	cpuid
	movl	%edx,%edi
```

首先将 `0x80000001` 放入 `eax` 中，然后执行 `cpuid` 指令来得到处理器信息。这条指令的结果会存放在 `edx` 中，我们把他再放到 `edi` 里。

现在我们把 `MSR_EFER` （即 `0xc0000080`）放入 `ecx`，然后执行 `rdmsr` 指令来读取CPU中的Model Specific Register (MSR)。


```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
```

返回结果将存放于 `edx:eax` 。下面展示了 `EFER` 各个位的含义：


```
63                                                                              32
 --------------------------------------------------------------------------------
|                                                                               |
|                                Reserved MBZ                                   |
|                                                                               |
 --------------------------------------------------------------------------------
31                            16  15      14      13   12  11   10  9  8 7  1   0
 --------------------------------------------------------------------------------
|                              | T |       |       |    |   |   |   |   |   |   |
| Reserved MBZ                 | C | FFXSR | LMSLE |SVME|NXE|LMA|MBZ|LME|RAZ|SCE|
|                              | E |       |       |    |   |   |   |   |   |   |
 --------------------------------------------------------------------------------
```

在这里我们不会介绍每一个位的含义，没有涉及到的位和其他的 MSR 将会在专门的部分介绍。在我们将 `EFER` 读入 `edx:eax` 之后，通过 `btsl` 来将 `_EFER_SCE` （即第0位）置1，设置 `SCE` 位将会启用 `SYSCALL` 以及 `SYSRET` 指令。下一步我们检查 `edi`（即 `cpuid` 的结果（见上）） 中的第20位。如果第 `20` 位（即 `NX` 位）置位，我们就只把 `EFER_SCE`写入MSR。

```assembly
	btsl	$_EFER_SCE, %eax
	btl	    $20,%edi
	jnc     1f
	btsl	$_EFER_NX, %eax
	btsq	$_PAGE_BIT_NX,early_pmd_flags(%rip)
1:	wrmsr
```

如果支持 [NX](https://en.wikipedia.org/wiki/NX_bit) 那么我们就把 `_EFER_NX` 也写入MSR。在设置了 [NX](https://en.wikipedia.org/wiki/NX_bit) 后，还要对 `cr0` （[control register](https://en.wikipedia.org/wiki/Control_register)） 中的一些位进行设置：


* `X86_CR0_PE` - 系统处于保护模式;
* `X86_CR0_MP` - 与CR0的TS标志位一同控制 WAIT/FWAIT 指令的功能；
* `X86_CR0_ET` - 386允许指定外部数学协处理器为80287或80387;
* `X86_CR0_NE` - 如果置位，则启用内置的x87浮点错误报告，否则启用PC风格的x87错误检测；
* `X86_CR0_WP` - 如果置位，则CPU在特权等级为0时无法写入只读内存页;
* `X86_CR0_AM` - 当AM位置位、EFLGS中的AC位置位、特权等级为3时，进行对齐检查;
* `X86_CR0_PG` - 启用分页.

```assembly
#define CR0_STATE	(X86_CR0_PE | X86_CR0_MP | X86_CR0_ET | \
			 X86_CR0_NE | X86_CR0_WP | X86_CR0_AM | \
			 X86_CR0_PG)
movl	$CR0_STATE, %eax
movq	%rax, %cr0
```

为了从汇编执行[C语言](https://en.wikipedia.org/wiki/C_%28programming_language%29)代码，我们需要建立一个栈。首先将[栈指针](https://en.wikipedia.org/wiki/Stack_register) 指向一个内存中合适的区域，然后重置[FLAGS寄存器](https://en.wikipedia.org/wiki/FLAGS_register)

```assembly
movq stack_start(%rip), %rsp
pushq $0
popfq
```

在这里最有意思的地方在于 `stack_start`。它也定义在[当前的源文件](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)中：

```assembly
GLOBAL(stack_start)
.quad  init_thread_union+THREAD_SIZE-8
```

对于 `GLOABL` 我们应该很熟悉了。它在 [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/linkage.h) 头文件中定义如下：

```C
#define GLOBAL(name)    \
         .globl name;           \
         name:
```

`THREAD_SIZE` 定义在 [arch/x86/include/asm/page_64_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_64_types.h)，它依赖于 `KASAN_STACK_ORDER` 的值:

```C
#define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

首先来考虑当禁用了 [kasan](http://lxr.free-electrons.com/source/Documentation/kasan.txt) 并且 `PAGE_SIZE` 大小为4096时的情况。此时 `THREAD_SIZE` 将为 `16` KB，代表了一个线程的栈的大小。为什么是`线程`？我们知道每一个[进程](https://en.wikipedia.org/wiki/Process_%28computing%29)可能会有[父进程](https://en.wikipedia.org/wiki/Parent_process)和[子进程](https://en.wikipedia.org/wiki/Child_process)。事实上，父进程和子进程使用不同的栈空间，每一个新进程都会拥有一个新的内核栈。在Linux内核中，这个栈由 `thread_info` 结构中的一个[union](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B)表示：


```C
union thread_union {
         struct thread_info thread_info;
         unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

例如，`init_thread_union`定义如下：

```C
union thread_union init_thread_union __init_task_data =
	{ INIT_THREAD_INFO(init_task) };
```

其中 `INIT_THREAD_INFO` 接受 `task_struct` 结构类型的参数，并进行一些初始化操作：

```C
#define INIT_THREAD_INFO(tsk)		\
{                                               \
	.task		= &tsk,                         \
	.flags		= 0,                            \
	.cpu		= 0,                            \
	.addr_limit	= KERNEL_DS,                    \
}
```

`task_struct` 结构在内核中代表了对进程的描述。因此，`thread_union` 包含了关于一个进程的低级信息，并且其位于进程栈底：

```
+-----------------------+
|                       |
|                       |
|                       |
|     Kernel stack      |
|                       |
|                       |
|                       |
|-----------------------|
|                       |
|  struct thread_info   |
|                       |
+-----------------------+
```

需要注意的是我们在栈顶保留了 `8` 个字节的空间，用来保护对下一个内存页的非法访问。

在初期启动栈设置好之后，使用 `lgdt` 指令来更新[全局描述符表](https://en.wikipedia.org/wiki/Global_Descriptor_Table)：

```assembly
lgdt	early_gdt_descr(%rip)
```

其中 `early_gdt_descr` 定义如下：

```assembly
early_gdt_descr:
	.word	GDT_ENTRIES*8-1
early_gdt_descr_base:
	.quad	INIT_PER_CPU_VAR(gdt_page)
```

需要重新加载 `全局描述附表` 的原因是，虽然目前内核工作在用户空间的低地址中，但很快内核将会在它自己的内存地址空间中运行。下面让我们来看一下 `early_gdt_descr` 的定义。全局描述符表包含了32项，用于内核代码、数据、线程局部存储段等：

```C
#define GDT_ENTRIES 32
```

现在来看一下 `early_gdt_descr_base`. 首先，`gdt_page` 的定义在[arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h)中:

```C
struct gdt_page {
	struct desc_struct gdt[GDT_ENTRIES];
} __attribute__((aligned(PAGE_SIZE)));
```

它只包含了一项 `desc_struct` 的数组`gdt`。`desc_struct`定义如下:

```C
struct desc_struct {
         union {
                 struct {
                         unsigned int a;
                         unsigned int b;
                 };
                 struct {
                         u16 limit0;
                         u16 base0;
                         unsigned base1: 8, type: 4, s: 1, dpl: 2, p: 1;
                         unsigned limit: 4, avl: 1, l: 1, d: 1, g: 1, base2: 8;
                 };
         };
 } __attribute__((packed));
```

它跟 `GDT` 描述符的定义很像。同时需要注意的是，`gdt_page`结构是 `PAGE_SIZE`(` 4096`) 对齐的，即 `gdt` 将会占用一页内存。

下面我们来看一下 `INIT_PER_CPU_VAR`，它定义在 [arch/x86/include/asm/percpu.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/percpu.h)，只是将给定的参数与 `init_per_cpu__`连接起来：

```C
#define INIT_PER_CPU_VAR(var) init_per_cpu__##var
```

所以在宏展开之后，我们会得到 `init_per_cpu__gdt_page`。而在 [linker script](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S) 中可以发现:

```
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(gdt_page);
```

`INIT_PER_CPU` 扩展后也将得到 `init_per_cpu__gdt_page` 并将它的值设置为相对于 `__per_cpu_load` 的偏移量。这样，我们就得到了新GDT的正确的基地址。

per-CPU变量是2.6内核中的特性。顾名思义，当我们创建一个 `per-CPU` 变量时，每个CPU都会拥有一份它自己的拷贝，在这里我们创建的是 `gdt_page` per-CPU变量。这种类型的变量有很多有点，比如由于每个CPU都只访问自己的变量而不需要锁等。因此在多处理器的情况下，每一个处理器核心都将拥有一份自己的 `GDT` 表，其中的每一项都代表了一块内存，这块内存可以由在这个核心上运行的线程访问。这里 [Concepts/per-cpu](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html) 有关于 `per-CPU` 变量的更详细的介绍。

在加载好了新的全局描述附表之后，跟之前一样我们重新加载一下各个段：

```assembly
	xorl %eax,%eax
	movl %eax,%ds
	movl %eax,%ss
	movl %eax,%es
	movl %eax,%fs
	movl %eax,%gs
```

在所有这些步骤都结束后，我们需要设置一下 `gs` 寄存器，令它指向一个特殊的栈 `irqstack`，用于处理[中断](https://en.wikipedia.org/wiki/Interrupt)：

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr
```

其中， `MSR_GS_BASE` 为：

```C
#define MSR_GS_BASE             0xc0000101
```

我们需要把 `MSR_GS_BASE` 放入 `ecx` 寄存器，同时利用 `wrmsr` 指令向 `eax` 和 `edx` 处的地址加载数据（即指向 `initial_gs`）。`cs`, `fs`, `ds` 和 `ss` 段寄存器在64位模式下不用来寻址，但 `fs` 和 `gs` 可以使用。 `fs` 和 `gs` 有一个隐含的部分（与实模式下的 `cs` 段寄存器类似），这个隐含部分存储了一个描述符，其指向 [Model Specific Registers](https://en.wikipedia.org/wiki/Model-specific_register)。因此上面的 `0xc0000101` 是一个 `gs.base` MSR 地址。当发生[系统调用](https://en.wikipedia.org/wiki/System_call) 或者 [中断](https://en.wikipedia.org/wiki/Interrupt)时，入口点处并没有内核栈，因此 `MSR_GS_BASE` 将会用来存放中断栈。

接下来我们把实模式中的 bootparam 结构的地址放入 `rdi` (要记得 `rsi` 从一开始就保存了这个结构体的指针)，然后跳转到C语言代码：

```assembly
	movq	initial_code(%rip),%rax
	pushq	$0
	pushq	$__KERNEL_CS
	pushq	%rax
	lretq
```

这里我们把 `initial_code` 放入 `rax` 中，并且向栈里分别压入一个无用的地址、`__KERNEL_CS` 和 `initial_code` 的地址。随后的 `lreq` 指令表示从栈上弹出返回地址并跳转。`initial_code` 同样定义在这个文件里：

```assembly
	.balign	8
	GLOBAL(initial_code)
	.quad	x86_64_start_kernel
	...
	...
	...
```

可以看到 `initial_code` 包含了 `x86_64_start_kernel` 的地址，其定义在 [arch/x86/kerne/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c)：

```C
asmlinkage __visible void __init x86_64_start_kernel(char * real_mode_data) {
	...
	...
	...
}
```

这个函数接受一个参数 `real_mode_data`（刚才我们把实模式下数据的地址保存到了 `rdi` 寄存器中）。

这个函数是内核中第一个执行的C语言代码！

走进 start_kernel
--------------------------------------------------------------------------------

在我们真正到达“内核入口点”-[init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c#L489)中的start_kernel函数之前，我们还需要最后的准备工作：

首先在 `x86_64_start_kernel` 函数中可以看到一些检查工作：

```C
BUILD_BUG_ON(MODULES_VADDR < __START_KERNEL_map);
BUILD_BUG_ON(MODULES_VADDR - __START_KERNEL_map < KERNEL_IMAGE_SIZE);
BUILD_BUG_ON(MODULES_LEN + KERNEL_IMAGE_SIZE > 2*PUD_SIZE);
BUILD_BUG_ON((__START_KERNEL_map & ~PMD_MASK) != 0);
BUILD_BUG_ON((MODULES_VADDR & ~PMD_MASK) != 0);
BUILD_BUG_ON(!(MODULES_VADDR > __START_KERNEL));
BUILD_BUG_ON(!(((MODULES_END - 1) & PGDIR_MASK) == (__START_KERNEL & PGDIR_MASK)));
BUILD_BUG_ON(__fix_to_virt(__end_of_fixed_addresses) <= MODULES_END);
```

这些检查包括：模块的虚拟地址不能低于内核 text 段基地址 `__START_KERNEL_map` ，包含模块的内核 text 段的空间大小不能小于内核镜像大小等等。`BUILD_BUG_ON` 宏定义如下：

```C
#define BUILD_BUG_ON(condition) ((void)sizeof(char[1 - 2*!!(condition)]))
```

我们来理解一下这些巧妙的设计是怎么工作的。首先以第一个条件 `MODULES_VADDR < __START_KERNEL_map` 为例：`!!conditions` 等价于 `condition != 0`，这代表如果 `MODULES_VADDR < __START_KERNEL_map` 为真，则 `!!(condition)` 为1，否则为0。执行`2*!!(condition)`之后数值变为 `2` 或 `0`。因此，这个宏执行完后可能产生两种不同的行为：

* 编译错误。因为我们尝试取获取一个字符数组索引为负数的变量的大小。
* 没有编译错误。

就是这么简单，通过C语言中某些常量导致编译错误的技巧实现了这一设计。

接下来 start_kernel 调用了 `cr4_init_shadow` 函数，其中存储了每个CPU中 `cr4` 的Shadow Copy。上下文切换可能会修改 `cr4` 中的位，因此需要保存每个CPU中 `cr4` 的内容。在这之后将会调用 `reset_early_page_tables` 函数，它重置了所有的全局页目录项，同时向 `cr3` 中重新写入了的全局页目录表的地址：

```C
for (i = 0; i < PTRS_PER_PGD-1; i++)
	early_level4_pgt[i].pgd = 0;

next_early_pgt = 0;

write_cr3(__pa_nodebug(early_level4_pgt));
```

很快我们就会设置新的页表。在这里我们遍历了所有的全局页目录项（其中 `PTRS_PER_PGD` 为 `512`），将其设置为0。之后将 `next_early_pgt` 设置为0（会在下一篇文章中介绍细节），同时把 `early_level4_pgt` 的物理地址写入 `cr3`。`__pa_nodebug` 是一个宏，将被扩展为：

```C
((unsigned long)(x) - __START_KERNEL_map + phys_base)
```

此后我们清空了从 `__bss_stop` 到 `__bss_start` 的 `_bss` 段，下一步将是建立初期 `IDT（中断描述符表）` 的处理代码，内容很多，我们将会留到下一个部分再来探究。

总结
--------------------------------------------------------------------------------

第一部分关于Linux内核的初始化过程到这里就结束了。

如果你有任何问题或建议，请在twitter上联系我 [0xAX](https://twitter.com/0xAX)，或者通过[邮件](anotherworldofworld@gmail.com)与我沟通，还可以新开[issue](https://github.com/MintCN/linux-insides-zh/issues/new)。

下一部分我们会看到初期中断处理程序的初始化过程、内核空间的内存映射等。


相关链接
--------------------------------------------------------------------------------

* [Model Specific Register](http://en.wikipedia.org/wiki/Model-specific_register)
* [Paging](http://xinqiu.gitbooks.io/linux-insides-cn/content/Theory/linux-theory-1.html)
* [Previous part - Kernel decompression](http://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-5.html)
* [NX](http://en.wikipedia.org/wiki/NX_bit)
* [ASLR](http://en.wikipedia.org/wiki/Address_space_layout_randomization)
