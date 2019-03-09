Linux 内核系统调用 第一节 
================================================================================

简介
--------------------------------------------------------------------------------

这次提交为 [linux内核解密](http://xinqiu.gitbooks.io/linux-insides-cn/content/) 添加一个新的章节，从标题就可以知道, 这一章节将介绍Linux 内核中 [System Call](https://en.wikipedia.org/wiki/System_call) 的概念。章节内容的选择并非偶然。在前一[章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/index.html)我们了解了中断及中断处理。系统调用的概念与中断非常相似，这是因为软件中断是执行系统调用最常见的方式。接下来我们将从不同的角度来审视系统调用相关概念。例如，从用户空间发起系统调用时会发生什么，Linux内核中一组系统调用处理器的实现，[VDSO](https://en.wikipedia.org/wiki/VDSO) 和 [vsyscall](https://lwn.net/Articles/446528/) 的概念以及其他信息。

在了解 Linux 内核系统调用执行过程之前，让我们先来了解一些系统调用的相关原理。

什么是系统调用?
--------------------------------------------------------------------------------

系统调用就是从用户空间发起的内核服务请求。操作系统内核其实会提供很多服务，比如当程序想要读写文件、监听某个[socket](https://en.wikipedia.org/wiki/Network_socket)端口、删除或创建目录或者程序结束时，都会执行系统调用。换句话说，系统调用其实就是一些由用户空间程序调用去处理某些请求的 [C] (https://en.wikipedia.org/wiki/C_%28programming_language%29) 内核空间函数。

Linux 内核提供一系列的函数，但这些函数与CPU架构相关。 例如：[x86_64](https://en.wikipedia.org/wiki/X86-64) 提供 [322](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl) 个系统调用，[x86](https://en.wikipedia.org/wiki/X86) 提供 [358](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_32.tbl) 个不同的系统调用。
系统调用仅仅是一些函数。 我们看一个使用汇编语言编写的简单 `Hello world` 示例:

```assembly
.data

msg:
    .ascii "Hello, world!\n"
    len = . - msg

.text
    .global _start

_start:
    movq  $1, %rax
    movq  $1, %rdi
    movq  $msg, %rsi
    movq  $len, %rdx
    syscall

    movq  $60, %rax
    xorq  %rdi, %rdi
    syscall
```

使用下面的命令可编译这些语句:

```
$ gcc -c test.S
$ ld -o test test.o
```

执行:

```
./test
Hello, world!
```

这些代码是 Linux `x86_64` 架构下 `Hello world` 简单的汇编程序，代码包含两段:

* `.data`
* `.text`

第一段 - `.data` 存储程序的初始数据 (在示例中为`Hello world` 字符串)，第二段 - `.text` 包含程序的代码。代码可分为两部分: 第一部分为第一个 `syscall` 指令之前的代码，第二部分为两个 `syscall` 指令之间的代码。在示例程序及一般应用中， `syscall` 指令有什么功能？[64-ia-32-architectures-software-developer-vol-2b-manual](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)中提到:

######

```
SYSCALL 可以以优先级0调起系统调用处理程序，它通过加载IA32_LSTAR MSR至RIP完成调用(在RCX中保存 SYSCALL 指令地址之后)。
(WRMSR 指令确保IA32_LSTAR MSR总是包含一个连续的地址。)
...
...
...
SYSCALL 将 IA32_STAR MSR 的 47：32 位加载至 CS 和 SS 段选择器。总之，CS 和 SS 描述符缓存不是从哪些选择器所引用的描述符(在 GDT 或者 LDT 中)加载的。

相反，描述符缓存用固定值加载。确保由段选择器得到的描述符与从固定值加载至描述符缓存的描述符保持一致是操作系统的本职工作，但 SYSCALL 指令不保证两者的一致。
```

使用[arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/entry_64.S)汇编程序中定义的 `entry_SYSCALL_64` 初始化 `syscalls`
同时 `SYSCALL` 指令进入[arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/cpu/common.c) 源码文件中的 `IA32_STAR` [Model specific register](https://en.wikipedia.org/wiki/Model-specific_register):

```C
wrmsrl(MSR_LSTAR, entry_SYSCALL_64);
```

因此，`syscall` 指令唤醒一个系统调用对应的处理程序。但是如何确定调用哪个处理程序？事实上这些信息从通用目的[寄存器](https://en.wikipedia.org/wiki/Processor_register)得到。正如[系统调用表](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl)中描述，每个系统调用对应特定的编号。上面的示例中, 第一个系统调用是 - `write` 将数据写入指定文件。在系统调用表中查找 write 系统调用.[write](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L10) 系统调用的编号为 - `1`。在示例中通过`rax`寄存器传递该编号，接下来的几个通用目的寄存器: `%rdi`, `%rsi` 和 `%rdx` 分别保存 `write` 系统调用的三个参数。 在示例中它们分别是：

* [文件描述符](https://en.wikipedia.org/wiki/File_descriptor) (`1` 是[stdout](https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29))    
*  参数字符串指针 　　
* 数据的大小 　　

是的，你没有看错，这就是系统调用的参数。正如上文所示, 系统调用仅仅是内核空间的 `C` 函数。示例中第一个系统调用为 write ，在 [fs/read_write.c] (https://github.com/torvalds/linux/blob/master/fs/read_write.c) 源文件中定义如下:

```C
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	...
	...
	...
}
```

或者是:

```C
ssize_t write(int fd, const void *buf, size_t nbytes);
```

暂时不用担心宏 `SYSCALL_DEFINE3` ,稍后再做讨论。

示例的第二部分也是一样的, 但调用了另一系统调用[exit](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L69)。这个系统调用仅需一个参数:

* Return value

该参数说明程序退出的方式。[strace](https://en.wikipedia.org/wiki/Strace) 工具可根据程序的名称输出系统调用的过程:

```
$ strace test
execve("./test", ["./test"], [/* 62 vars */]) = 0
write(1, "Hello, world!\n", 14Hello, world!
)         = 14
_exit(0)                                = ?

+++ exited with 0 +++
```

 `strace` 输出的第一行, [execve](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L68) 系统调用来执行程序，第二、三行为程序中使用的系统调用: `write` 和 `exit`。注意示例中通过通用目的寄存器传递系统调用的参数。寄存器的顺序是指定的，该顺序在- [x86-64 calling conventions] (https://en.wikipedia.org/wiki/X86_calling_conventions#x86-64_calling_conventions)中定义。
 `x86_64` 架构的声明在另一个特别的文档中 - [System V Application Binary Interface. PDF](http://www.x86-64.org/documentation/abi.pdf)。通常，函数参数被置于寄存器或者堆栈中。正确的顺序为:

* `rdi` 
* `rsi` 
* `rdx` 
* `rcx` 
* `r8` 
* `r9` 

这六个寄存器分别对应函数的前六个参数。若函数多于六个参数，其他参数将被放在堆栈中。 

我们不会在代码中直接使用系统调用，但当我们想打印一些东西的时候肯定会用到，检测一个文件的权限或是读写数据都会用到系统调用。

例如:

```C
#include <stdio.h>

int main(int argc, char **argv)
{
   FILE *fp;
   char buff[255];

   fp = fopen("test.txt", "r");
   fgets(buff, 255, fp);
   printf("%s\n", buff);
   fclose(fp);

   return 0;
}
```

Linux内核中没有 `fopen`, `fgets`, `printf` 和 `fclose` 系统调用，而是 `open`, `read` `write` 和 `close`。`fopen`, `fgets`, `printf` 和 `fclose` 仅仅是 `C` [standard library](https://en.wikipedia.org/wiki/GNU_C_Library)中定义的函数。事实上这些函数是系统调用的封装。我们不会在代码中直接使用系统调用，而是通过标准库的[封装](https://en.wikipedia.org/wiki/Wrapper_function)函数。主要原因非常简单: 系统调用执行的要快，非常快。系统调用快的同时也要非常小。而标准库会在执行系统调用前，确保系统调用参数设置正确并且完成一些其他不同的检查。我们用以下命令编译下示例程序：

```
$ gcc test.c -o test
```

通过[ltrace](https://en.wikipedia.org/wiki/Ltrace)工具观察:

```
$ ltrace ./test
__libc_start_main([ "./test" ] <unfinished ...>
fopen("test.txt", "r")                                             = 0x602010
fgets("Hello World!\n", 255, 0x602010)                             = 0x7ffd2745e700
puts("Hello World!\n"Hello World!

)                                                                  = 14
fclose(0x602010)                                                   = 0
+++ exited (status 0) +++
```

`ltrace`工具显示出了程序在用户空间的调用。 `fopen` 函数打开给定的文本文件,  `fgets` 函数读取文件内容至 `buf` 缓存，`puts` 输出文件内容至 `stdout` ， `fclose` 函数根据文件描述符关闭函数。如上文描述，这些函数调用特定的系统调用。例如： `puts` 内部调用 `write` 系统调用，`ltrace` 添加 `-S`可观察到这一调用:

```
write@SYS(1, "Hello World!\n\n", 14) = 14
```

系统调用是普遍存在的。每个程序都需要打开/写/读文件，网络连接，内存分配和许多其他功能，这些只能由内核提供。[proc](https://en.wikipedia.org/wiki/Procfs) 文件系统有一个具有特定格式的特殊文件: `/proc/${pid}/syscall`记录了正在被进程调用的系统调用的编号和参数寄存器。例如,进程号 1 的程序是[systemd](https://en.wikipedia.org/wiki/Systemd):

```
$ sudo cat /proc/1/comm
systemd

$ sudo cat /proc/1/syscall
232 0x4 0x7ffdf82e11b0 0x1f 0xffffffff 0x100 0x7ffdf82e11bf 0x7ffdf82e11a0 0x7f9114681193
```

编号为 `232` 的系统调用为 [epoll_wait](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L241)，该调用等待 [epoll](https://en.wikipedia.org/wiki/Epoll) 文件描述符的I/O事件. 例如我用来编写这一节的 `emacs` 编辑器:

```
$ ps ax | grep emacs
2093 ?        Sl     2:40 emacs

$ sudo cat /proc/2093/comm
emacs

$ sudo cat /proc/2093/syscall
270 0xf 0x7fff068a5a90 0x7fff068a5b10 0x0 0x7fff068a59c0 0x7fff068a59d0 0x7fff068a59b0 0x7f777dd8813c
```

编号为 `270` 的系统调用是 [sys_pselect6](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L279) ，该系统调用使 `emacs` 监控多个文件描述符。

现在我们对系统调用有所了解，知道什么是系统调用及为什么需要系统调用。接下来，讨论示例程序中使用的 `write` 系统调用

write系统调用的实现
--------------------------------------------------------------------------------

查看Linux内核源文件中写系统调用的实现。[fs/read_write.c](https://github.com/torvalds/linux/blob/master/fs/read_write.c) 源码文件中的 `write` 系统调用定义如下：
```C
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos = file_pos_read(f.file);
		ret = vfs_write(f.file, buf, count, &pos);
		if (ret >= 0)
			file_pos_write(f.file, pos);
		fdput_pos(f);
	}

	return ret;
}
```

首先，宏 `SYSCALL_DEFINE3` 在头文件 [include/linux/syscalls.h](https://github.com/torvalds/linux/blob/master/include/linux/syscalls.h) 中定义并且作为 `sys_name(...)` 函数定义的扩展。该宏的定义如下:

```C
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)

#define SYSCALL_DEFINEx(x, sname, ...)                \
        SYSCALL_METADATA(sname, x, __VA_ARGS__)       \
        __SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
```

宏 `SYSCALL_DEFINE3` 的参数有代表系统调用的名称的 `name`  和可变个数的参数。 这个宏仅仅为 `SYSCALL_DEFINEx` 宏的扩展确定了传入宏的参数个数。 `_##name` 作为未来系统调用名称的存根 (更多关于 `##`符号连结可参阅[documentation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html) of [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection))。让我们来看看 `SYSCALL_DEFINEx` 这个宏，这个宏扩展为以下两个宏:

* `SYSCALL_METADATA`;
* `__SYSCALL_DEFINEx`.

第一个宏 `SYSCALL_METADATA` 的实现依赖于`CONFIG_FTRACE_SYSCALLS`内核配置选项。 从选项的名称可知，它允许 tracer 捕获系统调用的进入和退出。若该内核配置选项开启，宏 `SYSCALL_METADATA`  执行头文件[include/trace/syscall.h](https://github.com/torvalds/linux/blob/master/include/trace/syscall.h) 中`syscall_metadata` 结构的初始化，该结构中包含多种有用字段例如系统调用的名称, 系统调用[表](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl)中的编号、参数个数、参数类型列表等:

```C
#define SYSCALL_METADATA(sname, nb, ...)                             \
	...                                                              \
	...                                                              \
	...                                                              \
    struct syscall_metadata __used                                   \
              __syscall_meta_##sname = {                             \
                    .name           = "sys"#sname,                   \
                    .syscall_nr     = -1,                            \
                    .nb_args        = nb,                            \
                    .types          = nb ? types_##sname : NULL,     \
                    .args           = nb ? args_##sname : NULL,      \
                    .enter_event    = &event_enter_##sname,          \
                    .exit_event     = &event_exit_##sname,           \
                    .enter_fields   = LIST_HEAD_INIT(__syscall_meta_##sname.enter_fields), \
             };                                                                            \

    static struct syscall_metadata __used                           \
              __attribute__((section("__syscalls_metadata")))       \
             *__p_syscall_meta_##sname = &__syscall_meta_##sname;
```

若内核配置时 `CONFIG_FTRACE_SYSCALLS` 未开启，此时宏 `SYSCALL_METADATA`扩展为空字符串:

```C
#define SYSCALL_METADATA(sname, nb, ...)
```

第二个宏 `__SYSCALL_DEFINEx` 扩展为以下五个函数的定义:

```C
#define __SYSCALL_DEFINEx(x, name, ...)                                 \
        asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))       \
                __attribute__((alias(__stringify(SyS##name))));         \
                                                                        \
        static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__));  \
                                                                        \
        asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__));      \
                                                                        \
        asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__))       \
        {                                                               \
                long ret = SYSC##name(__MAP(x,__SC_CAST,__VA_ARGS__));  \
                __MAP(x,__SC_TEST,__VA_ARGS__);                         \
                __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));       \
                return ret;                                             \
        }                                                               \
                                                                        \
        static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```

第一个函数 `sys##name` 是给定名称为 `sys_system_call_name` 的系统调用处理器函数的定义。 宏 `__SC_DECL` 的参数有 `__VA_ARGS__` 及组合调用传入参数系统类型和参数名称，因为宏定义中无法指定参数类型。宏 `__MAP` 应用宏 `__SC_DECL` 给 `__VA_ARGS__` 参数。其他的函数是 `__SYSCALL_DEFINEx`生成的，详细信息可以查阅[CVE-2009-0029](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-0029) 此处不再深究。总之，write的系统调用函数定义应该是长这样:

```C
asmlinkage long sys_write(unsigned int fd, const char __user * buf, size_t count);
```

现在我们对系统调用的定义有一定了解，再来回头看看 `write` 系统调用的实现:

```C
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos = file_pos_read(f.file);
		ret = vfs_write(f.file, buf, count, &pos);
		if (ret >= 0)
			file_pos_write(f.file, pos);
		fdput_pos(f);
	}

	return ret;
}
```

从代码可知，该调用有三个参数:

* `fd` - 文件描述符
* `buf` - 写缓冲区
* `count` - 写缓冲区大小

该调用的功能是将用户定义的缓冲中的数据写入指定的设备或文件。注意第二个参数 `buf`, 定义了 `__user` 属性。该属性的主要目的是通过 [sparse](https://en.wikipedia.org/wiki/Sparse) 工具检查 Linux 内核代码。sparse 定义于 [include/linux/compiler.h] (https://github.com/torvalds/linux/blob/master/include/linux/compiler.h) 头文件中，并依赖 Linux 内核中 `__CHECKER__` 的定义。以上全是关于 `sys_write` 系统调用的有用元信息。我们可以看到，它的实现开始于 `f` 结构的定义，`f` 结构包含 `fd` 结构类型，`fd`是 Linux 内核中的文件描述符，也是我们存放 `fdget_pos` 函数调用结果的地方。`fdget_pos` 函数在相同的[源文件](https://github.com/torvalds/linux/blob/master/fs/read_write.c)中定义，其实就是 `__to_fd` 函数的扩展:

```C
static inline struct fd fdget_pos(int fd)
{
        return __to_fd(__fdget_pos(fd));
}
```

 `fdget_pos` 的主要目的是将给定的只有数字的文件描述符转化为 `fd` 结构。 通过一长链的函数调用， `fdget_pos` 函数得到当前进程的文件描述符表, `current->files`, 并尝试从表中获取一致的文件描述符编号。当获取到给定文件描述符的 `fd` 结构后, 检查文件并返回文件是否存在。通过调用函数 `file_pos_read` 获取当前处于文件中的位置。函数返回文件的 `f_pos` 字段:

```C
static inline loff_t file_pos_read(struct file *file)
{
        return file->f_pos;
}
```

接下来再调用 `vfs_write` 函数。 `vfs_write` 函数在源码文件 [fs/read_write.c](https://github.com/torvalds/linux/blob/master/fs/read_write.c) 中定义。其功能为 - 向指定文件的指定位置写入指定缓冲中的数据。此处不深入 `vfs_write` 函数的细节，因为这个函数与`系统调用`没有太多联系，反而与另一章节[虚拟文件系统](https://en.wikipedia.org/wiki/Virtual_file_system)相关。`vfs_write` 结束相关工作后, 检查结果若成功执行，使用`file_pos_write` 函数改变在文件中的位置:

```C
if (ret >= 0)
	file_pos_write(f.file, pos);
```

这恰好使用给定的位置更新给定文件的 `f_pos`:

```C
static inline void file_pos_write(struct file *file, loff_t pos)
{
        file->f_pos = pos;
}
```

在 `write` 系统调用处理函数的结尾处, 我们可以看到以下函数调用:

```C
fdput_pos(f);
```

这是在解锁在共享文件描述符的线程并发写文件时保护文件位置的互斥量 `f_pos_lock`。

我们讨论了Linux内核提供的系统调用的部分实现。显然略过了 `write` 系统调用实现的部分内容，正如文中所述, 在该章节中仅关心系统调用的相关内容，不讨论与其他子系统相关的内容，例如[虚拟文件系统](https://en.wikipedia.org/wiki/Virtual_file_system).

总结
--------------------------------------------------------------------------------
  
第一部分介绍了Linux内核中的系统调用概念。到目前为止，我们已经介绍了系统调用的理论，在下一部分中，我们将继续深入这个主题，讨论与系统调用相关的Linux内核代码。  

若存在疑问及建议, 在twitter @[0xAX](https://twitter.com/0xAX), 通过[email](anotherworldofworld@gmail.com) 或者创建 [issue](https://github.com/MintCN/linux-insides-zh/issues/new).

**由于英语是我的第一语言由此造成的不便深感抱歉。若发现错误请提交 PR 至 [linux-insides](https://github.com/MintCN/linux-insides-zh).**

链接
--------------------------------------------------------------------------------

* [system call](https://en.wikipedia.org/wiki/System_call)
* [vdso](https://en.wikipedia.org/wiki/VDSO)
* [vsyscall](https://lwn.net/Articles/446528/)
* [general purpose registers](https://en.wikipedia.org/wiki/Processor_register)
* [socket](https://en.wikipedia.org/wiki/Network_socket)
* [C programming language](https://en.wikipedia.org/wiki/C_%28programming_language%29)
* [x86](https://en.wikipedia.org/wiki/X86)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [x86-64 calling conventions](https://en.wikipedia.org/wiki/X86_calling_conventions#x86-64_calling_conventions)
* [System V Application Binary Interface. PDF](http://www.x86-64.org/documentation/abi.pdf)
* [GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [Intel manual. PDF](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [system call table](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl)
* [GCC macro documentation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html)
* [file descriptor](https://en.wikipedia.org/wiki/File_descriptor)
* [stdout](https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29)
* [strace](https://en.wikipedia.org/wiki/Strace)
* [standard library](https://en.wikipedia.org/wiki/GNU_C_Library)
* [wrapper functions](https://en.wikipedia.org/wiki/Wrapper_function)
* [ltrace](https://en.wikipedia.org/wiki/Ltrace)
* [sparse](https://en.wikipedia.org/wiki/Sparse)
* [proc file system](https://en.wikipedia.org/wiki/Procfs)
* [Virtual file system](https://en.wikipedia.org/wiki/Virtual_file_system)
* [systemd](https://en.wikipedia.org/wiki/Systemd)
* [epoll](https://en.wikipedia.org/wiki/Epoll)
* [Previous chapter](http://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/index.html)
