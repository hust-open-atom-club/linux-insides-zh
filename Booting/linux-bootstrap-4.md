内核引导过程. Part 4.
================================================================================

切换到64位模式
--------------------------------------------------------------------------------

这是 `内核引导过程` 的第四部分，我们将会看到在[保护模式](https://zh.wikipedia.org/wiki/%E4%BF%9D%E8%AD%B7%E6%A8%A1%E5%BC%8F)中的最初几步，比如确认CPU是否支持[长模式](https://zh.wikipedia.org/wiki/%E9%95%BF%E6%A8%A1%E5%BC%8F)，[SSE](https://zh.wikipedia.org/wiki/SSE)和[分页](https://zh.wikipedia.org/wiki/%E5%88%86%E9%A0%81)以及页表的初始化，在这部分的最后我们还将讨论如何切换到[长模式](https://zh.wikipedia.org/wiki/%E9%95%BF%E6%A8%A1%E5%BC%8F)。

**注意：这部分将会有大量的汇编代码，如果你不熟悉汇编，建议你找本书参考一下。**

在[前一章节](linux-bootstrap-3.md)，我们停在了跳转到位于 [arch/x86/boot/pmjump.S](http://lxr.free-electrons.com/source/arch/x86/boot/pmjump.S?v=3.18) 的 32 位入口点这一步：

```assembly
jmpl	*%eax
```

回忆一下， `eax` 寄存器包含了 32 位入口点的地址。我们可以在 [x86 linux 内核引导协议](https://www.kernel.org/doc/Documentation/x86/boot.txt) 中找到相关内容：

```
When using bzImage, the protected-mode kernel was relocated to 0x100000
```

```
当使用 bzImage 时，保护模式下的内核被重定位至 0x100000
```


让我们检查一下 32 位入口点的寄存器值来确保这是对的：

```
eax            0x100000	1048576
ecx            0x0	    0
edx            0x0	    0
ebx            0x0	    0
esp            0x1ff5c	0x1ff5c
ebp            0x0	    0x0
esi            0x14470	83056
edi            0x0	    0
eip            0x100000	0x100000
eflags         0x46	    [ PF ZF ]
cs             0x10	16
ss             0x18	24
ds             0x18	24
es             0x18	24
fs             0x18	24
gs             0x18	24
```

我们在这里可以看到 `cs` 寄存器包含了 - `0x10` （回忆前一章节，这代表了全局描述符表中的第二个索引项）， `eip` 寄存器的值是 `0x100000`，并且包括代码段在内的所有内存段的基地址都为0。所以我们可以得到物理地址： `0:0x100000` 或者 `0x100000`，这和协议规定的一样。现在让我们从 32 位入口点开始。

32 位入口点
--------------------------------------------------------------------------------

我们可以在汇编源码 [arch/x86/boot/compressed/head_64.S](http://lxr.free-electrons.com/source/arch/x86/boot/compressed/head_64.S?v=3.18) 中找到 32 位入口点的定义。

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
....
....
....
ENDPROC(startup_32)
```

首先，为什么目录名叫做 `被压缩的 (compressed)` ？实际上 `bzimage` 是由 `vmlinux + 头文件 + 内核启动代码` 被 gzip 压缩之后获得的。我们在前几个章节已经看到了启动内核的代码。所以， `head_64.S` 的主要目的就是做好进入长模式的准备之后进入长模式，进入以后再解压内核。在这一章节，我们将会看到直到内核解压缩之前的所有步骤。

在 `arch/x86/boot/compressed` 目录下有两个文件：

* [head_32.S](http://lxr.free-electrons.com/source/arch/x86/boot/compressed/head_32.S?v=3.18)
* [head_64.S](http://lxr.free-electrons.com/source/arch/x86/boot/compressed/head_64.S?v=3.18)

但是，你可能还记得我们这本书只和 `x86_64` 有关，所以我们只会关注 `head_64.S` ；在我们这里 `head_32.S` 没有被用到。让我们看一下 [arch/x86/boot/compressed/Makefile](http://lxr.free-electrons.com/source/arch/x86/boot/compressed/Makefile?v=3.18)。在那里我们可以看到以下目标：

```Makefile
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
	$(obj)/string.o $(obj)/cmdline.o \
	$(obj)/piggy.o $(obj)/cpuflags.o
```

注意 `$(obj)/head_$(BITS).o` 。这意味着我们将会选择基于 `$(BITS)` 所设置的文件执行链接操作，即 head_32.o 或者 head_64.o。`$(BITS)` 在 [arch/x86/Makefile](http://lxr.free-electrons.com/source/arch/x86/Makefile?v=3.18) 之中根据 .config 文件另外定义：

```Makefile
ifeq ($(CONFIG_X86_32),y)
        BITS := 32
        ...
		...
else
        BITS := 64
		...
		...
endif
```

现在我们知道从哪里开始了，那就来吧。

必要时重新加载内存段寄存器
--------------------------------------------------------------------------------

正如上面阐述的，我们先从 [arch/x86/boot/compressed/head_64.S](http://lxr.free-electrons.com/source/arch/x86/boot/compressed/head_64.S?v=3.18) 这个汇编文件开始。首先我们看到了在 `startup_32` 之前的特殊段属性定义：

```assembly
    __HEAD
	.code32
ENTRY(startup_32)
```

这个 `__HEAD` 是一个定义在头文件 [include/linux/init.h](http://lxr.free-electrons.com/source/include/linux/init.h?v=3.18) 中的宏，展开后就是下面这个段的定义：

```C
#define __HEAD		.section	".head.text","ax"
```

其拥有 `.head.text` 的命名和 `ax` 标记。在这里，这些标记告诉我们这个段是[可执行的](https://en.wikipedia.org/wiki/Executable)或者换种说法，包含了代码。我们可以在 [arch/x86/boot/compressed/vmlinux.lds.S](http://lxr.free-electrons.com/source/arch/x86/boot/compressed/vmlinux.lds.S?v=3.18) 这个链接脚本里找到这个段的定义： 

```
SECTIONS
{
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
	}
```

如果你不熟悉 `GNU LD` 这个链接脚本语言的语法，你可以在[这个文档](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)中找到更多信息。简单来说，这个 `.` 符号是一个链接器的特殊变量 - 位置计数器。其被赋值为相对于该段的偏移。在这里，我们将位置计数器赋值为0，这意味着我们的代码被链接到内存的 `0` 偏移处。此外，我们可以从注释里找到更多信息：

```
Be careful parts of head_64.S assume startup_32 is at address 0.
```

```
要小心， head_64.S 中一些部分假设 startup_32 位于地址 0。
```

好了，现在我们知道我们在哪里了，接下来就是深入 `startup_32` 函数的最佳时机。

在 `startup_32` 函数的开始，我们可以看到 `cld` 指令将[标志寄存器](http://baike.baidu.com/view/1845107.htm)的 `DF` （方向标志）位清空。当方向标志被清空，所有的串操作指令像[stos](http://x86.renejeschke.de/html/file_module_x86_id_306.html)， [scas](http://x86.renejeschke.de/html/file_module_x86_id_287.html)等等将会增加索引寄存器 `esi` 或者 `edi` 的值。我们需要清空方向标志是因为接下来我们会使用汇编的串操作指令来做为页表腾出空间等工作。

在我们清空 `DF` 标志后，下一步就是从内核加载头中的 `loadflags` 字段来检查 `KEEP_SEGMENTS` 标志。你是否还记得在本书的[最初一节](linux-bootstrap-1.md)，我们已经看到过 `loadflags` 。在那里我们检查了 `CAN_USE_HEAP` 标记以使用堆。现在我们需要检查 `KEEP_SEGMENTS` 标记。这些标记在 linux 的[引导协议](https://www.kernel.org/doc/Documentation/x86/boot.txt)文档中有描述：

```
Bit 6 (write): KEEP_SEGMENTS
  Protocol: 2.07+
  - If 0, reload the segment registers in the 32bit entry point.
  - If 1, do not reload the segment registers in the 32bit entry point.
    Assume that %cs %ds %ss %es are all set to flat segments with
	a base of 0 (or the equivalent for their environment).
```

```
第 6 位 (写): KEEP_SEGMENTS
  协议版本: 2.07+
  - 为0，在32位入口点重载段寄存器
  - 为1，不在32位入口点重载段寄存器。假设 %cs %ds %ss %es 都被设到基地址为0的普通段中（或者在他们的环境中等价的位置）。
```

所以，如果 `KEEP_SEGMENTS` 位在 `loadflags` 中没有被设置，我们需要重置 `ds` , `ss` 和 `es` 段寄存器到一个基地址为 `0` 的普通段中。如下：

```C
	testb $(1 << 6), BP_loadflags(%esi)
	jnz 1f

	cli
	movl	$(__BOOT_DS), %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
```

记住 `__BOOT_DS` 是 `0x18` （位于[全局描述符表](https://en.wikipedia.org/wiki/Global_Descriptor_Table)中数据段的索引）。如果设置了 `KEEP_SEGMENTS` ，我们就跳转到最近的 `1f` 标签，或者当没有 `1f` 标签，则用 `__BOOT_DS` 更新段寄存器。这非常简单，但是这是一个有趣的操作。如果你已经读了[前一章节](linux-bootstrap-3.md)，你或许还记得我们在 [arch/x86/boot/pmjump.S](http://lxr.free-electrons.com/source/arch/x86/boot/pmjump.S?v=3.18) 中切换到[保护模式](https://zh.wikipedia.org/wiki/%E4%BF%9D%E8%AD%B7%E6%A8%A1%E5%BC%8F)的时候已经更新了这些段寄存器。那么为什么我们还要去关心这些段寄存器的值呢？答案很简单，Linux 内核也有32位的引导协议，如果一个引导程序之前使用32位协议引导内核，那么在 `startup_32` 之前的代码就会被忽略。在这种情况下 `startup_32` 将会变成引导程序之后的第一个入口点，不保证段寄存器会不会处于未知状态。

在我们检查了 `KEEP_SEGMENTS` 标记并且给段寄存器设置了正确的值之后，下一步就是计算我们代码的加载和编译运行之间的位置偏差了。记住 `setup.ld.S` 包含了以下定义：在 `.head.text` 段的开始 `. = 0` 。这意味着这一段代码被编译成从 `0` 地址运行。我们可以在 `objdump` 工具的输出中看到：

```
arch/x86/boot/compressed/vmlinux:     file format elf64-x86-64


Disassembly of section .head.text:

0000000000000000 <startup_32>:
   0:   fc                      cld
   1:   f6 86 11 02 00 00 40    testb  $0x40,0x211(%rsi)
```

 `objdump` 工具告诉我们 `startup_32` 的地址是 `0` 。但实际上并不是。我们当前的目标是获知我们实际上在哪里。在[长模式](https://zh.wikipedia.org/wiki/%E9%95%BF%E6%A8%A1%E5%BC%8F)下，这非常简单，因为其支持 `rip` 相对寻址，但是我们当前处于[保护模式](https://zh.wikipedia.org/wiki/%E4%BF%9D%E8%AD%B7%E6%A8%A1%E5%BC%8F)下。我们将会使用一个常用的方法来确定 `startup_32` 的地址。我们需要定义一个标签并且跳转到它，然后把栈顶抛出到一个寄存器中：

```assembly
call label
label: pop %reg
```

在这之后，那个寄存器将会包含标签的地址，让我们看看在 Linux 内核中类似的寻找 `startup_32` 地址的代码：

```assembly
	leal	(BP_scratch+4)(%esi), %esp
	call	1f
1:  popl	%ebp
	subl	$1b, %ebp
```

回忆前一节， `esi` 寄存器包含了 [boot_params](http://lxr.free-electrons.com/source/arch/x86/include/uapi/asm/bootparam.h?v=3.18#L113) 结构的地址，这个结构在我们切换到保护模式之前已经被填充了。`bootparams` 这个结构体包含了一个特殊的字段 `scratch` ，其偏移量为 `0x1e4` 。这个 4 字节的区域将会成为 `call` 指令的临时栈。我们把 `scratch` 的地址加 4 存入 `esp` 寄存器。我们之所以在 `BP_scratch` 基础上加 `4` 是因为，如之前所说的，这将成为一个临时的栈，而在 `x86_64` 架构下，栈是自顶向下生长的。所以我们的栈指针就会指向栈顶。接下来我们就可以看到我上面描述的过程。我们跳转到 `1f` 标签并且把该标签的地址放入 `ebp` 寄存器，因为在执行 `call` 指令之后我们把返回地址放到了栈顶。那么，目前我们拥有 `1f` 标签的地址，也能够很容易得到 `startup_32` 的地址。我们只需要把我们从栈里得到的地址减去标签的地址：

```
startup_32 (0x0)     +-----------------------+
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
1f (0x0 + 1f offset) +-----------------------+ %ebp - 实际物理地址
                     |                       |
                     |                       |
                     +-----------------------+
```

 `startup_32` 被链接为在 `0x0` 地址运行，这意味着 `1f` 的地址为 `0x0 + 1f 的偏移量` 。实际上偏移量大概是 `0x22` 字节。 `ebp` 寄存器包含了 `1f` 标签的实际物理地址。所以如果我们从 `ebp` 中减去 `1f` ，我们就会得到 `startup_32` 的实际物理地址。Linux 内核的[引导协议](https://www.kernel.org/doc/Documentation/x86/boot.txt)描述了保护模式下的内核基地址是 `0x100000` 。我们可以用 [gdb](https://zh.wikipedia.org/wiki/GNU%E4%BE%A6%E9%94%99%E5%99%A8) 来验证。让我们启动调试器并且在 `1f` 的地址 `0x100022` 添加断点。如果这是正确的，我们将会看到在 `ebp` 寄存器中值为 `0x100022` ：

```
$ gdb
(gdb)$ target remote :1234
Remote debugging using :1234
0x0000fff0 in ?? ()
(gdb)$ br *0x100022
Breakpoint 1 at 0x100022
(gdb)$ c
Continuing.

Breakpoint 1, 0x00100022 in ?? ()
(gdb)$ i r
eax            0x18	0x18
ecx            0x0	0x0
edx            0x0	0x0
ebx            0x0	0x0
esp            0x144a8	0x144a8
ebp            0x100021	0x100021
esi            0x142c0	0x142c0
edi            0x0	0x0
eip            0x100022	0x100022
eflags         0x46	[ PF ZF ]
cs             0x10	0x10
ss             0x18	0x18
ds             0x18	0x18
es             0x18	0x18
fs             0x18	0x18
gs             0x18	0x18
```

如果我们执行下一条指令 `subl	$1b, %ebp` ，我们将会看到：

```
nexti
...
ebp            0x100000	0x100000
...
```

好了，那是对的。`startup_32` 的地址是 `0x100000` 。在我们知道了 `startup_32` 的地址之后，我们可以开始准备切换到[长模式](https://zh.wikipedia.org/wiki/%E9%95%BF%E6%A8%A1%E5%BC%8F)了。我们的下一个目标是建立栈并且确认 CPU 对长模式和 [SSE](https://zh.wikipedia.org/wiki/SSE) 的支持。

栈的建立和 CPU 的确认
--------------------------------------------------------------------------------

如果不知道 `startup_32` 标签的地址，我们就无法建立栈。我们可以把栈看作是一个数组，并且栈指针寄存器 `esp` 必须指向数组的底部。当然我们可以在自己的代码里定义一个数组，但是我们需要知道其真实地址来正确配置栈指针。让我们看一下代码：

```assembly
	movl	$boot_stack_end, %eax
	addl	%ebp, %eax
	movl	%eax, %esp
```

 `boots_stack_end` 标签被定义在同一个汇编文件 [arch/x86/boot/compressed/head_64.S](http://lxr.free-electrons.com/source/arch/x86/boot/compressed/head_64.S?v=3.18) 中，位于 [.bss](https://en.wikipedia.org/wiki/.bss) 段：

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

首先，我们把 `boot_stack_end` 放到 `eax` 寄存器中。那么 `eax` 寄存器将包含 `boot_stack_end` 链接后的地址或者说 `0x0 + boot_stack_end` 。为了得到 `boot_stack_end` 的实际地址，我们需要加上 `startup_32` 的实际地址。回忆一下，前面我们找到了这个地址并且把它存到了 `ebp` 寄存器中。最后，`eax` 寄存器将会包含 `boot_stack_end` 的实际地址，我们只需要将其加到栈指针上。

在外面建立了栈之后，下一步是 CPU 的确认。既然我们将要切换到 `长模式` ，我们需要检查 CPU 是否支持 `长模式` 和 `SSE`。我们将会在跳转到 `verify_cpu` 函数之后执行：

```assembly
	call	verify_cpu
	testl	%eax, %eax
	jnz	no_longmode
```

这个函数定义在 [arch/x86/kernel/verify_cpu.S](http://lxr.free-electrons.com/source/arch/x86/kernel/verify_cpu.S?v=3.18) 中，只是包含了几个对 [cpuid](https://en.wikipedia.org/wiki/CPUID) 指令的调用。该指令用于获取处理器的信息。在我们的情况下，它检查了对 `长模式` 和 `SSE` 的支持，通过 `eax` 寄存器返回0表示成功，1表示失败。

如果 `eax` 的值不是 0 ，我们就跳转到 `no_longmode` 标签，用 `hlt` 指令停止 CPU ，期间不会发生硬件中断：

```assembly
no_longmode:
1:
	hlt
	jmp     1b
```

如果 `eax` 的值为0，万事大吉，我们可以继续。

计算重定位地址
--------------------------------------------------------------------------------

下一步是在必要的时候计算解压缩之后的地址。首先，我们需要知道内核重定位的意义。我们已经知道 Linux 内核的32位入口点地址位于 `0x100000` 。但是那是一个32位的入口。默认的内核基地址由内核配置项 `CONFIG_PHYSICAL_START` 的值所确定，其默认值为 `0x1000000` 或 `16 MB` 。这里的主要问题是如果内核崩溃了，内核开发者需要一个配置于不同地址加载的 `救援内核` 来进行 [kdump](https://www.kernel.org/doc/Documentation/kdump/kdump.txt)。Linux 内核提供了特殊的配置选项以解决此问题 - `CONFIG_RELOCATABLE` 。我们可以在内核文档中找到：

```
This builds a kernel image that retains relocation information
so it can be loaded someplace besides the default 1MB.

Note: If CONFIG_RELOCATABLE=y, then the kernel runs from the address
it has been loaded at and the compile time physical address
(CONFIG_PHYSICAL_START) is used as the minimum location.
```

```
这建立了一个保留了重定向信息的内核镜像，这样就可以在默认的 1MB 位置之外加载了。

注意：如果 CONFIG_RELOCATABLE=y， 那么 内核将会从其被加载的位置运行，编译时的物理地址 (CONFIG_PHYSICAL_START) 将会被作为最低地址位置的限制。
```

简单来说，这意味着相同配置下的 Linux 内核可以从不同地址被启动。这是通过将程序以 [位置无关代码](https://zh.wikipedia.org/wiki/%E5%9C%B0%E5%9D%80%E6%97%A0%E5%85%B3%E4%BB%A3%E7%A0%81) 的形式编译来达到的。如果我们参考 [/arch/x86/boot/compressed/Makefile](http://lxr.free-electrons.com/source/arch/x86/boot/compressed/Makefile?v=3.18)，我们将会看到解压器的确是用 `-fPIC` 标记编译的：

```Makefile
KBUILD_CFLAGS += -fno-strict-aliasing -fPIC
```

当我们使用位置无关代码时，一段代码的地址是由一个控制地址加上程序计数器计算得到的。我们可以从任意一个地址加载使用这种方式寻址的代码。这就是为什么我们需要获得 `startup_32` 的实际地址。现在让我们回到 Linux 内核代码。我们目前的目标是计算出内核解压的地址。这个地址的计算取决于内核配置项 `CONFIG_RELOCATABLE` 。让我们看代码：

```assembly
#ifdef CONFIG_RELOCATABLE
	movl	%ebp, %ebx
	movl	BP_kernel_alignment(%esi), %eax
	decl	%eax
	addl	%eax, %ebx
	notl	%eax
	andl	%eax, %ebx
	cmpl	$LOAD_PHYSICAL_ADDR, %ebx
	jge	1f
#endif
	movl	$LOAD_PHYSICAL_ADDR, %ebx
1:
	addl	$z_extract_offset, %ebx
```

记住 `ebp` 寄存器的值就是 `startup_32` 标签的物理地址。如果在内核配置中 `CONFIG_RELOCATABLE` 内核配置项开启，我们就把这个地址放到 `ebx` 寄存器中，对齐到 `2M` 的整数倍 ，然后和 `LOAD_PHYSICAL_ADDR` 的值比较。 `LOAD_PHYSICAL_ADDR` 宏定义在头文件 [arch/x86/include/asm/boot.h](http://lxr.free-electrons.com/source/arch/x86/include/asm/boot.h?v=3.18) 中，如下：

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

我们可以看到该宏只是展开成对齐的 `CONFIG_PHYSICAL_ALIGN` 值，其表示了内核加载位置的物理地址。在比较了 `LOAD_PHYSICAL_ADDR` 和 `ebx` 的值之后，我们给 `startup_32` 加上偏移来获得解压内核镜像的地址。如果 `CONFIG_RELOCATABLE` 选项在内核配置时没有开启，我们就直接将默认的地址加上 `z_extract_offset` 。

在前面的操作之后，`ebp` 包含了我们加载时的地址，`ebx` 被设为内核解压缩的目标地址。

进入长模式前的准备工作
--------------------------------------------------------------------------------

在我们得到了重定位内核镜像的基地址之后，我们需要做切换到64位模式之前的最后准备。首先，我们需要更新[全局描述符表](https://en.wikipedia.org/wiki/Global_Descriptor_Table)：

```assembly
	leal	gdt(%ebp), %eax
	movl	%eax, gdt+2(%ebp)
	lgdt	gdt(%ebp)
```

在这里我们把 `ebp` 寄存器加上 `gdt` 的偏移存到 `eax` 寄存器。接下来我们把这个地址放到 `ebp` 加上 `gdt+2` 偏移的位置上，并且用 `lgdt` 指令载入 `全局描述符表` 。为了理解这个神奇的 `gdt` 偏移量，我们需要关注 `全局描述符表` 的定义。我们可以在同一个[源文件](http://lxr.free-electrons.com/source/arch/x86/boot/compressed/head_64.S?v=3.18)中找到其定义：

```assembly
	.data
gdt:
	.word	gdt_end - gdt
	.long	gdt
	.word	0
	.quad	0x0000000000000000	/* NULL descriptor */
	.quad	0x00af9a000000ffff	/* __KERNEL_CS */
	.quad	0x00cf92000000ffff	/* __KERNEL_DS */
	.quad	0x0080890000000000	/* TS descriptor */
	.quad   0x0000000000000000	/* TS continued */
gdt_end:
```

我们可以看到其位于 `.data` 段，并且包含了5个描述符： `null` 、内核代码段、内核数据段和其他两个任务描述符。我们已经在[上一章节](linux-bootstrap-3.md)载入了 `全局描述符表` ，和我们现在做的差不多，但是将描述符改为 `CS.L = 1` `CS.D = 0` 从而在 `64` 位模式下执行。我们可以看到， `gdt` 的定义从两个字节开始： `gdt_end - gdt` ，代表了 `gdt` 表的最后一个字节，或者说表的范围。接下来的4个字节包含了 `gdt` 的基地址。记住 `全局描述符表` 保存在 `48位 GDTR-全局描述符表寄存器` 中，由两个部分组成：

* 全局描述符表的大小 (16位）
* 全局描述符表的基址 (32位)

所以，我们把 `gdt` 的地址放到 `eax` 寄存器，然后存到 `.long	gdt` 或者 `gdt+2`。现在我们已经建立了 `GDTR` 寄存器的结构，并且可以用 `lgdt` 指令载入 `全局描述符表` 了。

在我们载入 `全局描述符表` 之后，我们必须启用 [PAE](http://en.wikipedia.org/wiki/Physical_Address_Extension) 模式。方法是将 `cr4` 寄存器的值传入 `eax` ，将第5位置1，然后再写回 `cr4` 。

```assembly
	movl	%cr4, %eax
	orl	$X86_CR4_PAE, %eax
	movl	%eax, %cr4
```

现在我们已经接近完成进入64位模式前的所有准备工作了。最后一步是建立页表，但是在此之前，这里有一些关于长模式的知识。

长模式
--------------------------------------------------------------------------------

[长模式](https://zh.wikipedia.org/wiki/%E9%95%BF%E6%A8%A1%E5%BC%8F)是 [x86_64](https://en.wikipedia.org/wiki/X86-64) 系列处理器的原生模式。首先让我们看一看 `x86_64` 和 `x86` 的一些区别。

 `64位` 模式提供了一些新特性，比如：

* 从 `r8` 到 `r15` 8个新的通用寄存器，并且所有通用寄存器都是64位的了。
* 64位指令指针 - `RIP` ;
* 新的操作模式 - 长模式;
* 64位地址和操作数;
* RIP 相对寻址 (我们将会在接下来的章节看到一个例子).

长模式是一个传统保护模式的扩展，其由两个子模式构成：

* 64位模式
* 兼容模式

为了切换到 `64位` 模式，我们需要完成以下操作：

* 启用 [PAE](https://en.wikipedia.org/wiki/Physical_Address_Extension);
* 建立页表并且将顶级页表的地址放入 `cr3` 寄存器;
* 启用 `EFER.LME` ;
* 启用分页;


我们已经通过设置 `cr4` 控制寄存器中的 `PAE` 位启动 `PAE` 了。在下一个段落，我们就要建立[页表](https://zh.wikipedia.org/wiki/%E5%88%86%E9%A0%81)的结构了。

初期页表初始化
--------------------------------------------------------------------------------

现在，我们已经知道了在进入 `64位` 模式之前，我们需要先建立页表，那么就让我们看看如何建立初期的 `4G` 启动页表。

**注意：我不会在这里解释虚拟内存的理论，如果你想知道更多，查看本节最后的链接**

Linux 内核使用 `4级` 页表，通常我们会建立6个页表：

* 1 个 `PML4` 或称为 `4级页映射` 表，包含 1 个项；
* 1 个 `PDP` 或称为 `页目录指针` 表，包含 4 个项；
* 4 个 页目录表，一共包含 `2048` 个项；

让我们看看其实现方式。首先我们在内存中为页表清理一块缓存。每个表都是 `4096` 字节，所以我们需要 `24` KB 的空间：

```assembly
	leal	pgtable(%ebx), %edi
	xorl	%eax, %eax
	movl	$((4096*6)/4), %ecx
	rep	stosl
```

我们把和 `ebx` 相关的 `pgtable` 的地址放到 `edi` 寄存器中，清空 `eax` 寄存器，并将 `ecx` 赋值为 `6144` 。 `rep stosl` 指令将会把 `eax` 的值写到 `edi` 指向的地址，然后给 `edi` 加 4 ， `ecx` 减 4 ，重复直到 `ecx` 小于等于 0 。所以我们才把 `6144` 赋值给 `ecx` 。

 `pgtable` 定义在 [arch/x86/boot/compressed/head_64.S](http://lxr.free-electrons.com/source/arch/x86/boot/compressed/head_64.S?v=3.18) 的最后：

```assembly
	.section ".pgtable","a",@nobits
	.balign 4096
pgtable:
	.fill 6*4096, 1, 0
```

我们可以看到，其位于 `.pgtable` 段，大小为 `24KB` 。

在我们为 `pgtable` 分配了空间之后，我们可以开始构建顶级页表 - `PML4` ：

```assembly
	leal	pgtable + 0(%ebx), %edi
	leal	0x1007 (%edi), %eax
	movl	%eax, 0(%edi)
```

还是在这里，我们把和 `ebx` 相关的，或者说和 `startup_32` 相关的 `pgtable` 的地址放到 `ebi` 寄存器。接下来我们把相对此地址偏移 `0x1007` 的地址放到 `eax` 寄存器中。 `0x1007` 是 `PML4` 的大小 `4096` 加上 `7` 。这里的 `7` 代表了 `PML4` 的项标记。在我们这里，这些标记是 `PRESENT+RW+USER` 。在最后我们把第一个 `PDP（页目录指针）` 项的地址写到 `PML4` 中。

在接下来的一步，我们将会在 `页目录指针（PDP）` 表（3级页表）建立 4 个带有 `PRESENT+RW+USE` 标记的 `Page Directory （2级页表）` 项：

```assembly
	leal	pgtable + 0x1000(%ebx), %edi
	leal	0x1007(%edi), %eax
	movl	$4, %ecx
1:  movl	%eax, 0x00(%edi)
	addl	$0x00001000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

我们把 3 级页目录指针表的基地址（从 `pgtable` 表偏移 `4096` 或者 `0x1000` ）放到 `edi` ，把第一个 2 级页目录指针表的首项的地址放到 `eax` 寄存器。把 `4` 赋值给 `ecx` 寄存器，其将会作为接下来循环的计数器，然后将第一个页目录指针项写到 `edi` 指向的地址。之后， `edi` 将会包含带有标记 `0x7` 的第一个页目录指针项的地址。接下来我们就计算后面的几个页目录指针项的地址，每个占 8 字节，把地址赋值给 `eax` ，然后回到循环开头将其写入 `edi` 所在地址。建立页表结构的最后一步就是建立 `2048` 个 `2MB` 页的页表项。

```assembly
	leal	pgtable + 0x2000(%ebx), %edi
	movl	$0x00000183, %eax
	movl	$2048, %ecx
1:  movl	%eax, 0(%edi)
	addl	$0x00200000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

在这里我们做的几乎和上面一样，所有的表项都带着标记 - `$0x00000183` - `PRESENT + WRITE + MBZ` 。最后我们将会拥有 `2048` 个 `2MB` 大的页，或者说：

```python
>>> 2048 * 0x00200000
4294967296
```

一个 `4G` 页表。我们刚刚完成我们的初期页表结构，其映射了 `4G` 大小的内存，现在我们可以把高级页表 `PML4` 的地址放到 `cr3` 寄存器中了：

```assembly
	leal	pgtable(%ebx), %eax
	movl	%eax, %cr3
```

这样就全部结束了。所有的准备工作都已经完成，我们可以开始看如何切换到长模式了。

切换到长模式
--------------------------------------------------------------------------------

首先我们需要设置 [MSR](http://en.wikipedia.org/wiki/Model-specific_register) 中的 `EFER.LME` 标记为 `0xC0000080` ：

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_LME, %eax
	wrmsr
```

在这里我们把 `MSR_EFER` 标记（在 [arch/x86/include/uapi/asm/msr-index.h](http://lxr.free-electrons.com/source/arch/x86/include/uapi/asm/msr-index.h?v=3.18#L7) 中定义）放到 `ecx` 寄存器中，然后调用 `rdmsr` 指令读取 [MSR](http://en.wikipedia.org/wiki/Model-specific_register) 寄存器。在 `rdmsr` 执行之后，我们将会获得 `edx:eax` 中的结果值，其取决于 `ecx` 的值。我们通过 `btsl` 指令检查 `EFER_LME` 位，并且通过 `wrmsr` 指令将 `eax` 的数据写入 `MSR` 寄存器。

下一步我们将内核段代码地址入栈（我们在 GDT 中定义了），然后将 `startup_64` 的地址导入 `eax` 。

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
```

在这之后我们把这个地址入栈然后通过设置 `cr0` 寄存器中的 `PG` 和 `PE` 启用分页：

```assembly
	movl	$(X86_CR0_PG | X86_CR0_PE), %eax
	movl	%eax, %cr0
```

然后执行：

```assembly
lret
```

指令。记住前一步我们已经将 `startup_64` 函数的地址入栈，在 `lret` 指令之后，CPU 取出了其地址跳转到那里。

这些步骤之后我们最后来到了64位模式：

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
....
....
....
```


就是这样！

总结
--------------------------------------------------------------------------------

这是 linux 内核启动流程的第4部分。如果你有任何的问题或者建议，你可以留言，也可以直接发消息给我 [twitter](https://twitter.com/0xAX) 或者创建一个 [issue](https://github.com/0xAX/linux-insides/issues/new)。

下一节我们将会看到内核解压缩流程和其他更多。

**如果你发现文中描述有任何问题，请提交一个 PR 到 [linux-insides-zh](https://github.com/MintCN/linux-insides-zh) 。**

相关链接
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Intel® 64 and IA-32 Architectures Software Developer’s Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [GNU linker](http://www.eecs.umich.edu/courses/eecs373/readings/Linker.pdf)
* [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)
* [Paging](http://en.wikipedia.org/wiki/Paging)
* [Model specific register](http://en.wikipedia.org/wiki/Model-specific_register)
* [.fill instruction](http://www.chemie.fu-berlin.de/chemnet/use/info/gas/gas_7.html)
* [Previous part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md)
* [Paging on osdev.org](http://wiki.osdev.org/Paging)
* [Paging Systems](https://www.cs.rutgers.edu/~pxk/416/notes/09a-paging.html)
* [x86 Paging Tutorial](http://www.cirosantilli.com/x86-paging/)
