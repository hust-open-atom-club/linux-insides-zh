内核启动过程，第三部分
================================================================================

显示模式初始化和进入保护模式
--------------------------------------------------------------------------------

这一章是`内核启动过程`的第三部分，在[前一章](linux-bootstrap-2.md#kernel-booting-process-part-2)中，我们的内核启动过程之旅停在了对 `set_video` 函数的调用（这个函数定义在 [main.c](http://lxr.free-electrons.com/source/arch/x86/boot/main.c?v=3.18#L181)）。在这一章中，我们将接着上一章继续我们的内核启动之旅。在这一章你将读到下面的内容：
- 显示模式的初始化，
- 在进入保护模式之前的准备工作，
- 正式进入保护模式

**注意** 如果你对保护模式一无所知，你可以查看[前一章](linux-bootstrap-2.md#protected-mode) 的相关内容。另外，你也可以查看下面这些[链接](linux-bootstrap-2.md#links) 以了解更多关于保护模式的内容。

就像我们前面所说的，我们将从 `set_video` 函数开始我们这章的内容，你可以在 [arch/x86/boot/video.c](http://lxr.free-electrons.com/source/arch/x86/boot/video.c?v=3.18#L315) 找到这个函数的定义。 这个函数首先从 `boot_params.hdr` 数据结构获取显示模式设置：

```C
u16 mode = boot_params.hdr.vid_mode;
```

至于 `boot_params.hdr` 数据结构中的内容，是通过 `copy_boot_params` 函数实现的 （关于这个函数的实现细节请查看上一章的内容），`boot_params.hdr` 中的 `vid_mode` 是引导程序必须填入的字段。你可以在 `kernel boot protocol` 文档中找到关于 `vid_mode` 的详细信息：

```
Offset	Proto	Name		Meaning
/Size
01FA/2	ALL	    vid_mode	Video mode control
```

而在 `linux kernel boot protocol` 文档中定义了如何通过命令行参数的方式为 `vid_mode` 字段传入相应的值：

```
**** SPECIAL COMMAND LINE OPTIONS
vga=<mode>
	<mode> here is either an integer (in C notation, either
	decimal, octal, or hexadecimal) or one of the strings
	"normal" (meaning 0xFFFF), "ext" (meaning 0xFFFE) or "ask"
	(meaning 0xFFFD).  This value should be entered into the
	vid_mode field, as it is used by the kernel before the command
	line is parsed.
```

根据上面的描述，我们可以通过将 `vga` 选项写入 grub 或者写到引导程序的配置文件，从而让内核命令行得到相应的显示模式设置信息。这个选项可以接受不同类型的值来表示相同的意思。比如你可以传入 0XFFFD 或者 ask，这2个值都表示需要显示一个菜单让用户选择想要的显示模式。下面的链接就给出了这个菜单：

![video mode setup menu](http://oi59.tinypic.com/ejcz81.jpg)

通过这个菜单，用户可以选择想要进入的显示模式。不过在我们进一步了解显示模式的设置过程之前，让我们先回头了解一些重要的概念。

内核数据类型
--------------------------------------------------------------------------------

在前面的章节中，我们已经接触到了一个类似于 `u16` 的内核数据类型。下面列出了更多内核支持的数据类型：


| Type | char | short | int | long | u8 | u16 | u32 | u64 |
|------|------|-------|-----|------|----|-----|-----|-----|
| Size |  1   |   2   |  4  |   8  |  1 |  2  |  4  |  8  |

如果你尝试阅读内核代码，最好能够牢记这些数据类型。

堆操作 API
--------------------------------------------------------------------------------

在 `set_video` 函数将 `vid_mod` 的值设置完成之后，将调用 `RESET_HEAP` 宏将 HEAP 头指向 `_end` 符号。`RESET_HEAP` 宏定义在  [boot.h](http://lxr.free-electrons.com/source/arch/x86/boot/boot.h?v=3.18#L199)：

```C
#define RESET_HEAP() ((void *)( HEAP = _end ))
```

如果你阅读过第二部分，你应该还记得在第二部分中，我们通过 [`init_heap`](http://lxr.free-electrons.com/source/arch/x86/boot/main.c?v=3.18#L116) 函数完成了 HEAP 的初始化。在 `boot.h` 中定义了一系列的方法来操作被初始化之后的 HEAP。这些操作包括：

```C
#define RESET_HEAP() ((void *)( HEAP = _end ))
```

就像我们在前面看到的，这个宏只是简单的将 HEAP 头设置到 `_end` 标号。在上一章中我们已经说明了 `_end` 标号，在 `boot.h` 中通过 `extern char _end[];` 来引用（从这里可以看出，在内核初始化的时候堆和栈是共享内存空间的，详细的信息可以查看第一章的堆栈初始化和第二章的堆初始化）：

下面一个是 `GET_HEAP` 宏：

```C
#define GET_HEAP(type, n) \
	((type *)__get_heap(sizeof(type),__alignof__(type),(n)))
```

可以看出这个宏调用了 `__get_heap` 函数来进行内存的分配。`__get_heap` 需要下面3个参数来进行内存分配操作：

* 某个数据类型所占用的字节数
* `__alignof__(type)` 返回对于请求的数据类型需要怎样的对齐方式 ( 根据我的了解这个是 gcc 提供的一个功能 ）
* `n` 需要分配多少个对应数据类型的对象

下面是 `__get_heap` 函数的实现：

```C
static inline char *__get_heap(size_t s, size_t a, size_t n)
{
	char *tmp;

	HEAP = (char *)(((size_t)HEAP+(a-1)) & ~(a-1));
	tmp = HEAP;
	HEAP += s*n;
	return tmp;
}
```

现在让我们来了解这个函数是如何工作的。 这个函数首先根据对齐方式要求（参数 `a` ）调整 `HEAP` 的值，然后将 `HEAP` 值赋值给一个临时变量 `tmp`。接下来根据需要分配的对象的个数（参数 `n` ），预留出所需要的内存，然后将 `tmp` 返回给调用端。

最后一个关于 HEAP 的操作是：

```C
static inline bool heap_free(size_t n)
{
	return (int)(heap_end - HEAP) >= (int)n;
}
```

这个函数简单做了一个减法 `heap_end - HEAP`，如果相减的结果大于请求的内存，那么就返回真，否则返回假。

我们已经看到了所有可以对 HEAP 进行操作，下面让我们继续显示模式设置过程。

设置显示模式
--------------------------------------------------------------------------------

在我们分析了内核数据类型以及和 HEAP 相关的操作之后，让我们回来继续分析显示模式的初始化。在 `RESET_HEAP()` 函数被调用之后，`set_video` 函数接着调用 `store_mode_params` 函数将对应显示模式的相关参数写入 `boot_params.screen_info` 字段。这个字段的结构定义可以在 [include/uapi/linux/screen_info.h](https://github.com/0xAX/linux/blob/master/include/uapi/linux/screen_info.h) 中找到。

`store_mode_params` 函数将调用 `store_cursor_position` 函数将当前屏幕上光标的位置保存起来。下面让我们来看 `store_cursor_poistion` 函数是如何实现的。

首先函数初始化一个类型为 `biosregs` 的变量，将其中的 `AH` 寄存器内容设置成 `0x3`，然后调用 `0x10` BIOS 中断。当中断调用返回之后，`DL` 和 `DH` 寄存器分别包含了当前光标的行和列信息。接着，这2个信息将被保存到 `boot_params.screen_info` 字段的 `orig_x` 和 `orig_y`字段。

在 `store_cursor_position` 函数执行完毕之后，`store_mode_params` 函数将调用 `store_video_mode` 函数将当前使用的显示模式保存到 `boot_params.screen_info.orig_video_mode`。

接下來 `store_mode_params` 函数将根据当前显示模式的设定，给 `video_segment` 变量设置正确的值（实际上就是设置显示内存的起始地址）。在 BIOS 将控制权转移到引导扇区的时候，显示内存地址和显示模式的对应关系如下表所示：

```
0xB000:0x0000 	32 Kb 	Monochrome Text Video Memory
0xB800:0x0000 	32 Kb 	Color Text Video Memory
```

根据上表，如果当前显示模式是 MDA, HGC 或者单色 VGA 模式，那么 `video_sgement` 的值将被设置成 `0xB000`；如果当前显示模式是彩色模式，那么 `video_segment` 的值将被设置成 `0xB800`。在这之后，`store_mode_params` 函数将保存字体大小信息到 `boot_params.screen_info.orig_video_points`：

```C
//保存字体大小信息
set_fs(0);
font_size = rdfs16(0x485);
boot_params.screen_info.orig_video_points = font_size;
```

这段代码首先调用 `set_fs` 函数（在 [boot.h](https://github.com/0xAX/linux/blob/master/arch/x86/boot/boot.h) 中定义了许多类似的函数进行寄存器操作）将数字 `0` 放入 `FS` 寄存器。接着从内存地址 `0x485` 处获取字体大小信息并保存到 `boot_params.screen_info.orig_video_points`。

```
 x = rdfs16(0x44a);
 y = (adapter == ADAPTER_CGA) ? 25 : rdfs8(0x484)+1;
```

接下来代码将从地址 `0x44a` 处获得屏幕列信息，从地址 `0x484` 处获得屏幕行信息，并将它们保存到 `boot_params.screen_info.orig_video_cols` 和 `boot_params.screen_info.orig_video_lines`。到这里，`store_mode_params` 的执行就结束了。

接下来，`set_video` 函数将调用 `save_screen` 函数将当前屏幕上的所有信息保存到 HEAP 中。这个函数首先获得当前屏幕的所有信息（包括屏幕大小，当前光标位置，屏幕上的字符信息），并且保存到 `saved_screen` 结构体中。这个结构体的定义如下所示：

```C
static struct saved_screen {
	int x, y;
	int curx, cury;
	u16 *data;
} saved;
```

接下来函数将检查 HEAP 中是否有足够的空间保存这个结构体的数据：

```C
if (!heap_free(saved.x*saved.y*sizeof(u16)+512))
		return;
```

如果 HEAP 有足够的空间，代码将在 HEAP 中分配相应的空间并且将 `saved_screen` 保存到 HEAP。

接下来 `set_video` 函数将调用 `probe_cards(0)`（这个函数定义在  [arch/x86/boot/video-mode.c](https://github.com/0xAX/linux/blob/master/arch/x86/boot/video-mode.c#L33)）。 这个函数简单遍历所有的显卡，并通过调用驱动程序设置显卡所支持的显示模式：

```C
for (card = video_cards; card < video_cards_end; card++) {
		if (card->unsafe == unsafe) {
			if (card->probe)
				card->nmodes = card->probe();
			else
				card->nmodes = 0;
		}
}
```

如果你仔细看上面的代码，你会发现 `video_cards` 这个变量并没有被声明，那么程序怎么能够正常编译执行呢？实际上很简单，它指向了一个在 [arch/x86/boot/setup.ld](https://github.com/0xAX/linux/blob/master/arch/x86/boot/setup.ld) 中定义的叫做 `.videocards` 的内存段：
```
	.videocards	: {
		video_cards = .;
		*(.videocards)
		video_cards_end = .;
	}
```
那么这段内存里面存放的数据是什么呢，下面我们就来详细分析。在内核初始化代码中，对于每个支持的显示模式都是使用下面的代码进行定义的：

```C
static __videocard video_vga = {
	.card_name	= "VGA",
	.probe		= vga_probe,
	.set_mode	= vga_set_mode,
};
```

`__videocard` 是一个宏定义，如下所示：

```C
#define __videocard struct card_info __attribute__((used,section(".videocards")))
```

因此 `__videocard` 是一个 `card_info` 结构，这个结构定义如下：

```C
struct card_info {
	const char *card_name;
	int (*set_mode)(struct mode_info *mode);
	int (*probe)(void);
	struct mode_info *modes;
	int nmodes;
	int unsafe;
	u16 xmode_first;
	u16 xmode_n;
};
```

在 `.videocards` 内存段实际上存放的就是所有被内核初始化代码定义的 `card_info` 结构（可以看成是一个数组），所以 `probe_cards` 函数可以使用 `video_cards`，通过循环遍历所有的 `card_info`。

在 `probe_cards` 执行完成之后，我们终于进入 `set_video` 函数的主循环了。在这个循环中，如果 `vid_mode=ask`，那么将显示一个菜单让用户选择想要的显示模式，然后代码将根据用户的选择或者 `vid_mod` 的值 ，通过调用 `set_mode` 函数来设置正确的显示模式。如果设置成功，循环结束，否则显示菜单让用户选择显示模式，继续进行设置显示模式的尝试。

```c
for (;;) {
      if (mode == ASK_VGA)
          mode = mode_menu();

      if (!set_mode(mode))
          break;

      printf("Undefined video mode number: %x\n", mode);
      mode = ASK_VGA;
  }
```

你可以在 [video-mode.c](https://github.com/0xAX/linux/blob/master/arch/x86/boot/video-mode.c#L147) 中找到 `set_mode` 函数的定义。这个函数只接受一个参数，这个参数是对应的显示模式的数字表示（这个数字来自于显示模式选择菜单，或者从内核命令行参数获得）。

`set_mode` 函数首先检查传入的 `mode` 参数，然后调用 `raw_set_mode` 函数。而后者将遍历内核知道的所有 `card_info` 信息，如果发现某张显卡支持传入的模式，这调用 `card_info` 结构中保存的 `set_mode` 函数地址进行显卡显示模式的设置。以 `video_vga` 这个 `card_info` 结构来说，保存在其中的 `set_mode` 函数就指向了 `vga_set_mode` 函数。下面的代码就是 `vga_set_mode` 函数的实现，这个函数根据输入的 vga 显示模式，调用不同的函数完成显示模式的设置：

```C
static int vga_set_mode(struct mode_info *mode)
{
	vga_set_basic_mode();

	force_x = mode->x;
	force_y = mode->y;

	switch (mode->mode) {
	case VIDEO_80x25:
		break;
	case VIDEO_8POINT:
		vga_set_8font();
		break;
	case VIDEO_80x43:
		vga_set_80x43();
		break;
	case VIDEO_80x28:
		vga_set_14font();
		break;
	case VIDEO_80x30:
		vga_set_80x30();
		break;
	case VIDEO_80x34:
		vga_set_80x34();
		break;
	case VIDEO_80x60:
		vga_set_80x60();
		break;
	}
	return 0;
}
```

在上面的代码中，每个 `vga_set***` 函数只是简单调用 `0x10` BIOS 中断来进行显示模式的设置。

在显卡的显示模式被正确设置之后，这个最终的显示模式被写回  `boot_params.hdr.vid_mode`。

接下来 `set_video` 函数将调用 `vesa_store_edid` 函数， 这个函数只是简单的将  [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data) (**E**xtended **D**isplay **I**dentification **D**ata) 写入内存，以便于内核访问。最后， `set_video` 将调用 `do_restore` 函数将前面保存的当前屏幕信息还原到屏幕上。

到这里为止，显示模式的设置完成，接下来我们可以切换到保护模式了。

在切换到保护模式之前的最后的准备工作
--------------------------------------------------------------------------------

在进入保护模式之前的最后一个函数调用发生在 [main.c](http://lxr.free-electrons.com/source/arch/x86/boot/main.c?v=3.18#L184) 中的 `go_to_protected_mode` 函数，就像这个函数的注释说的，这个函数将进行最后的准备工作然后进入保护模式，下面就让我们来具体看看最后的准备工作是什么，以及系统是如何切换到保护模式的。

`go_to_protected_mode` 函数本身定义在 [arch/x86/boot/pm.c](http://lxr.free-electrons.com/source/arch/x86/boot/pm.c?v=3.18#L104)。 这个函数调用了一些其他的函数进行最后的准备工作，下面就让我们来具体看看这些函数。

`go_to_protected_mode` 函数首先调用的是 `realmode_switch_hook` 函数，后者如果发现 `realmode_switch` hook， 那么将调用它并禁止 [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt) 中断，反之将直接禁止 NMI 中断。只有当 bootloader 运行在宿主环境下（比如在 DOS 下运行 ）， hook 才会被使用。你可以在 [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) (see **ADVANCED BOOT LOADER HOOKS**) 中详细了解 hook 函数的信息。

```c
/*
 * Invoke the realmode switch hook if present; otherwise
 * disable all interrupts.
 */
static void realmode_switch_hook(void)
{
	if (boot_params.hdr.realmode_swtch) {
		asm volatile("lcallw *%0"
			     : : "m" (boot_params.hdr.realmode_swtch)
			     : "eax", "ebx", "ecx", "edx");
	} else {
		asm volatile("cli");
		outb(0x80, 0x70); /* Disable NMI */
		io_delay();
	}
}
```

`realmode_switch` 指向了一个16 位实模式代码地址（远跳转指针），这个16位代码将禁止 NMI 中断。所以在上述代码中，如果 `realmode_swtch` hook 存在，代码是用了 `lcallw` 指令进行远函数调用。在我的环境中，因为不存在这个 hook ，所以代码是直接进入 `else` 部分进行了 NMI 的禁止：

```assembly
asm volatile("cli");
outb(0x80, 0x70);	/* Disable NMI */
io_delay();
```

上面的代码首先调用 `cli` 汇编指令清除了中断标志 `IF`，这条指令执行之后，外部中断就被禁止了，紧接着的下一行代码就禁止了 NMI 中断。

这里简单介绍一下中断。中断是由硬件或者软件产生的，当中断产生的时候， CPU 将得到通知。这个时候， CPU 将停止当前指令的执行，保存当前代码的环境，然后将控制权移交到中断处理程序。当中断处理程序完成之后，将恢复中断之前的运行环境，从而被中断的代码将继续运行。 NMI 中断是一类特殊的中断，往往预示着系统发生了不可恢复的错误，所以在正常运行的操作系统中，NMI 中断是不会被禁止的，但是在进入保护模式之前，由于特殊需求，代码禁止了这类中断。我们将在后续的章节中对中断做更多的介绍，这里就不展开了。

现在让我们回到上面的代码，在 NMI 中断被禁止之后（通过写 `0x80` 进 CMOS 地址寄存器 `0x70` ），函数接着调用了 `io_delay` 函数进行了短暂的延时以等待 I/O 操作完成。下面就是 `io_delay` 函数的实现：

```C
static inline void io_delay(void)
{
	const u16 DELAY_PORT = 0x80;
	asm volatile("outb %%al,%0" : : "dN" (DELAY_PORT));
}
```

对 I/O 端口 `0x80` 写入任何的字节都将得到 1 ms 的延时。在上面的代码中，代码将 `al` 寄存器中的值写到了这个端口。在这个 `io_delay` 调用完成之后， `realmode_switch_hook` 函数就完成了所有工作，下面让我们进入下一个函数。

下一个函数调用是 `enable_a20`，这个函数使能 [A20 line](http://en.wikipedia.org/wiki/A20_line)，你可以在 [arch/x86/boot/a20.c](http://lxr.free-electrons.com/source/arch/x86/boot/a20.c?v=3.18) 找到这个函数的定义，这个函数会尝试使用不同的方式来使能 A20 地址线。首先这个函数将调用 `a20_test_short`（该函数将调用 `a20_test` 函数） 来检测 A20 地址线是否已经被激活了：

```C
static int a20_test(int loops)
{
	int ok = 0;
	int saved, ctr;

	set_fs(0x0000);
	set_gs(0xffff);

	saved = ctr = rdfs32(A20_TEST_ADDR);

    while (loops--) {
		wrfs32(++ctr, A20_TEST_ADDR);
		io_delay();	/* Serialize and make delay constant */
		ok = rdgs32(A20_TEST_ADDR+0x10) ^ ctr;
		if (ok)
			break;
	}

	wrfs32(saved, A20_TEST_ADDR);
	return ok;
}
```

这个函数首先将 `0x0000` 放入 `FS` 寄存器，将 `0xffff` 放入 `GS` 寄存器。然后通过 `rdfs32` 函数调用，将 `A20_TEST_ADDR` 内存地址的内容放入 `saved` 和 `ctr` 变量。

接下来我们使用 `wrfs32` 函数将更新过的 `ctr` 的值写入 `fs:gs` ，接着延时 1ms， 然后从 `GS:A20_TEST_ADDR+0x10` 读取内容，如果该地址内容不为0，那么 A20 已经被激活。如果 A20 没有被激活，代码将尝试使用多种方法进行 A20 地址激活。其中的一种方法就是调用 BIOS `0X15` 中断激活 A20 地址线。

如果 `enabled_a20` 函数调用失败，显示一个错误消息并且调用 `die` 函数结束操作系统运行。`die` 函数定义在 [arch/x86/boot/header.S](http://lxr.free-electrons.com/source/arch/x86/boot/header.S?v=3.18):

```assembly
die:
	hlt
	jmp	die
	.size	die, .-die
```

A20 地址线被激活之后，`reset_coprocessor` 函数被调用：

 ```C
outb(0, 0xf0);
outb(0, 0xf1);
```

这个函数非常简单，通过将 `0` 写入 I/O 端口 `0xf0` 和 `0xf1` 以复位数字协处理器。

接下来 `mask_all_interrupts` 函数将被调用：

```C
outb(0xff, 0xa1);       /* Mask all interrupts on the secondary PIC */
outb(0xfb, 0x21);       /* Mask all but cascade on the primary PIC */
```

这个函数调用屏蔽了从中断控制器 (注：中断控制器的原文是 Programmable Interrupt Controller) 的所有中断，和主中断控制器上除IRQ2以外的所有中断（IRQ2是主中断控制器上的级联中断，所有从中断控制器的中断将通过这个级联中断报告给 CPU ）。

到这里位置，我们就完成了所有的准备工作，下面我们就将正式开始从实模式转换到保护模式。

设置中断描述符表
--------------------------------------------------------------------------------

现在内核将调用 `setup_idt` 方法来设置中断描述符表（ IDT ）：

```C
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

上面的代码使用 `lidtl` 指令将 `null_idt` 所指向的中断描述符表引入寄存器 IDT。由于 `null_idt` 没有设定中断描述符表的长度（长度为 0 ），所以这段指令执行之后，实际上没有任何中断调用被设置成功（所有中断调用都是空的），在后面的章节中我们将看到正确的设置。`null_idt` 是一个 `gdt_ptr` 结构的数据，这个结构的定义如下所示：

```C
struct gdt_ptr {
	u16 len;
	u32 ptr;
} __attribute__((packed));
```

在上面的定义中，我们可以看到上面这个结构包含一个 16 bit 的长度字段，和一个 32 bit 的指针字段。`__attribute__((packed))` 意味着这个结构就只包含 48 bit 信息（没有字节对齐优化）。在下面一节中，我们将看到相同的结构将被导入 `GDTR` 寄存器（如果你还记得上一章的内容，应该记得 GDTR 寄存器是 48 bit 长度的）。

设置全局描述符表
--------------------------------------------------------------------------------

在设置完中断描述符表之后，我们将使用 `setup_gdt` 函数来设置全局描述符表（关于全局描述符表，大家可以参考[上一章](linux-bootstrap-2.md#protected-mode) 的内容）。在 `setup_gdt` 函数中，使用 `boot_gdt` 数组定义了需要引入 GDTR 寄存器的段描述符信息：

```C
   //GDT_ENTRY_BOOT_CS 定义在http://lxr.free-electrons.com/source/arch/x86/include/asm/segment.h#L19 = 2
	static const u64 boot_gdt[] __attribute__((aligned(16))) = {
		[GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
		[GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
		[GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
	};
```

在上面的 `boot_gdt` 数组中，我们定义了代码，数据和 TSS 段(Task State Segment, 任务状态段)的段描述符，因为我们并没有设置任何的中断调用（记得上面说的 `null_idt`吗？），所以 TSS 段并不会被使用到。TSS 段存在的唯一目的就是让 Intel 处理器能够正确进入保护模式。下面让我们详细了解一下 `boot_gdt` 这个数组，首先，这个数组被 `__attribute__((aligned(16)))` 修饰，这就意味着这个数组将以 16 字节为单位对齐。让我们通过下面的例子来了解一下什么叫 16 字节对齐：

```C
#include <stdio.h>

struct aligned {
	int a;
}__attribute__((aligned(16)));

struct nonaligned {
	int b;
};

int main(void)
{
	struct aligned    a;
	struct nonaligned na;

	printf("Not aligned - %zu \n", sizeof(na));
	printf("Aligned - %zu \n", sizeof(a));

	return 0;
}
```

上面的代码可以看出，一旦指定了 16 字节对齐，即使结构中只有一个 `int` 类型的字段，整个结构也将占用 16 个字节：

```
$ gcc test.c -o test && test
Not aligned - 4
Aligned - 16
```

因为在 `boot_gdt` 的定义中， `GDT_ENTRY_BOOT_CS = 2`，所以在数组中有2个空项，第一项是一个空的描述符，第二项在代码中没有使用。在没有 `align 16` 之前，整个结构占用了（8*5=40）个字节，加了 `align 16` 之后，结构就占用了 48 字节 。

上面代码中出现的 `GDT_ENTRY` 是一个宏定义，这个宏接受 3 个参数（标志，基地址，段长度）来产生段描述符结构。让我们来具体分析上面数组中的代码段描述符（ `GDT_ENTRY_BOOT_CS` ）来看看这个宏是如何工作的，对于这个段，`GDT_ENTRY` 接受了下面 3 个参数：

* 基地址  - 0
* 段长度 - 0xfffff
* 标志 - 0xc09b

上面这些数字表明，这个段的基地址是 0， 段长度是 `0xfffff` （ 1 MB ），而标志字段展开之后是下面的二进制数据：

```
1100 0000 1001 1011
```

这些二进制数据的具体含义如下:

* 1    - (G) 这里为 1，表示段的实际长度是 `0xfffff * 4kb ` = `4GB`
* 1    - (D) 表示这个段是一个32位段
* 0    - (L) 这个代码段没有运行在 long mode
* 0    - (AVL) Linux 没有使用
* 0000 - 段长度的4个位
* 1    - (P) 段已经位于内存中
* 00   - (DPL) - 段优先级为0
* 1    - (S) 说明这个段是一个代码或者数据段
* 101  - 段类型为可执行/可读
* 1    - 段可访问

关于段描述符的更详细的信息你可以从上一章中获得 [上一章](linux-bootstrap-2.md)，你也可以阅读 [Intel® 64 and IA-32 Architectures Software Developer's Manuals 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)获取全部信息。

在定义了数组之后，代码将获取 GDT 的长度：

```C
gdt.len = sizeof(boot_gdt)-1;
```

接下来是将 GDT 的地址放入 gdt.ptr 中：

```C
gdt.ptr = (u32)&boot_gdt + (ds() << 4);
```

这里的地址计算很简单，因为我们还在实模式，所以就是 （ ds << 4 + 数组起始地址）。

最后通过执行 `lgdtl` 指令将 GDT 信息写入 GDTR 寄存器：

```C
asm volatile("lgdtl %0" : : "m" (gdt));
```

切换进入保护模式
--------------------------------------------------------------------------------

`go_to_protected_mode` 函数在完成 IDT, GDT 初始化，并禁止了 NMI 中断之后，将调用 `protected_mode_jump` 函数完成从实模式到保护模式的跳转：

```C
protected_mode_jump(boot_params.hdr.code32_start, (u32)&boot_params + (ds() << 4));
```

`protected_mode_jump` 函数定义在 [arch/x86/boot/pmjump.S](http://lxr.free-electrons.com/source/arch/x86/boot/pmjump.S?v=3.18#L26)，它接受下面2个参数:

* 保护模式代码的入口
* `boot_params` 结构的地址

第一个参数保存在 `eax` 寄存器，而第二个参数保存在 `edx` 寄存器。

代码首先在 `boot_params` 地址放入 `esi` 寄存器，然后将 `cs` 寄存器内容放入 `bx` 寄存器，接着执行 `bx << 4 + 标号为2的代码的地址`，这样一来 `bx` 寄存器就包含了标号为2的代码的地址。接下来代码将把数据段索引放入 `cx` 寄存器，将  TSS 段索引放入 `di` 寄存器：

```assembly
movw	$__BOOT_DS, %cx
movw	$__BOOT_TSS, %di
```

就像前面我们看到的 `GDT_ENTRY_BOOT_CS` 的值为2，每个段描述符都是 8 字节，所以 `cx` 寄存器的值将是 `2*8 = 16`，`di` 寄存器的值将是 `4*8 =32`。

接下来，我们通过设置 `CR0` 寄存器相应的位使 CPU 进入保护模式：

```assembly
movl	%cr0, %edx
orb	$X86_CR0_PE, %dl
movl	%edx, %cr0
```

在进入保护模式之后，通过一个长跳转进入 32 位代码：

```assembly
	.byte	0x66, 0xea
2:	.long	in_pm32
	.word	__BOOT_CS ;(GDT_ENTRY_BOOT_CS*8) = 16，段描述符表索引
```

这段代码中
* `0x66` 操作符前缀允许我们混合执行 16 位和 32 位代码
* `0xea` - 跳转指令的操作符
* `in_pm32` 跳转地址偏移
* `__BOOT_CS` 代码段描述符索引

在执行了这个跳转命令之后，我们就在保护模式下执行代码了：

```assembly
.code32
.section ".text32","ax"
```

保护模式代码的第一步就是重置所有的段寄存器（除了 `CS` 寄存器）:

```assembly
GLOBAL(in_pm32)
movl	%ecx, %ds
movl	%ecx, %es
movl	%ecx, %fs
movl	%ecx, %gs
movl	%ecx, %ss
```

还记得我们在实模式代码中将 `$__BOOT_DS` （数据段描述符索引）放入了 `cx` 寄存器，所以上面的代码设置所有段寄存器（除了 `CS` 寄存器）指向数据段。接下来代码将所有的通用寄存器清 0 ：

```assembly
xorl	%ecx, %ecx
xorl	%edx, %edx
xorl	%ebx, %ebx
xorl	%ebp, %ebp
xorl	%edi, %edi
```

最后使用长跳转跳入正在的 32 位代码（通过参数传入的地址）

```
jmpl	*%eax ;?jmpl cs:eax?
```

到这里，我们就进入了保护模式开始执行代码了，下一章我们将分析这段 32 位代码到底做了些什么。

结论
--------------------------------------------------------------------------------

这章到这里就结束了，在下一章中我们将具体介绍这章最后跳转到的 32 位代码，并且了解系统是如何进入  [long mode](http://en.wikipedia.org/wiki/Long_mode)的。

如果你有任何的问题或者建议，你可以留言，也可以直接发消息给我[twitter](https://twitter.com/0xAX).

**如果你发现文中描述有任何问题，请提交一个 PR 到 [linux-insides-zh](https://github.com/MintCN/linux-insides-zh) 。**

链接
--------------------------------------------------------------------------------

* [VGA](http://en.wikipedia.org/wiki/Video_Graphics_Array)
* [VESA BIOS Extensions](http://en.wikipedia.org/wiki/VESA_BIOS_Extensions)
* [Data structure alignment](http://en.wikipedia.org/wiki/Data_structure_alignment)
* [Non-maskable interrupt](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [A20](http://en.wikipedia.org/wiki/A20_line)
* [GCC designated inits](https://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Designated-Inits.html)
* [GCC type attributes](https://gcc.gnu.org/onlinedocs/gcc/Type-Attributes.html)
* [Previous part](linux-bootstrap-2.md)

