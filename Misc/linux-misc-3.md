介绍
---------------

在写 [linux-insides](http://0xax.gitbooks.io/linux-insides/content/) 一书的过程中，我收到了很多邮件询问关于[链接器](https://zh.wikipedia.org/wiki/%E9%93%BE%E6%8E%A5%E5%99%A8)和链接器脚本的问题。所以我决定写这篇文章来介绍链接器和目标文件的链接方面的知识。

如果我们打开维基百科的 `链接器` 页，我们将会看到如下定义：

>在计算机科学中，链接器（英文：Linker），是一个计算机程序，它将一个或多个由编译器生成的目标文件链接为一个单独的可执行文件，库文件或者另外一个目标文件

如果你曾经用 C 写过至少一个程序，那你就已经见过以 `*.o` 扩展名结尾的文件了。这些文件是[目标文件](https://en.wikipedia.org/wiki/Object_file)。目标文件是一块块的机器码和数据，其数据包含了引用其他目标文件或库的数据和函数的占位符地址，也包括其自身的函数和数据列表。链接器的主要目的就是收集/处理每一个目标文件的代码和数据，将它们转成最终的可执行文件或者库。在这篇文章里，我们会试着研究这个流程的各个方面。开始吧。

链接流程
---------------

让我们按以下结构创建一个项目：

```
*-linkers
*--main.c
*--lib.c
*--lib.h
```

我们的 `main.c` 源文件包含了：

```C
#include <stdio.h>

#include "lib.h"

int main(int argc, char **argv) {
	printf("factorial of 5 is: %d\n", factorial(5));
	return 0;
}
```

`lib.c` 文件包含了：

```C
int factorial(int base) {
	int res,i = 1;
	
	if (base == 0) {
		return 1;
	}

	while (i <= base) {
		res *= i;
		i++;
	}

	return res;
}
```

`lib.h` 文件包含了：

```C
#ifndef LIB_H
#define LIB_H

int factorial(int base);

#endif
```

现在让我们用以下命令单独编译 `main.c` 源码：

```
$ gcc -c main.c
```

如果我们用 `nm` 工具查看输出的目标文件，我们将会看到如下输出：

```
$ nm -A main.o
main.o:                 U factorial
main.o:0000000000000000 T main
main.o:                 U printf
```
`nm` 工具让我们能够看到给定目标文件的符号表列表。其包含了三列：第一列是该目标文件的名称和解析得到的符号地址。第二列包含了一个表示该符号状态的字符。这里 `U` 表示 `未定义`， `T` 表示该符号被置于 `.text` 段。在这里， `nm` 工具向我们展示了 `main.c` 文件里包含的三个符号：

* `factorial` - 在 `lib.c` 文件中定义的阶乘函数。因为我们只编译了 `main.c`，所以其不知道任何有关 `lib.c` 文件的事；
* `main` - 主函数;
* `printf` - 来自 [glibc](https://en.wikipedia.org/wiki/GNU_C_Library) 库的函数。 `main.c` 同样不知道任何与其相关的事。

目前我们可以从 `nm` 的输出中了解哪些事情呢？ `main.o` 目标文件包含了在地址 `0000000000000000` 处的本地变量 `main` （在被链接后其将会被赋予正确的地址），以及两个无法解析的符号。我们可以从 `main.o` 的反汇编输出中了解这些信息：

```
$ objdump -S main.o

main.o:     file format elf64-x86-64
Disassembly of section .text:

0000000000000000 <main>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	48 83 ec 10          	sub    $0x10,%rsp
   8:	89 7d fc             	mov    %edi,-0x4(%rbp)
   b:	48 89 75 f0          	mov    %rsi,-0x10(%rbp)
   f:	bf 05 00 00 00       	mov    $0x5,%edi
  14:	e8 00 00 00 00       	callq  19 <main+0x19>
  19:	89 c6                	mov    %eax,%esi
  1b:	bf 00 00 00 00       	mov    $0x0,%edi
  20:	b8 00 00 00 00       	mov    $0x0,%eax
  25:	e8 00 00 00 00       	callq  2a <main+0x2a>
  2a:	b8 00 00 00 00       	mov    $0x0,%eax
  2f:	c9                   	leaveq 
  30:	c3                   	retq   
```

这里我们只关注两个 `callq` 操作。这两个 `callq` 操作包含了 `链接器存根`，或者函数的名称和其相对当前的下一条指令的偏移。这些存根将会被更新到函数的真实地址。我们可以在下面的 `objdump` 输出看到这些函数的名字：

```
$ objdump -S -r main.o

...
  14:	e8 00 00 00 00       	callq  19 <main+0x19>
  15: R_X86_64_PC32	               factorial-0x4
  19:	89 c6                	mov    %eax,%esi
...
  25:	e8 00 00 00 00       	callq  2a <main+0x2a>
  26:   R_X86_64_PC32	               printf-0x4
  2a:	b8 00 00 00 00       	mov    $0x0,%eax
...
```

`objdump` 工具中的 `-r` 或 `--reloc ` 选项会打印文件的 `重定位` 条目。现在让我们更加深入重定位流程。

重定位
------------

重定位是连接符号引用和符号定义的流程。让我们看看前一段 `objdump` 的输出：

```
  14:	e8 00 00 00 00       	callq  19 <main+0x19>
  15:   R_X86_64_PC32	               factorial-0x4
  19:	89 c6                	mov    %eax,%esi
```

注意第一行的 `e8 00 00 00 00` 。`e8` 是 `call` 的 [操作码](https://en.wikipedia.org/wiki/Opcode) ，这一行的剩余部分是一个相对偏移。所以 `e8 00 00 00` 包含了一个单字节操作码，跟着一个四字节地址。注意 `00 00 00 00` 是 4 个字节。为什么只有 4 字节 而不是 `x86_64` 64 位机器上的 8 字节地址？其实我们用了 `-mcmodel=small` 选项来编译 `main.c` ！从 `gcc` 的指南上看：

```
-mcmodel=small

为小代码模型生成代码: 目标程序及其符号必须被链接到低于 2GB 的地址空间。指针是 64 位的。程序可以被动态或静态的链接。这是默认的代码模型。
```

当然，我们在编译时并没有将这一选项传给 `gcc` ，但是这是默认的。从上面摘录的 `gcc` 指南我们知道，我们的程序会被链接到低于 2 GB 的地址空间。因此 4 字节已经足够。所以我们有了 `call` 指令和一个未知的地址。当我们编译 `main.c` 以及它的依赖形成一个可执行文件时，关注阶乘函数的调用，我们看到：

```
$ gcc main.c lib.c -o factorial | objdump -S factorial | grep factorial

factorial:     file format elf64-x86-64
...
...
0000000000400506 <main>:
	40051a:	e8 18 00 00 00       	callq  400537 <factorial>
...
...
0000000000400537 <factorial>:
	400550:	75 07                	jne    400559 <factorial+0x22>
	400557:	eb 1b                	jmp    400574 <factorial+0x3d>
	400559:	eb 0e                	jmp    400569 <factorial+0x32>
	40056f:	7e ea                	jle    40055b <factorial+0x24>
...
...
```

在前面的输出中我们可以看到， `main` 函数的地址是 `0x0000000000400506`。为什么它不是从 `0x0` 开始的呢？你可能已经知道标准 C 程序是使用 `glibc` 的 C 标准库链接的（假设参数 `-nostdlib` 没有被传给 `gcc` ）。编译后的程序代码包含了用于在程序启动时初始化程序中数据的构造函数。这些函数需要在程序启动前被调用，或者说在 `main` 函数之前被调用。为了让初始化和终止函数起作用，编译器必须在汇编代码中输出一些让这些函数在正确时间被调用的代码。执行这个程序将会启动位于特殊的 `.init` 段的代码。我们可以从以下的 objdump 输出中看出：

```
objdump -S factorial | less

factorial:     file format elf64-x86-64

Disassembly of section .init:

00000000004003a8 <_init>:
  4003a8:       48 83 ec 08             sub    $0x8,%rsp
  4003ac:       48 8b 05 a5 05 20 00    mov    0x2005a5(%rip),%rax        # 600958 <_DYNAMIC+0x1d0>
```

注意其开始于相对 `glibc` 代码偏移 `0x00000000004003a8` 的地址。我们也可以运行 `readelf` ，在 [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 输出中检查： 

```
$ readelf -d factorial | grep \(INIT\)
 0x000000000000000c (INIT)               0x4003a8
 ```

所以， `main` 函数的地址是 `0000000000400506` ，为相对于 `.init` 段的偏移地址。我们可以从输出中看出，`factorial` 函数的地址是 `0x0000000000400537` ，并且现在调用 `factorial` 函数的二进制代码是 `e8 18 00 00 00`。我们已经知道 `e8` 是 `call` 指令的操作码，接下来的 `18 00 00 00` （注意 `x86_64`中地址是小头存储的，所以是 `00 00 00 18` ）是从 `callq` 到 `factorial` 函数的偏移。

```python
>>> hex(0x40051a + 0x18 + 0x5) == hex(0x400537)
True
```

所以我们把 `0x18`和 `0x5` 加到 `call` 指令的地址上。偏移是从接下来一条指令开始算起的。我们的调用指令是 5 字节长（`e8 18 00 00 00`）并且 `0x18` 是从 `factorial` 函数之后的调用算起的偏移。编译器一般按程序地址从零开始创建目标文件。但是如果程序由多个目标文件生成，这些地址会重叠。

我们在这一段看到的是 `重定位` 流程。这个流程为程序中各个部分赋予加载地址，调整程序中的代码和数据以反映出赋值的地址。

好了，现在我们知道了一点关于链接器和重定位的知识，是时候通过链接我们的目标文件来来学习更多关于链接器的知识了。

GNU 链接器
-----------------

如标题所说，在这篇文章中，我将会使用 [GNU 链接器](https://en.wikipedia.org/wiki/GNU_linker) 或者说 `ld` 。当然我们可以使用 `gcc` 来链接我们的 `factorial` 项目： 

```
$ gcc main.c lib.o -o factorial
```

在这之后，作为结果我们将会得到可执行文件—— `factorial`：

```
./factorial 
factorial of 5 is: 120
```

但是 `gcc` 不会链接目标文件。取而代之，其会使用 `GUN ld` 链接器的包装—— `collect2`。 

```
~$ /usr/lib/gcc/x86_64-linux-gnu/4.9/collect2 --version
collect2 version 4.9.3
/usr/bin/ld --version
GNU ld (GNU Binutils for Debian) 2.25
...
...
...
```

好，我们可以使用 gcc 并且其会为我们的程序生成可执行文件。但是让我们看看如何使用 `GUN ld` 实现相同的目的。首先，让我们尝试用如下样例链接这些目标文件：

```
ld main.o lib.o -o factorial
```

尝试一下，你将会得到如下错误：

```
$ ld main.o lib.o -o factorial
ld: warning: cannot find entry symbol _start; defaulting to 00000000004000b0
main.o: In function `main':
main.c:(.text+0x26): undefined reference to `printf'
```

这里我们可以看到两个问题：

* 链接器无法找到 `_start` 符号；
* 链接器对 `printf` 一无所知。

首先，让我们尝试理解好像是我们程序运行所需要的 `_start` 入口符号是什么。当我开始学习编程时，我知道了 `main` 函数是程序的入口点。我认为你们也是如此认为的 :) 但实际上这不是入口点，`_start` 才是。 `_start` 符号被 `crt1.0` 所定义。我们可以用如下指令发现它：

```
$ objdump -S /usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o

/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <_start>:
   0:	31 ed                	xor    %ebp,%ebp
   2:	49 89 d1             	mov    %rdx,%r9
   ...
   ...
   ...
```

我们将该目标文件作为第一个参数传递给 `ld` 指令（如上所示）。现在让我们尝试链接它，会得到如下结果：

```
ld /usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o \
main.o lib.o -o factorial

/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o: In function `_start':
/tmp/buildd/glibc-2.19/csu/../sysdeps/x86_64/start.S:115: undefined reference to `__libc_csu_fini'
/tmp/buildd/glibc-2.19/csu/../sysdeps/x86_64/start.S:116: undefined reference to `__libc_csu_init'
/tmp/buildd/glibc-2.19/csu/../sysdeps/x86_64/start.S:122: undefined reference to `__libc_start_main'
main.o: In function `main':
main.c:(.text+0x26): undefined reference to `printf'
```

不幸的是，我们甚至会看到更多报错。我们可以在这里看到关于未定义 `printf` 的旧错误以及另外三个未定义的引用：

* `__libc_csu_fini`
* `__libc_csu_init`
* `__libc_start_main`

`_start` 符号被定义在 `glibc` 源文件的汇编文件 [sysdeps/x86_64/start.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/start.S;h=0d27a38e9c02835ce17d1c9287aa01be222e72eb;hb=HEAD) 中。我们可以在那里找到如下汇编代码： 

```assembly
mov $__libc_csu_fini, %R8_LP
mov $__libc_csu_init, %RCX_LP
...
call __libc_start_main
```

这里我们传递了 `.init` 和 `.fini` 段的入口点地址，它们包含了程序开始和结束时被执行的代码。并且在结尾我们看到对我们程序的 `main` 函数的调用。这三个符号被定义在源文件 [csu/elf-init.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=csu/elf-init.c;hb=1d4bbc54bd4f7d85d774871341b49f4357af1fb7) 中。如下两个目标文件：

* `crtn.o`;
* `crti.o`.

定义了 .init 和 .fini 段的开端和尾声（分别为符号 `_init` 和 `_fini` ）。

`crtn.o` 目标文件包含了 `.init` 和 `.fini` 这些段： 

```
$ objdump -S /usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crtn.o

0000000000000000 <.init>:
   0:	48 83 c4 08          	add    $0x8,%rsp
   4:	c3                   	retq   

Disassembly of section .fini:

0000000000000000 <.fini>:
   0:	48 83 c4 08          	add    $0x8,%rsp
   4:	c3                   	retq   
```

且 `crti.o` 目标文件包含了符号 `_init` 和 `_fini`。让我们再次尝试链接这两个目标文件： 

```
$ ld \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crti.o \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crtn.o main.o lib.o \
-o factorial
```

当然，我们会得到相同的错误。现在我们需要把 `-lc` 选项传递给 `ld` 。这个选项将会在环境变量 `$LD_LIBRARY_PATH` 指定的目录中搜索标准库。让我们再次尝试用 `-lc` 选项链接：

```
$ ld \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crti.o \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crtn.o main.o lib.o -lc \
-o factorial
```

最后我们获得了一个可执行文件，但是如果我们尝试运行它，我们会遇到奇怪的结果：

```
$ ./factorial 
bash: ./factorial: No such file or directory
```

这里除了什么问题？让我们用 [readelf](https://sourceware.org/binutils/docs/binutils/readelf.html) 工具看看这个可执行文件：

```
$ readelf -l factorial 

Elf file type is EXEC (Executable file)
Entry point 0x4003c0
There are 7 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040
                 0x0000000000000188 0x0000000000000188  R E    8
  INTERP         0x00000000000001c8 0x00000000004001c8 0x00000000004001c8
                 0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x0000000000000610 0x0000000000000610  R E    200000
  LOAD           0x0000000000000610 0x0000000000600610 0x0000000000600610
                 0x00000000000001cc 0x00000000000001cc  RW     200000
  DYNAMIC        0x0000000000000610 0x0000000000600610 0x0000000000600610
                 0x0000000000000190 0x0000000000000190  RW     8
  NOTE           0x00000000000001e4 0x00000000004001e4 0x00000000004001e4
                 0x0000000000000020 0x0000000000000020  R      4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     10

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.ABI-tag .hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .text .fini .rodata .eh_frame 
   03     .dynamic .got .got.plt .data 
   04     .dynamic 
   05     .note.ABI-tag 
   06     
```

注意这奇怪的一行：

```
  INTERP         0x00000000000001c8 0x00000000004001c8 0x00000000004001c8
                 0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

`elf` 文件的 `.interp` 段保存了一个程序解释器的路径名，或者说 `.interp` 段就包含了一个动态链接器名字的 `ascii` 字符串。动态链接器是 Linux 的一部分，其通过将库的内容从磁盘复制到内存中以加载和链接一个可执行文件被执行所需要的动态链接库。我们可以从 `readelf` 命令的输出中看到，针对 `x86_64` 架构，其被放在 `/lib64/ld-linux-x86-64.so.2`。现在让我们把 `ld-linux-x86-64.so.2` 的路径和 `-dynamic-linker` 选项一起传递给 `ld` 调用，然后会看到如下结果：

```
$ gcc -c main.c lib.c

$ ld \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crti.o \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crtn.o main.o lib.o \
-dynamic-linker /lib64/ld-linux-x86-64.so.2 \
-lc -o factorial
```

现在我们可以像普通可执行文件一样执行它了：

```
$ ./factorial

factorial of 5 is: 120
```

成功了！在第一行，我们把源文件 `main.c` 和 `lib.c` 编译成目标文件。执行 `gcc` 之后我们将会获得 `main.o` 和 `lib.o`： 

```
$ file lib.o main.o
lib.o:  ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
main.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```

在这之后，我们用所需的系统目标文件和库连链接我们的程序。我们刚看了一个简单的关于如何用 `gcc` 编译器和 `GNU ld` 链接器编译和链接一个 C 程序的样例。在这个样例中，我们使用了一些 `GNU linker` 的命令行选项，但是除了 `-o`、`-dynamic-linker` 等，它还支持其他很多选项。此外，`GNU ld` 还拥有其自己的语言来控制链接过程。在接下来的两个段落中我们深入讨论。

实用的 GNU 链接器命令行选项
----------------------------------------------

正如我之前所说，你也可以从 `GNU linker` 的指南看到，其拥有大量的命令行选项。我们已经在这篇文章见到一些： `-o <output>` - 告诉 `ld` 将链接结果输出成一个叫做 `output` 的文件，`-l<name>` - 通过文件名添加指定存档或者目标文件，`-dynamic-linker` 通过名字指定动态链接器。当然， `ld` 支持更多选项，让我们看看其中的一些。

第一个实用的选项是 `@file` 。在这里 `file` 指定了命令行选项将读取的文件名。比如我们可以创建一个叫做 `linker.ld` 的文件，把我们上一个例子里面的命令行参数放进去然后执行：

```
$ ld @linker.ld
```

下一个命令行选项是 `-b` 或 `--format`。这个命令行选项指定了输入的目标文件的格式是 `ELF`, `DJGPP/COFF` 等。针对输出文件也有相同功能的选项 `--oformat=output-format` 。

下一个命令行选项是 `--defsym` 。该选项的完整格式是 `--defsym=symbol=expression` 。它允许在输出文件中创建包含了由表达式给出了绝对地址的全局符号。在下面的例子中，我们会发现这个命令行选项很实用：在 Linux 内核源码中关于 ARM 架构内核解压的 Makefile - [arch/arm/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/master/arch/arm/boot/compressed/Makefile)，我们可以找到如下定义：

```
LDFLAGS_vmlinux = --defsym _kernel_bss_size=$(KBSS_SZ)
```

正如我们所知，其在输出文件中用 `.bss` 段的大小定义了 `_kernel_bss_size` 符号。这个符号将会作为第一个 [汇编文件](https://github.com/torvalds/linux/blob/master/arch/arm/boot/compressed/head.S) 在内核解压阶段被执行：

```assembly
ldr r5, =_kernel_bss_size
```

下一个选项是 `-shared` ，其允许我们创建共享库。`-M` 或者说 `-map <filename>` 命令行选项会打印带符号信息的链接映射内容。在这里是：

```
$ ld -M @linker.ld
...
...
...
.text           0x00000000004003c0      0x112
 *(.text.unlikely .text.*_unlikely .text.unlikely.*)
 *(.text.exit .text.exit.*)
 *(.text.startup .text.startup.*)
 *(.text.hot .text.hot.*)
 *(.text .stub .text.* .gnu.linkonce.t.*)
 .text          0x00000000004003c0       0x2a /usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o
...
...
...
 .text          0x00000000004003ea       0x31 main.o
                0x00000000004003ea                main
 .text          0x000000000040041b       0x3f lib.o
                0x000000000040041b                factorial
```

当然，`GNU 链接器` 支持标准的命令行选项：`--help` 和 `--version` 能够打印 `ld` 的命令帮助、使用方法和版本。以上就是所有关于 `GNU 链接器` 命令行选项的内容。当然这不是 `ld` 工具支持的所有命令行选项。你可以在指南中找到 `ld` 工具的完整文档。

链接器控制语言
----------------------------------------------

如我之前所说， `ld` 支持它自己的语言。它接受由一种 AT&T 链接器控制语法的超集编写的链接器控制语言文件，以提供对链接过程明确且完全的控制。接下来让我们关注其中细节。

我们可以通过链接器语言控制：

* 输入文件；
* 输出文件；
* 文件格式；
* 段的地址；
* 其他更多...

用链接器控制语言编写的命令通常被放在一个被称作链接器脚本的文件中。我们可以通过命令行选项 `-T` 将其传递给 `ld` 。一个链接器脚本的主要命令是 `SECTIONS` 指令。每个链接器脚本必须包含这个指令，并且其决定了输出文件的 `映射` 。特殊变量 `.` 包含了当前输出的位置。让我们写一个简单的汇编程序，然后看看如何使用链接器脚本来控制程序的链接。我们将会使用一个 hello world 程序作为样例。

```assembly
.data
	msg	.ascii "hello, world!",`\n`

.text

	global	_start
  
_start:
	mov    $1,%rax
	mov    $1,%rdi
	mov    $msg,%rsi
	mov    $14,%rdx
	syscall

	mov    $60,%rax
	mov    $0,%rdi
	syscall
```

我们可以用以下命令编译并链接：

```
$ as -o hello.o hello.asm
$ ld -o hello hello.o
```

我们的程序包含了两个段： `.text` 包含了程序的代码， `.data` 段包含了被初始化的变量。让我们写一个简单的链接脚本，然后尝试用它来链接我们的 `hello.asm` 汇编文件。我们的脚本是：

```
/*
 * Linker script for the factorial
 */
OUTPUT(hello) 
OUTPUT_FORMAT("elf64-x86-64")
INPUT(hello.o)

SECTIONS
{
	. = 0x200000;
	.text : {
	      *(.text)
	}

	. = 0x400000;
	.data : {
	      *(.data)
	}
}
```

在前三行你可以看到 `C` 风格的注释。之后是 `OUTPUT` 和 `OUTPUT_FORMAT` 命令，指定了我们的可执行文件名称和格式。下一个指令，`INPUT`，指定了给 `ld` 的输入文件。接下来，我们可以看到主要的 `SECTIONS` 指令，正如我写的，它是必须存在于每个链接器脚本中。`SECTIONS` 命令表示了输出文件中的段的集合和顺序。在 `SECTIONS` 命令的开头，我们可以看到一行 `. = 0x200000` 。我上面已经写过，`.` 命令指向输出中的当前位置。这一行说明代码段应该被加载到地址 `0x200000`。`. = 0x400000`一行说明数据段应该被加载到地址`0x400000` 。`. = 0x200000`之后的第二行定义 `.text` 作为输出段。我们可以看到其中的 `*(.text)` 表达式。 `*` 符号是一个匹配任意文件名的通配符。换句话说，`*(.text)` 表达式代表所有输入文件中的所有 `.text` 输入段。在我们的样例中，我们可以将其重写为 `hello.o(.text)` 。在地址计数器 `. = 0x400000` 之后，我们可以看到数据段的定义。

我们可以用以下语句进行编译和链接：

```
$ as -o hello.o hello.S && ld -T linker.script && ./hello
hello, world!
```

如果我们用 `objdump` 工具深入查看，我们可以看到 `.text` 段从地址 `0x200000` 开始， `.data` 段从 `0x400000` 开始：

```
$ objdump -D hello

Disassembly of section .text:

0000000000200000 <_start>:
  200000:	48 c7 c0 01 00 00 00 	mov    $0x1,%rax
  ...

Disassembly of section .data:

0000000000400000 <msg>:
  400000:	68 65 6c 6c 6f       	pushq  $0x6f6c6c65
  ...
```

除了我们已经看到的命令，另外还有一些。首先是 `ASSERT(exp, message)` ，保证给定的表达式不为零。如果为零，那么链接器会退出同时返回错误码，打印错误信息。如果你已经阅读了 [linux-insides](https://xinqiu.gitbooks.io/linux-insides-cn/content/) 的 Linux 内核启动流程，你或许知道 Linux 内核的设置头的偏移为 `0x1f1`。在 Linux 内核的链接器脚本中，我们可以看到下面的校验：

```
. = ASSERT(hdr == 0x1f1, "The setup header has the wrong offset!");
```

`INCLUDE filename` 允许我们在当前的链接器脚本中包含外部符号。我们可以在一个链接器脚本中给一个符号赋值。 `ld` 支持一些赋值操作符：

* symbol = expression   ;
* symbol += expression  ;
* symbol -= expression  ;
* symbol *= expression  ;
* symbol /= expression  ;
* symbol <<= expression ;
* symbol >>= expression ;
* symbol &= expression  ;
* symbol |= expression  ;

正如你注意到的，所有操作符都是 C 赋值操作符。比如我们可以在我们的链接器脚本中使用：

```
START_ADDRESS = 0x200000;
DATA_OFFSET   = 0x200000;

SECTIONS
{
	. = START_ADDRESS;
	.text : {
	      *(.text)
	}

	. = START_ADDRESS + DATA_OFFSET;
	.data : {
	      *(.data)
	}
}
```

你可能已经注意到了链接器脚本中表达式的语法和 C 表达式相同。除此之外，这个链接控制语言还支持如下内嵌函数：

* `ABSOLUTE` - 返回给定表达式的绝对值；
* `ADDR` - 接受段，返回其地址；
* `ALIGN` - 返回和给定表达式下一句的边界对齐的位置计数器（ `.` 操作符）的值；
* `DEFINED` - 如果给定符号在全局符号表中，返回 `1`，否则 `0`；
* `MAX` and `MIN` - 返回两个给定表达式中的最大、最小值；
* `NEXT` - 返回一个是当前表达式倍数的未分配地址；
* `SIZEOF` - 返回给定名字的段以字节计数的大小。

以上就是全部了。

总结
-----------------

这是关于链接器文章的结尾。在这篇文章中，我们已经学习了很多关于链接器的知识，比如什么是链接器、为什么需要它、如何使用它等等...

如果你发现文中描述有任何问题，请提交一个 PR 到 [linux-insides-zh](https://github.com/MintCN/linux-insides-zh) 。

相关链接
-----------------

* [Book about Linux kernel insides](http://0xax.gitbooks.io/linux-insides/content/)
* [linker](https://en.wikipedia.org/wiki/Linker_%28computing%29)
* [object files](https://en.wikipedia.org/wiki/Object_file)
* [glibc](https://en.wikipedia.org/wiki/GNU_C_Library)
* [opcode](https://en.wikipedia.org/wiki/Opcode)
* [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
* [GNU linker](https://en.wikipedia.org/wiki/GNU_linker)
* [My posts about assembly programming for x86_64](http://0xax.github.io/categories/assembly/)
* [readelf](https://sourceware.org/binutils/docs/binutils/readelf.html)
