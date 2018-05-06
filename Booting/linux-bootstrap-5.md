内核引导过程. Part 5.
================================================================================

内核解压
--------------------------------------------------------------------------------

这是`内核引导过程`系列文章的第五部分。在[前一部分](linux-bootstrap-4.md#transition-to-the-long-mode)我们看到了切换到64位模式的过程，在这一部分我们会从这里继续。我们会看到跳进内核代码的最后步骤：内核解压前的准备、重定位和直接内核解压。所以...让我们再次深入内核源码。

内核解压前的准备
--------------------------------------------------------------------------------

我们停在了跳转到`64位`入口点——`startup_64`的跳转之前，它在源文件 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S) 里面。在之前的部分，我们已经在`startup_32`里面看到了到`startup_64`的跳转：

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
	...
	...
	...
	pushl	%eax
	...
	...
	...
	lret
```

由于我们加载了新的`全局描述符表`并且在其他模式有CPU的模式转换（在我们这里是`64位`模式），我们可以在`startup_64`的开头看到数据段的建立：

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
	xorl	%eax, %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
	movl	%eax, %fs
	movl	%eax, %gs
```

除`cs`之外的段寄存器在我们进入`长模式`时已经重置。

下一步是计算内核编译时的位置和它被加载的位置的差：

```assembly
#ifdef CONFIG_RELOCATABLE
	leaq	startup_32(%rip), %rbp
	movl	BP_kernel_alignment(%rsi), %eax
	decl	%eax
	addq	%rax, %rbp
	notq	%rax
	andq	%rax, %rbp
	cmpq	$LOAD_PHYSICAL_ADDR, %rbp
	jge	1f
#endif
	movq	$LOAD_PHYSICAL_ADDR, %rbp
1:
	movl	BP_init_size(%rsi), %ebx
	subl	$_end, %ebx
	addq	%rbp, %rbx
```

`rbp`包含了解压后内核的起始地址，在这段代码执行之后`rbx`会包含用于解压的重定位内核代码的地址。我们已经在`startup_32`看到类似的代码（你可以看之前的部分[计算重定位地址](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-4.md#calculate-relocation-address)），但是我们需要再做这个计算，因为引导加载器可以用64位引导协议，而`startup_32`在这种情况下不会执行。

下一步，我们可以看到栈指针的设置和标志寄存器的重置：

```assembly
	leaq	boot_stack_end(%rbx), %rsp

	pushq	$0
	popfq
```

如上所述，`rbx`寄存器包含了内核解压代码的起始地址，我们把这个地址的`boot_stack_entry`偏移地址相加放到表示栈顶指针的`rsp`寄存器。在这一步之后，栈就是正确的。你可以在汇编源码文件 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S) 的末尾找到`boot_stack_end`的定义：

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

它在`.bss`节的末尾，就在`.pgtable`前面。如果你查看 [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/vmlinux.lds.S) 链接脚本，你会找到`.bss`和`.pgtable`的定义。

由于我们设置了栈，在我们计算了解压了的内核的重定位地址后，我们可以复制压缩了的内核到以上地址。在查看细节之前，我们先看这段汇编代码：

```assembly
	pushq	%rsi
	leaq	(_bss-8)(%rip), %rsi
	leaq	(_bss-8)(%rbx), %rdi
	movq	$_bss, %rcx
	shrq	$3, %rcx
	std
	rep	movsq
	cld
	popq	%rsi
```

首先我们把`rsi`压进栈。我们需要保存`rsi`的值，因为这个寄存器现在存放指向`boot_params`的指针，这是包含引导相关数据的实模式结构体（你一定记得这个结构体，我们在开始设置内核的时候就填充了它）。在代码的结尾，我们会重新恢复指向`boot_params`的指针到`rsi`.

接下来两个`leaq`指令用`_bss - 8`偏移和`rip`和`rbx`计算有效地址并存放到`rsi`和`rdi`. 我们为什么要计算这些地址？实际上，压缩了的代码镜像存放在这份复制了的代码（从`startup_32`到当前的代码）和解压了的代码之间。你可以通过查看链接脚本 [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/vmlinux.lds.S) 验证：

```
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
	}
	.rodata..compressed : {
		*(.rodata..compressed)
	}
	.text :	{
		_text = .; 	/* Text */
		*(.text)
		*(.text.*)
		_etext = . ;
	}
```

注意`.head.text`节包含了`startup_32`. 你可以从之前的部分回忆起它：

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
...
...
...
```

`.text`节包含解压代码：

```assembly
	.text
relocated:
...
...
...
/*
 * Do the decompression, and jump to the new kernel..
 */
...
```

`.rodata..compressed`包含了压缩了的内核镜像。所以`rsi`包含`_bss - 8`的绝对地址，`rdi`包含`_bss - 8`的重定位的相对地址。在我们把这些地址放入寄存器时，我们把`_bss`的地址放到了`rcx`寄存器。正如你在`vmlinux.lds.S`链接脚本中看到了一样，它和设置/内核代码一起在所有节的末尾。现在我们可以开始用`movsq`指令每次8字节地从`rsi`到`rdi`复制代码。

注意在数据复制前有`std`指令：它设置`DF`标志，意味着`rsi`和`rdi`会递减。换句话说，我们会从后往前复制这些字节。最后，我们用`cld`指令清除`DF`标志，并恢复`boot_params`到`rsi`.

现在我们有`.text`节的重定位后的地址，我们可以跳到那里：

```assembly
	leaq	relocated(%rbx), %rax
	jmp	*%rax
```

在内核解压前的最后准备
--------------------------------------------------------------------------------

在上一段我们看到了`.text`节从`relocated`标签开始。它做的第一件事是清空`.bss`节：

```assembly
	xorl	%eax, %eax
	leaq    _bss(%rip), %rdi
	leaq    _ebss(%rip), %rcx
	subq	%rdi, %rcx
	shrq	$3, %rcx
	rep	stosq
```

我们要初始化`.bss`节，因为我们很快要跳转到[C](https://en.wikipedia.org/wiki/C_%28programming_language%29)代码。这里我们就清空`eax`，把`_bss`的地址放到`rdi`，把`_ebss`放到`rcx`，然后用`rep stosq`填零。

最后，我们可以调用`extract_kernel`函数：

```assembly
	pushq	%rsi
	movq	%rsi, %rdi
	leaq	boot_heap(%rip), %rsi
	leaq	input_data(%rip), %rdx
	movl	$z_input_len, %ecx
	movq	%rbp, %r8
	movq	$z_output_len, %r9
	call	extract_kernel
	popq	%rsi
```

我们再一次设置`rdi`为指向`boot_params`结构体的指针并把它保存到栈中。同时我们设置`rsi`指向用于内核解压的区域。最后一步是准备`extract_kernel`的参数并调用这个解压内核的函数。`extract_kernel`函数在 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/misc.c) 源文件定义并有六个参数：

* `rmode` - 指向 [boot_params](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973//arch/x86/include/uapi/asm/bootparam.h#L114) 结构体的指针，`boot_params`被引导加载器填充或在早期内核初始化时填充
* `heap` - 指向早期启动堆的起始地址 `boot_heap` 的指针
* `input_data` - 指向压缩的内核，即 `arch/x86/boot/compressed/vmlinux.bin.bz2` 的指针
* `input_len` - 压缩的内核的大小
* `output` - 解压后内核的起始地址
* `output_len` - 解压后内核的大小

所有参数根据 [System V Application Binary Interface](http://www.x86-64.org/documentation/abi.pdf) 通过寄存器传递。我们已经完成了所有的准备工作，现在我们可以看内核解压的过程。

内核解压
--------------------------------------------------------------------------------

就像我们在之前的段落中看到了那样，`extract_kernel`函数在源文件 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/misc.c) 定义并有六个参数。正如我们在之前的部分看到的，这个函数从图形/控制台初始化开始。我们要再次做这件事，因为我们不知道我们是不是从[实模式](https://en.wikipedia.org/wiki/Real_mode)开始，或者是使用了引导加载器，或者引导加载器用了32位还是64位启动协议。

在最早的初始化步骤后，我们保存空闲内存的起始和末尾地址。

```C
free_mem_ptr     = heap;
free_mem_end_ptr = heap + BOOT_HEAP_SIZE;
```

在这里 `heap` 是我们在 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S) 得到的 `extract_kernel` 函数的第二个参数：

```assembly
leaq	boot_heap(%rip), %rsi
```

如上所述，`boot_heap`定义为：

```assembly
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
```

在这里`BOOT_HEAP_SIZE`是一个展开为`0x10000`(对`bzip2`内核是`0x400000`)的宏，代表堆的大小。

在堆指针初始化后，下一步是从 [arch/x86/boot/compressed/kaslr.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/kaslr.c#L425) 调用`choose_random_location`函数。我们可以从函数名猜到，它选择内核镜像解压到的内存地址。看起来很奇怪，我们要寻找甚至是`选择`内核解压的地址，但是Linux内核支持[kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization)，为了安全，它允许解压内核到随机的地址。

在这一部分，我们不会考虑Linux内核的加载地址的随机化，我们会在下一部分讨论。

现在我们回头看 [misc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/misc.c#L404). 在获得内核镜像的地址后，需要有一些检查以确保获得的随机地址是正确对齐的，并且地址没有错误：

```C
if ((unsigned long)output & (MIN_KERNEL_ALIGN - 1))
	error("Destination physical address inappropriately aligned");

if (virt_addr & (MIN_KERNEL_ALIGN - 1))
	error("Destination virtual address inappropriately aligned");

if (heap > 0x3fffffffffffUL)
	error("Destination address too large");

if (virt_addr + max(output_len, kernel_total_size) > KERNEL_IMAGE_SIZE)
	error("Destination virtual address is beyond the kernel mapping area");

if ((unsigned long)output != LOAD_PHYSICAL_ADDR)
    error("Destination address does not match LOAD_PHYSICAL_ADDR");

if (virt_addr != LOAD_PHYSICAL_ADDR)
	error("Destination virtual address changed when not relocatable");
```

在所有这些检查后，我们可以看到熟悉的消息：

```
Decompressing Linux... 
```

然后调用解压内核的`__decompress`函数：

```C
__decompress(input_data, input_len, NULL, NULL, output, output_len, NULL, error);
```

`__decompress`函数的实现取决于在内核编译期间选择什么压缩算法：

```C
#ifdef CONFIG_KERNEL_GZIP
#include "../../../../lib/decompress_inflate.c"
#endif

#ifdef CONFIG_KERNEL_BZIP2
#include "../../../../lib/decompress_bunzip2.c"
#endif

#ifdef CONFIG_KERNEL_LZMA
#include "../../../../lib/decompress_unlzma.c"
#endif

#ifdef CONFIG_KERNEL_XZ
#include "../../../../lib/decompress_unxz.c"
#endif

#ifdef CONFIG_KERNEL_LZO
#include "../../../../lib/decompress_unlzo.c"
#endif

#ifdef CONFIG_KERNEL_LZ4
#include "../../../../lib/decompress_unlz4.c"
#endif
```

在内核解压之后，最后两个函数是`parse_elf`和`handle_relocations`.这些函数的主要用途是把解压后的内核移动到正确的位置。事实上，解压过程会[原地](https://en.wikipedia.org/wiki/In-place_algorithm)解压，我们还是要把内核移动到正确的地址。我们已经知道，内核镜像是一个[ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)可执行文件，所以`parse_elf`的主要目标是移动可加载的段到正确的地址。我们可以在`readelf`的输出看到可加载的段：

```
readelf -l vmlinux

Elf file type is EXEC (Executable file)
Entry point 0x1000000
There are 5 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
                 0x0000000000893000 0x0000000000893000  R E    200000
  LOAD           0x0000000000a93000 0xffffffff81893000 0x0000000001893000
                 0x000000000016d000 0x000000000016d000  RW     200000
  LOAD           0x0000000000c00000 0x0000000000000000 0x0000000001a00000
                 0x00000000000152d8 0x00000000000152d8  RW     200000
  LOAD           0x0000000000c16000 0xffffffff81a16000 0x0000000001a16000
                 0x0000000000138000 0x000000000029b000  RWE    200000
```

`parse_elf`函数的目标是加载这些段到从`choose_random_location`函数得到的`output`地址。这个函数从检查ELF签名标志开始：

```C
Elf64_Ehdr ehdr;
Elf64_Phdr *phdrs, *phdr;

memcpy(&ehdr, output, sizeof(ehdr));

if (ehdr.e_ident[EI_MAG0] != ELFMAG0 ||
    ehdr.e_ident[EI_MAG1] != ELFMAG1 ||
    ehdr.e_ident[EI_MAG2] != ELFMAG2 ||
    ehdr.e_ident[EI_MAG3] != ELFMAG3) {
        error("Kernel is not a valid ELF file");
        return;
}
```

如果是无效的，它会打印一条错误消息并停机。如果我们得到一个有效的`ELF`文件，我们从给定的`ELF`文件遍历所有程序头，并用正确的地址复制所有可加载的段到输出缓冲区：

```C
	for (i = 0; i < ehdr.e_phnum; i++) {
		phdr = &phdrs[i];

		switch (phdr->p_type) {
		case PT_LOAD:
#ifdef CONFIG_RELOCATABLE
			dest = output;
			dest += (phdr->p_paddr - LOAD_PHYSICAL_ADDR);
#else
			dest = (void *)(phdr->p_paddr);
#endif
			memmove(dest, output + phdr->p_offset, phdr->p_filesz);
			break;
		default:
			break;
		}
	}
```

这就是全部的工作。

从现在开始，所有可加载的段都在正确的位置。

在`parse_elf`函数之后是调用`handle_relocations`函数。这个函数的实现依赖于`CONFIG_X86_NEED_RELOCS`内核配置选项，如果它被启用，这个函数调整内核镜像的地址，只有在内核配置时启用了`CONFIG_RANDOMIZE_BASE`配置选项才会调用。`handle_relocations`函数的实现足够简单。这个函数从基准内核加载地址的值减掉`LOAD_PHYSICAL_ADDR`的值，从而我们获得内核链接后要加载的地址和实际加载地址的差值。在这之后我们可以进行内核重定位，因为我们知道内核加载的实际地址、它被链接的运行的地址和内核镜像末尾的重定位表。

在内核重定位后，我们从`extract_kernel`回来，到 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S).

内核的地址在`rax`寄存器，我们跳到那里：

```assembly
jmp	*%rax
```

就是这样。现在我们就在内核里！

结论
--------------------------------------------------------------------------------

这是关于内核引导过程的第五部分的结尾。我们不会再看到关于内核引导的文章（可能有这篇和前面的文章的更新），但是会有关于其他内核内部细节的很多文章。

下一章会描述更高级的关于内核引导过程的细节，如加载地址随机化等等。

如果你有什么问题或建议，写个评论或在 [twitter](https://twitter.com/0xAX) 找我。

**如果你发现文中描述有任何问题，请提交一个 PR 到 [linux-insides-zh](https://github.com/MintCN/linux-insides-zh) 。**

链接
--------------------------------------------------------------------------------

* [address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization)
* [initrd](https://en.wikipedia.org/wiki/Initrd)
* [long mode](https://en.wikipedia.org/wiki/Long_mode)
* [bzip2](http://www.bzip.org/)
* [RDRand instruction](https://en.wikipedia.org/wiki/RdRand)
* [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [Programmable Interval Timers](https://en.wikipedia.org/wiki/Intel_8253)
* [Previous part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-4.md)
