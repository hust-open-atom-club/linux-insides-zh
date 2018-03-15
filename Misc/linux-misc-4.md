用户空间的程序启动过程
================================================================================

简介
--------------------------------------------------------------------------------

虽然 [linux-insides-zh](https://www.gitbook.com/book/xinqiu/linux-insides-cn/details) 大多描述的是内核相关的东西，但是我已经决定写一个大多与用户空间相关的部分。

[系统调用](https://zh.wikipedia.org/wiki/%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8)章节的[第四部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/linux-syscall-4.html)已经描述了当我们想运行一个程序， Linux 内核的行为。这部分我想研究一下从用户空间的角度，当我们在 Linux 系统上运行一个程序，会发生什么。

我不知道你知识储备如何，但是在我的大学时期我学到，一个 `C` 程序从一个叫做 main 的函数开始执行。而且，这是部分正确的。每时每刻，当我们开始写一个新的程序时，我们从下面的实例代码开始编程：

```C
int main(int argc, char *argv[]) {
	// Entry point is here
}
```

但是你如何对于底层编程感兴趣的话，可能你已经知道 `main` 函数并不是程序的真正入口。如果你在调试器中看了下面这个简单程序，就可以很确信这一点：

```C
int main(int argc, char *argv[]) {
	return 0;
}
```

让我们来编译并且在 [gdb](https://www.gnu.org/software/gdb/) 中运行这个程序：

```
$ gcc -ggdb program.c -o program
$ gdb ./program
The target architecture is assumed to be i386:x86-64:intel
Reading symbols from ./program...done.
```

让我们在 gdb 中执行 `info files` 这个指令。这个指令会打印关于被不同段占据的内存和调试目标的信息。

```
(gdb) info files
Symbols from "/home/alex/program".
Local exec file:
	`/home/alex/program', file type elf64-x86-64.
	Entry point: 0x400430
	0x0000000000400238 - 0x0000000000400254 is .interp
	0x0000000000400254 - 0x0000000000400274 is .note.ABI-tag
	0x0000000000400274 - 0x0000000000400298 is .note.gnu.build-id
	0x0000000000400298 - 0x00000000004002b4 is .gnu.hash
	0x00000000004002b8 - 0x0000000000400318 is .dynsym
	0x0000000000400318 - 0x0000000000400357 is .dynstr
	0x0000000000400358 - 0x0000000000400360 is .gnu.version
	0x0000000000400360 - 0x0000000000400380 is .gnu.version_r
	0x0000000000400380 - 0x0000000000400398 is .rela.dyn
	0x0000000000400398 - 0x00000000004003c8 is .rela.plt
	0x00000000004003c8 - 0x00000000004003e2 is .init
	0x00000000004003f0 - 0x0000000000400420 is .plt
	0x0000000000400420 - 0x0000000000400428 is .plt.got
	0x0000000000400430 - 0x00000000004005e2 is .text
	0x00000000004005e4 - 0x00000000004005ed is .fini
	0x00000000004005f0 - 0x0000000000400610 is .rodata
	0x0000000000400610 - 0x0000000000400644 is .eh_frame_hdr
	0x0000000000400648 - 0x000000000040073c is .eh_frame
	0x0000000000600e10 - 0x0000000000600e18 is .init_array
	0x0000000000600e18 - 0x0000000000600e20 is .fini_array
	0x0000000000600e20 - 0x0000000000600e28 is .jcr
	0x0000000000600e28 - 0x0000000000600ff8 is .dynamic
	0x0000000000600ff8 - 0x0000000000601000 is .got
	0x0000000000601000 - 0x0000000000601028 is .got.plt
	0x0000000000601028 - 0x0000000000601034 is .data
	0x0000000000601034 - 0x0000000000601038 is .bss
```

注意 `Entry point: 0x400430` 这一行。现在我们知道我们程序入口点的真正地址。让我们在这个地址下一个断点，然后运行我们的程序，看看会发生什么：

```
(gdb) break *0x400430
Breakpoint 1 at 0x400430
(gdb) run
Starting program: /home/alex/program 

Breakpoint 1, 0x0000000000400430 in _start ()
```

有趣。我们并没有看见 `main` 函数的执行，但是我们看见另外一个函数被调用。这个函数是 `_start` 而且根据调试器展现给我们看的，它是我们程序的真正入口。那么，这个函数是从哪里来的，又是谁调用了这个 `main` 函数，什么时候调用的。我会在后续部分尝试回答这些问题。

内核如何运行新程序
--------------------------------------------------------------------------------

首先，让我们来看一下下面这个简单的 `C` 程序：

```C
// program.c

#include <stdlib.h>
#include <stdio.h>

static int x = 1;

int y = 2;

int main(int argc, char *argv[]) {
	int z = 3;

	printf("x + y + z = %d\n", x + y + z);

	return EXIT_SUCCESS;
}
```

我们可以确定这个程序按照我们预期那样工作。让我们来编译它：

```
$ gcc -Wall program.c -o sum
```

并且执行：

```
$ ./sum
x + y + z = 6
```

好的，直到现在所有事情看起来听挺好。你可能已经知道一个特殊的[系统调用](https://zh.wikipedia.org/wiki/%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8)家族 - [exec*](http://man7.org/linux/man-pages/man3/execl.3.html) 系统调用。正如我们从帮助手册中读到的：

> The exec() family of functions replaces the current process image with a new process image.

如果你已经阅读过[系统调用](https://zh.wikipedia.org/wiki/%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8)章节的[第四部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/linux-syscall-4.html)，你可能就知道 execve 这个系统调用定义在 [files/exec.c](https://github.com/torvalds/linux/blob/master/fs/exec.c#L1859) 文件中，并且如下所示，

```C
SYSCALL_DEFINE3(execve,
		const char __user *, filename,
		const char __user *const __user *, argv,
		const char __user *const __user *, envp)
{
	return do_execve(getname(filename), argv, envp);
}
```

它以可执行文件的名字，命令行参数的集合以及环境变量的集合作为参数。正如你猜测的，每一件事都是 `do_execve` 函数完成的。在这里我将不描述这个函数的实现细节，因为你可以从[这里](https://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/linux-syscall-4.html)读到。但是，简而言之，`do_execve` 函数会检查诸如文件名是否有效，未超出进程数目限制等等。在这些检查之后，这个函数会解析 [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 格式的可执行文件，为新的可执行文件创建内存描述符，并且在栈，堆等内存区域填上适当的值。当二进制镜像设置完成，`start_thread` 函数会设置一个新的进程。这个函数是框架相关的，而且对于 [x86_64](https://en.wikipedia.org/wiki/X86-64) 框架，它的定义是在 [arch/x86/kernel/process_64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/process_64.c#L231) 文件中。

`start_thread` 为[段寄存器](https://en.wikipedia.org/wiki/X86_memory_segmentation)设置新的值。从这一点开始，新进程已经准备就绪。一旦[进程切换]((https://en.wikipedia.org/wiki/Context_switch))完成，控制权就会返回到用户空间，并且新的可执行文件将会执行。

这就是所有内核方面的内容。Linux 内核为执行准备二进制镜像，而且它的执行从上下文切换开始，结束之后将控制权返回用户空间。但是它并不能回答像 `_start` 来自哪里这样的问题。让我们在下一段尝试回答这些问题。

用户空间程序如何启动
--------------------------------------------------------------------------------

在之前的段落汇总，我们看到了内核是如何为可执行文件运行做准备工作的。让我们从用户空间来看这相同的工作。我们已经知道一个程序的入口点是 `_start` 函数。但是这个函数是从哪里来的呢？它可能来自于一个库。但是如果你记得清楚的话，我们在程序编译过程中并没有链接任何库。

```
$ gcc -Wall program.c -o sum
```

你可能会猜 `_start` 来自于[标准库](https://en.wikipedia.org/wiki/Standard_library)。是的，确实是这样。如果你尝试去重新编译我们的程序，并给 gcc 传递可以开启 `verbose mode` 的 `-v` 选项，你会看到下面的长输出。我们并不对整体输出感兴趣，让我们来看一下下面的步骤：

首先，使用 `gcc` 编译我们的程序：

```
$ gcc -v -ggdb program.c -o sum
...
...
...
/usr/libexec/gcc/x86_64-redhat-linux/6.1.1/cc1 -quiet -v program.c -quiet -dumpbase program.c -mtune=generic -march=x86-64 -auxbase test -ggdb -version -o /tmp/ccvUWZkF.s
...
...
...
```

`cc1` 编译器将编译我们的 `C` 代码并且生成 `/tmp/ccvUWZkF.s` 汇编文件。之后我们可以看见我们的汇编文件被 `GNU as` 编译器编译为目标文件：


```
$ gcc -v -ggdb program.c -o sum
...
...
...
as -v --64 -o /tmp/cc79wZSU.o /tmp/ccvUWZkF.s
...
...
...
```

最后我们的目标文件会被 `collect2` 链接到一起：

```
$ gcc -v -ggdb program.c -o sum
...
...
...
/usr/libexec/gcc/x86_64-redhat-linux/6.1.1/collect2 -plugin /usr/libexec/gcc/x86_64-redhat-linux/6.1.1/liblto_plugin.so -plugin-opt=/usr/libexec/gcc/x86_64-redhat-linux/6.1.1/lto-wrapper -plugin-opt=-fresolution=/tmp/ccLEGYra.res -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lc -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s --build-id --no-add-needed --eh-frame-hdr --hash-style=gnu -m elf_x86_64 -dynamic-linker /lib64/ld-linux-x86-64.so.2 -o test /usr/lib/gcc/x86_64-redhat-linux/6.1.1/../../../../lib64/crt1.o /usr/lib/gcc/x86_64-redhat-linux/6.1.1/../../../../lib64/crti.o /usr/lib/gcc/x86_64-redhat-linux/6.1.1/crtbegin.o -L/usr/lib/gcc/x86_64-redhat-linux/6.1.1 -L/usr/lib/gcc/x86_64-redhat-linux/6.1.1/../../../../lib64 -L/lib/../lib64 -L/usr/lib/../lib64 -L. -L/usr/lib/gcc/x86_64-redhat-linux/6.1.1/../../.. /tmp/cc79wZSU.o -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed /usr/lib/gcc/x86_64-redhat-linux/6.1.1/crtend.o /usr/lib/gcc/x86_64-redhat-linux/6.1.1/../../../../lib64/crtn.o
...
...
...
```

是的，我们可以看见一个很长的命令行选项列表被传递给链接器。让我们从另一条路行进。我们知道我们的程序都依赖标准库。

```
$ ldd program
	linux-vdso.so.1 (0x00007ffc9afd2000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f56b389b000)
	/lib64/ld-linux-x86-64.so.2 (0x0000556198231000)
```

从那里我们会用一些库函数，像 `printf` 。但是不止如此。这就是为什么当我们给编译器传递 `-nostdlib` 参数，我们会收到错误报告：
```
$ gcc -nostdlib program.c -o program
/usr/bin/ld: warning: cannot find entry symbol _start; defaulting to 000000000040017c
/tmp/cc02msGW.o: In function `main':
/home/alex/program.c:11: undefined reference to `printf'
collect2: error: ld returned 1 exit status
```

除了这些错误，我们还看见 `_start` 符号未定义。所以现在我们可以确定 `_start` 函数来自于标准库。但是即使我们链接标准库，它也无法成功编译：

```
$ gcc -nostdlib -lc -ggdb program.c -o program
/usr/bin/ld: warning: cannot find entry symbol _start; defaulting to 0000000000400350
```

好的，当我们使用 `/usr/lib64/libc.so.6` 链接我们的程序，编译器并不报告标准库函数的未定义引用，但是 `_start` 符号仍然未被解析。让我们重新回到 `gcc` 的冗长输出，看看 `collect2` 的参数。我们现在最重要的问题是我们的程序不仅链接了标准库，还有一些目标文件。第一个目标文件是 `/lib64/crt1.o` 。而且，如果我们使用 `objdump` 工具去看这个目标文件的内部，我们将看见 `_start` 符号：

```
$ objdump -d /lib64/crt1.o 

/lib64/crt1.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <_start>:
   0:	31 ed                	xor    %ebp,%ebp
   2:	49 89 d1             	mov    %rdx,%r9
   5:	5e                   	pop    %rsi
   6:	48 89 e2             	mov    %rsp,%rdx
   9:	48 83 e4 f0          	and    $0xfffffffffffffff0,%rsp
   d:	50                   	push   %rax
   e:	54                   	push   %rsp
   f:	49 c7 c0 00 00 00 00 	mov    $0x0,%r8
  16:	48 c7 c1 00 00 00 00 	mov    $0x0,%rcx
  1d:	48 c7 c7 00 00 00 00 	mov    $0x0,%rdi
  24:	e8 00 00 00 00       	callq  29 <_start+0x29>
  29:	f4                   	hlt    
```

因为 `crt1.o` 是一个共享目标文件，所以我们只看到桩而不是真正的函数调用。让我们来看一下 `_start` 函数的源码。因为这个函数是框架相关的，所以 `_start` 的实现是在 [sysdeps/x86_64/start.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/start.S;h=f1b961f5ba2d6a1ebffee0005f43123c4352fbf4;hb=HEAD) 这个汇编文件中。

`_start` 始于对 `ebp` 寄存器的清零，正如 [ABI]((https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf)) 所建议的。 

```assembly
xorl %ebp, %ebp
```

之后，将终止函数的地址放到 `r9` 寄存器中：

```assembly
mov %RDX_LP, %R9_LP
```

正如 [ELF](http://flint.cs.yale.edu/cs422/doc/ELF_Format.pdf) 标准所述，

> After the dynamic linker has built the process image and performed the relocations, each shared object
> gets the opportunity to execute some initialization code.
> ...
> Similarly, shared objects may have termination functions, which are executed with the atexit (BA_OS)
> mechanism after the base process begins its termination sequence.

所以我们需要把终止函数的地址放到 `r9` 寄存器，因为将来它会被当作第六个参数传递给 `__libc_start_main` 。注意，终止函数的地址初始是存储在 `rdx` 寄存器。除了 `%rdx` 和 `%rsp` 之外的其他寄存器保存未确定的值。`_start` 函数中真正的重点是调用 `__libc_start_main`。所以下一步就是为调用这个函数做准备。

`__libc_start_main` 的实现是在 [csu/libc-start.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=csu/libc-start.c;h=0fb98f1606bab475ab5ba2d0fe08c64f83cce9df;hb=HEAD) 文件中。让我们来看一下这个函数：

```C
STATIC int LIBC_START_MAIN (int (*main) (int, char **, char **),
 			                int argc,
			                char **argv,
 			                __typeof (main) init,
			                void (*fini) (void),
			                void (*rtld_fini) (void),
			                void *stack_end)
```

It takes address of the `main` function of a program, `argc` and `argv`. `init` and `fini` functions are constructor and destructor of the program. The `rtld_fini` is termination function which will be called after the program will be exited to terminate and free dynamic section. The last parameter of the `__libc_start_main` is the pointer to the stack of the program. Before we can call the `__libc_start_main` function, all of these parameters must be prepared and passed to it. Let's return to the [sysdeps/x86_64/start.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/start.S;h=f1b961f5ba2d6a1ebffee0005f43123c4352fbf4;hb=HEAD) assembly file and continue to see what happens before the `__libc_start_main` function will be called from there.

该函数以程序 `main` 函数的地址，`argc` 和 `argv` 作为输入。`init` 和 `fini` 函数分别是程序的构造函数和析构函数。`rtld_fini` 是当程序退出时调用的终止函数，用来终止以及释放动态段。`__libc_start_main` 函数的最后一个参数是一个指向程序栈的指针。在我们调用 `__libc_start_main` 函数之前，所有的参数都要被准备好，并且传递给它。让我们返回 [sysdeps/x86_64/start.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/start.S;h=f1b961f5ba2d6a1ebffee0005f43123c4352fbf4;hb=HEAD) 这个文件，继续看在 `__libc_start_main` 被调用之前发生了什么。

我们可以从栈上获取我们所需的 `__libc_start_main` 的所有参数。当 `_start` 被调用的时候，我们的栈如下所示：

```
+-----------------+
|       NULL      |
+-----------------+ 
|       envp      |
+-----------------+ 
|       NULL      |
+------------------
|       argv      | <- rsp
+------------------
|       argc      |
+-----------------+ 
```

当我们清零了 `ebp` 寄存器，并且将终止函数的地址保存到 `r9` 寄存器中之后，我们取出栈顶元素，放到 `rsi` 寄存器中。最终 `rsp` 指向 `argv` 数组，`rsi` 保存传递给程序的命令行参数的数目：
```
+-----------------+
|       NULL      |
+-----------------+ 
|       envp      |
+-----------------+ 
|       NULL      |
+------------------
|       argv      | <- rsp
+-----------------+
```

这之后，我们将 `argv` 数组的地址赋值给 `rdx` 寄存器中。

```assembly
popq %rsi
mov %RSP_LP, %RDX_LP
```

从这一时刻开始，我们已经有了 `argc` 和 `argv`。我们仍要将构造函数和析构函数的指针放到合适的寄存器，以及传递指向栈的指针。下面汇编代码的前三行按照 [ABI](https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf) 中的建议设置栈为 `16` 字节对齐，并将 `rax` 压栈：

```assembly
and  $~15, %RSP_LP
pushq %rax

pushq %rsp
mov $__libc_csu_fini, %R8_LP
mov $__libc_csu_init, %RCX_LP
mov $main, %RDI_LP
```

栈对齐之后，我们压栈栈的地址，并且将构造函数和析构函数的地址放到 `r8` 和 `rcx` 寄存器中，同时将 `main` 函数的地址放到 `rdi` 寄存器中。从这个时刻开始，我们可以调用 [csu/libc-start.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=csu/libc-start.c;h=0fb98f1606bab475ab5ba2d0fe08c64f83cce9df;hb=HEAD) 中的 `__libc_start_main` 函数。

在我们查看 `__libc_start_main` 函数之前，让我们添加 `/lib64/crt1.o` 文件并且再次尝试编译我们的程序：

```
$ gcc -nostdlib /lib64/crt1.o -lc -ggdb program.c -o program
/lib64/crt1.o: In function `_start':
(.text+0x12): undefined reference to `__libc_csu_fini'
/lib64/crt1.o: In function `_start':
(.text+0x19): undefined reference to `__libc_csu_init'
collect2: error: ld returned 1 exit status
```

现在我们看见了另外一个错误 - 未找到 `__libc_csu_fini` 和 `__libc_csu_init` 。我们知道这两个函数的地址被传递给 `__libc_start_main` 作为参数，同时这两个函数还是我们程序的构造函数和析构函数。但是在 `C` 程序中，构造函数和析构函数意味着什么呢？我们已经在 [ELF](http://flint.cs.yale.edu/cs422/doc/ELF_Format.pdf) 标准中看到：

> After the dynamic linker has built the process image and performed the relocations, each shared object
> gets the opportunity to execute some initialization code.
> ...
> Similarly, shared objects may have termination functions, which are executed with the atexit (BA_OS)
> mechanism after the base process begins its termination sequence.

所以链接器除了一般的段，如 `.text`, `.data` 之外创建了两个特殊的段：

* `.init`
* `.fini`

We can find it with `readelf` util:

我们可以通过 `readelf` 工具找到它们：

```
$ readelf -e test | grep init
  [11] .init             PROGBITS         00000000004003c8  000003c8

$ readelf -e test | grep fini
  [15] .fini             PROGBITS         0000000000400504  00000504
```

这两个将被替换为二进制镜像的开始和结尾，包含分别被称为构造函数和析构函数的例程。这些例程的要点是在程序的真正代码执行之前，做一些初始化/终结，像全局变量如 [errno](http://man7.org/linux/man-pages/man3/errno.3.html) ，为系统例程分配和释放内存等等。

你可能可以从这些函数的名字推测，这两个会在 `main` 函数之前和之后被调用。`.init` 和 `.fini` 段的定义在 `/lib64/crti.o` 中。如果我们添加这个目标文件：

```
$ gcc -nostdlib /lib64/crt1.o /lib64/crti.o  -lc -ggdb program.c -o program
```

我们不会收到任何错误报告。但是让我们尝试去运行我们的程序，看看发生什么：

```
$ ./program
Segmentation fault (core dumped)
```

是的，我们收到 `segmentation fault` 。让我们通过 `objdump` 看看 `lib64/crti.o` 的内容：

```
$ objdump -D /lib64/crti.o

/lib64/crti.o:     file format elf64-x86-64


Disassembly of section .init:

0000000000000000 <_init>:
   0:	48 83 ec 08          	sub    $0x8,%rsp
   4:	48 8b 05 00 00 00 00 	mov    0x0(%rip),%rax        # b <_init+0xb>
   b:	48 85 c0             	test   %rax,%rax
   e:	74 05                	je     15 <_init+0x15>
  10:	e8 00 00 00 00       	callq  15 <_init+0x15>

Disassembly of section .fini:

0000000000000000 <_fini>:
   0:	48 83 ec 08          	sub    $0x8,%rsp
```

正如上面所写的， `/lib64/crti.o` 目标文件包含 `.init` 和 `.fini` 段的定义，但是我们可以看见这个函数的桩。让我们看一下 [sysdeps/x86_64/crti.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/crti.S;h=e9d86ed08ab134a540e3dae5f97a9afb82cdb993;hb=HEAD) 文件中的源码：

```assembly
	.section .init,"ax",@progbits
	.p2align 2
	.globl _init
	.type _init, @function
_init:
	subq $8, %rsp
	movq PREINIT_FUNCTION@GOTPCREL(%rip), %rax
	testq %rax, %rax
	je .Lno_weak_fn
	call *%rax
.Lno_weak_fn:
	call PREINIT_FUNCTION
```

它包含 `.init` 段的定义，而且汇编代码设置 16 字节的对齐。之后，如果它不是零，我们调用 `PREINIT_FUNCTION`；否则不调用：

```
00000000004003c8 <_init>:
  4003c8:       48 83 ec 08             sub    $0x8,%rsp
  4003cc:       48 8b 05 25 0c 20 00    mov    0x200c25(%rip),%rax        # 600ff8 <_DYNAMIC+0x1d0>
  4003d3:       48 85 c0                test   %rax,%rax
  4003d6:       74 05                   je     4003dd <_init+0x15>
  4003d8:       e8 43 00 00 00          callq  400420 <__libc_start_main@plt+0x10>
  4003dd:       48 83 c4 08             add    $0x8,%rsp
  4003e1:       c3                      retq
```

where the `PREINIT_FUNCTION` is the `__gmon_start__` which does setup for profiling. You may note that we have no return instruction in the [sysdeps/x86_64/crti.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/crti.S;h=e9d86ed08ab134a540e3dae5f97a9afb82cdb993;hb=HEAD). Actually that's why we got segmentation fault. Prolog of `_init` and `_fini` is placed in the [sysdeps/x86_64/crtn.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/crtn.S;h=e9d86ed08ab134a540e3dae5f97a9afb82cdb993;hb=HEAD) assembly file:

其中，`PREINIT_FUNCTION` 是设置简况的 `__gmon_start__`。你可能发现，在 [sysdeps/x86_64/crti.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/crti.S;h=e9d86ed08ab134a540e3dae5f97a9afb82cdb993;hb=HEAD)中，我们没有 `return` 指令。事实上，这就是我们获得 `segmentation fault` 的原因。`_init` 和 `_fini` 的序言被放在 [sysdeps/x86_64/crtn.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/crtn.S;h=e9d86ed08ab134a540e3dae5f97a9afb82cdb993;hb=HEAD) 汇编文件中：

```assembly
.section .init,"ax",@progbits
addq $8, %rsp
ret

.section .fini,"ax",@progbits
addq $8, %rsp
ret
```

如果我们把它加到编译过程中，我们的程序会被成功编译和运行。

```
$ gcc -nostdlib /lib64/crt1.o /lib64/crti.o /lib64/crtn.o  -lc -ggdb program.c -o program

$ ./program
x + y + z = 6
```

结论
--------------------------------------------------------------------------------

现在让我们回到 `_start` 函数，以及尝试去浏览 `main` 函数被调用之前的完整调用链。

`_start` 总是被默认的 `ld` 脚本链接到程序 `.text` 段的起始位置：

```
$ ld --verbose | grep ENTRY
ENTRY(_start)
```

`_start` 函数定义在 [sysdeps/x86_64/start.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/start.S;h=f1b961f5ba2d6a1ebffee0005f43123c4352fbf4;hb=HEAD) 汇编文件中，并且在 `__libc_start_main` 被调用之前做一些准备工作，像从栈上获取 `argc/argv`，栈准备等。来自于 [csu/libc-start.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=csu/libc-start.c;h=0fb98f1606bab475ab5ba2d0fe08c64f83cce9df;hb=HEAD) 文件中的 `__libc_start_main` 函数注册构造函数和析构函数，开启线程，做一些安全相关的操作，比如在有需要的情况下设置 `stack canary`，调用初始化，最后调用程序的 `main` 函数以及返回结果退出。而构造函数和析构函数分别是 `main` 之前和之后被调用。

```C
result = main (argc, argv, __environ MAIN_AUXVEC_PARAM);
exit (result);
```

结束

链接
--------------------------------------------------------------------------------

* [system call](https://en.wikipedia.org/wiki/System_call)
* [gdb](https://www.gnu.org/software/gdb/)
* [execve](http://linux.die.net/man/2/execve)
* [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [segment registers](https://en.wikipedia.org/wiki/X86_memory_segmentation)
* [context switch](https://en.wikipedia.org/wiki/Context_switch)
* [System V ABI](https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf)
