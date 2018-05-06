Linux 内核系统调用 第三节
================================================================================

vsyscalls 和 vDSO
--------------------------------------------------------------------------------

这是讲解 Linux 内核中系统调用[章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/index.html)的第三部分，[前一节](http://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/linux-syscall-2.html)讨论了用户空间应用程序发起的系统调用的准备工作及系统调用的处理过程。在这一节将讨论两个与系统调用十分相似的概念，这两个概念是`vsyscall` 和 `vdso`。

我们已经了解什么是`系统调用`。这是 Linux 内核一种特殊的运行机制，使得用户空间的应用程序可以请求，像写入文件和打开套接字等特权级下的任务。正如你所了解的，在 Linux 内核中发起一个系统调用是特别昂贵的操作，因为处理器需要中断当前正在执行的任务，切换内核模式的上下文，在系统调用处理完毕后跳转至用户空间。以下的两种机制  - `vsyscall` 和d `vdso` 被设计用来加速系统调用的处理，在这一节我们将了解两种机制的工作原理。

 vsyscalls 介绍
--------------------------------------------------------------------------------

 `vsyscall` 或 `virtual system call` 是第一种也是最古老的一种用于加快系统调用的机制。 `vsyscall` 的工作原则其实十分简单。Linux 内核在用户空间映射一个包含一些变量及一些系统调用的实现的内存页。 对于 [X86_64](https://en.wikipedia.org/wiki/X86-64) 架构可以在 Linux 内核的 [文档] (https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/mm.txt) 找到关于这一内存区域的信息：

```
ffffffffff600000 - ffffffffffdfffff (=8 MB) vsyscalls
```

或:

```
~$ sudo cat /proc/1/maps | grep vsyscall
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```

因此, 这些系统调用将在用户空间下执行，这意味着将不发生 [上下文切换](https://en.wikipedia.org/wiki/Context_switch)。 `vsyscall` 内存页的映射在 [arch/x86/entry/vsyscall/vsyscall_64.c](https://github.com/torvalds/linux/blob/master/arch/x86/entry/vsyscall/vsyscall_64.c) 源代码中定义的 `map_vsyscall` 函数中实现。这一函数在 Linux 内核初始化时被 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c) 源代码中定义的函数`setup_arch` (我们在[第五章](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-5.html) Linux 内核的初始化中讨论过该函数)。

注意 `map_vsyscall` 函数的实现依赖于内核配置选项 `CONFIG_X86_VSYSCALL_EMULATION` :

```C
#ifdef CONFIG_X86_VSYSCALL_EMULATION
extern void map_vsyscall(void);
#else
static inline void map_vsyscall(void) {}
#endif
```

正如帮助文档中所描述的,  `CONFIG_X86_VSYSCALL_EMULATION` 配置选项: `使能 vsyscall 模拟`. 为何模拟 `vsyscall`? 事实上,  `vsyscall` 由于安全原因是一种遗留 [ABI](https://en.wikipedia.org/wiki/Application_binary_interface) 。虚拟系统调用具有绑定的地址, 意味着 `vsyscall` 的内存页的位置在任何时刻是相同，这一位置是在  `map_vsyscall` 函数中指定的。这一函数的实现如下: 

```C
void __init map_vsyscall(void)
{
    extern char __vsyscall_page;
    unsigned long physaddr_vsyscall = __pa_symbol(&__vsyscall_page);
	...
	...
	...
}
```

在 `map_vsyscall` 函数的开始，通过宏 `__pa_symbol` 获取了 `vsyscall` 内存页的物理地址(我们已在[第四章](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-4.html) of the Linux kernel initialization process)讨论了该宏的实现）。`__vsyscall_page` 在  [arch/x86/entry/vsyscall/vsyscall_emu_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/vsyscall/vsyscall_emu_64.S) 汇编源代码文件中定义， 具有如下的 [虚拟地址](https://en.wikipedia.org/wiki/Virtual_address_space):

```
ffffffff81881000 D __vsyscall_page
```

在 `.data..page_aligned, aw` [段](https://en.wikipedia.org/wiki/Memory_segmentation) 中 包含如下三中系统调用: 

* `gettimeofday`;
* `time`;
* `getcpu`.

或:

```assembly
__vsyscall_page:
	mov $__NR_gettimeofday, %rax
	syscall
	ret

	.balign 1024, 0xcc
	mov $__NR_time, %rax
	syscall
	ret

	.balign 1024, 0xcc
	mov $__NR_getcpu, %rax
	syscall
	ret
```

回到 `map_vsyscall` 函数及 `__vsyscall_page` 的实现，在得到 `__vsyscall_page` 的物理地址之后，使用 `__set_fixmap` 为  `vsyscall` 内存页 检查设置 [fix-mapped](http://xinqiu.gitbooks.io/linux-insides-cn/content/MM/linux-mm-2.html)地址的变量`vsyscall_mode`:

```C
if (vsyscall_mode != NONE)
	__set_fixmap(VSYSCALL_PAGE, physaddr_vsyscall,
                 vsyscall_mode == NATIVE
                             ? PAGE_KERNEL_VSYSCALL
                             : PAGE_KERNEL_VVAR);
```

The `__set_fixmap` takes three arguments: The first is index of the `fixed_addresses` [enum](https://en.wikipedia.org/wiki/Enumerated_type). In our case `VSYSCALL_PAGE` is the first element of the `fixed_addresses` enum for the `x86_64` architecture:

```C
enum fixed_addresses {
...
...
...
#ifdef CONFIG_X86_VSYSCALL_EMULATION
	VSYSCALL_PAGE = (FIXADDR_TOP - VSYSCALL_ADDR) >> PAGE_SHIFT,
#endif
...
...
...
```

该变量值为 `511`。第二个参数为映射内存页的物理地址，第三个参数为内存页的标志位。注意 `VSYSCALL_PAGE` 标志位依赖于变量 `vsyscall_mode` 。当 `vsyscall_mode` 变量为 `NATIVE` 时，  标志位为 `PAGE_KERNEL_VSYSCALL`，其他情况则是`PAGE_KERNEL_VVAR` 。两个宏 ( `PAGE_KERNEL_VSYSCALL` 及 `PAGE_KERNEL_VVAR`) 都将被扩展以下标志:

```C
#define __PAGE_KERNEL_VSYSCALL          (__PAGE_KERNEL_RX | _PAGE_USER)
#define __PAGE_KERNEL_VVAR              (__PAGE_KERNEL_RO | _PAGE_USER)
```

标志反映了 `vsyscall` 内存页的访问权限。两个标志都带有 `_PAGE_USER` 标志， 这意味着内存页可被运行于低特权级的用户模式进程访问。第二个标志位取决于 `vsyscall_mode` 变量的值。第一个标志  (`__PAGE_KERNEL_VSYSCALL`) 在 `vsyscall_mode` 为 `NATIVE` 时被设定。这意味着虚拟系统调用将以本地 `syscall` 指令的方式执行。另一情况下，在 `vsyscall_mode` 为 `emulate` 时 vsyscall  为 `PAGE_KERNEL_VVAR`，此时系统调用将被置于陷阱并被合理的模拟。  `vsyscall_mode` 变量通过 `vsyscall_setup` 获取值： 

```C
static int __init vsyscall_setup(char *str)
{
	if (str) {
		if (!strcmp("emulate", str))
			vsyscall_mode = EMULATE;
		else if (!strcmp("native", str))
			vsyscall_mode = NATIVE;
		else if (!strcmp("none", str))
			vsyscall_mode = NONE;
		else
			return -EINVAL;

		return 0;
	}

	return -EINVAL;
}
```

函数将在早期的内核分析时被调用：

```C
early_param("vsyscall", vsyscall_setup);
```

关于 `early_param` 宏的更多信息可以在[第六章](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-6.html) Linux 内核初始化中找到。

在函数 `vsyscall_map` 的最后仅通过 [BUILD_BUG_ON](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-1.html) 宏检查  `vsyscall` 内存页的虚拟地址是否等于变量 `VSYSCALL_ADDR` :

```C
BUILD_BUG_ON((unsigned long)__fix_to_virt(VSYSCALL_PAGE) !=
             (unsigned long)VSYSCALL_ADDR);
```

 就这样`vsyscall` 内存页设置完毕。上述的结果如下： 若设置 `vsyscall=native` 内核命令行参数，虚拟内存调用将以 [arch/x86/entry/vsyscall/vsyscall_emu_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry/vsyscall/vsyscall_emu_64.S) 文件中本地 `系统调用` 指令的方式执行。  [glibc](https://en.wikipedia.org/wiki/GNU_C_Library) 知道虚拟系统调用处理器的地址。注意虚拟系统调用的地址以 `1024` (或 `0x400`) 比特对齐。

```assembly
__vsyscall_page:
	mov $__NR_gettimeofday, %rax
	syscall
	ret

	.balign 1024, 0xcc
	mov $__NR_time, %rax
	syscall
	ret

	.balign 1024, 0xcc
	mov $__NR_getcpu, %rax
	syscall
	ret
```

 `vsyscall` 内存页的起始地址为 `ffffffffff600000` 。因此,  [glibc](https://en.wikipedia.org/wiki/GNU_C_Library) 知道所有虚拟系统调用处理器的地址。可以在 `glibc` 源码中找到这些地址的定义：

```C
#define VSYSCALL_ADDR_vgettimeofday   0xffffffffff600000
#define VSYSCALL_ADDR_vtime 	      0xffffffffff600400
#define VSYSCALL_ADDR_vgetcpu	      0xffffffffff600800
```

所有的虚拟系统调用请求都将映射至 `__vsyscall_page` + `VSYSCALL_ADDR_vsyscall_name` 偏置, 将虚拟内存系统调用的编号置于通用目的[寄存器](https://en.wikipedia.org/wiki/Processor_register)，本地的 x86_64 `系统调用`指令将被执行。

在第二种情况中, 若将 `vsyscall=emulate` 参数传递给内核命令行, 提升虚拟系统调用处理器的尝试导致一个 [page fault](https://en.wikipedia.org/wiki/Page_fault) 异常。 谨记,  `vsyscall` 内存页 具有 `__PAGE_KERNEL_VVAR` 的访问权限，这将禁止执行。 `do_page_fault` 函数是 `#PF` 或 page fault 的处理器。它将尝试了解最后一次 page fault 的原因。一种可能的场景是 `vsyscall` 模式为 `emulate` 情况下的虚拟系统调用。此时 `vsyscall` 将被 [arch/x86/entry/vsyscall/vsyscall_64.c](https://github.com/torvalds/linux/blob/master/arch/x86/entry/vsyscall/vsyscall_64.c) 源码中定义的 `emulate_vsyscall` 函数处理。

The `emulate_vsyscall` function gets the number of a virtual system call, checks it, prints error and sends [segementation fault](https://en.wikipedia.org/wiki/Segmentation_fault) single:

```C
...
...
...
vsyscall_nr = addr_to_vsyscall_nr(address);
if (vsyscall_nr < 0) {
	warn_bad_vsyscall(KERN_WARNING, regs, "misaligned vsyscall...);
	goto sigsegv;
}
...
...
...
sigsegv:
	force_sig(SIGSEGV, current);
	reutrn true;
```

As it checked number of a virtual system call, it does some yet another checks like `access_ok` violations and execute system call function depends on the number of a virtual system call:

```C
switch (vsyscall_nr) {
	case 0:
		ret = sys_gettimeofday(
			(struct timeval __user *)regs->di,
			(struct timezone __user *)regs->si);
		break;
	...
	...
	...
}
```

In the end we put the result of the `sys_gettimeofday` or another virtual system call handler to the `ax` general purpose register, as we did it with the normal system calls and restore the [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) register and add `8` bytes to the [stack pointer](https://en.wikipedia.org/wiki/Stack_register) register. This operation emulates `ret` instruction.

```C
	regs->ax = ret;

do_ret:
	regs->ip = caller;
	regs->sp += 8;
	return true;
```		

That's all. Now let's look on the modern concept - `vDSO`.

Introduction to vDSO
--------------------------------------------------------------------------------

As I already wrote above, `vsyscall` is an obsolete concept and replaced by the `vDSO` or `virtual dynamic shared object`. The main difference between the `vsyscall` and `vDSO` mechanisms is that `vDSO` maps memory pages into each process in a shared object [form](https://en.wikipedia.org/wiki/Library_%28computing%29#Shared_libraries), but `vsyscall` is static in memory and has the same address every time. For the `x86_64` architecture it is called -`linux-vdso.so.1`. All userspace applications linked with this shared library via the `glibc`. For example:

```
~$ ldd /bin/uname
	linux-vdso.so.1 (0x00007ffe014b7000)
	libc.so.6 => /lib64/libc.so.6 (0x00007fbfee2fe000)
	/lib64/ld-linux-x86-64.so.2 (0x00005559aab7c000)
```

Or:

```
~$ sudo cat /proc/1/maps | grep vdso
7fff39f73000-7fff39f75000 r-xp 00000000 00:00 0       [vdso]
```

Here we can see that [uname](https://en.wikipedia.org/wiki/Uname) util was linked with the three libraries:

* `linux-vdso.so.1`;
* `libc.so.6`;
* `ld-linux-x86-64.so.2`.

The first provides `vDSO` functionality, the second is `C` [standard library](https://en.wikipedia.org/wiki/C_standard_library) and the third is the program interpreter (more about this you can read in the part that describes [linkers](https://xinqiu.gitbooks.io/linux-insides-cn/content/Misc/linux-misc-3.html)). So, the `vDSO` solves limitations of the `vsyscall`. Implementation of the `vDSO` is similar to `vsyscall`.

Initialization of the `vDSO` occurs in the `init_vdso` function that defined in the [arch/x86/entry/vdso/vma.c](https://github.com/torvalds/linux/blob/master/arch/x86/entry/vdso/vma.c) source code file. This function starts from the initialization of the `vDSO` images for 32-bits and 64-bits depends on the `CONFIG_X86_X32_ABI` kernel configuration option:

```C
static int __init init_vdso(void)
{
	init_vdso_image(&vdso_image_64);

#ifdef CONFIG_X86_X32_ABI
	init_vdso_image(&vdso_image_x32);
#endif	
```

Both function initialize the `vdso_image` structure. This structure is defined in the two generated source code files: the [arch/x86/entry/vdso/vdso-image-64.c](https://github.com/torvalds/linux/blob/master/arch/x86/entry/vdso/vdso-image-64.c) and the [arch/x86/entry/vdso/vdso-image-64.c](https://github.com/torvalds/linux/blob/master/arch/x86/entry/vdso/vdso-image-64.c). These source code files generated by the [vdso2c](https://github.com/torvalds/linux/blob/master/arch/x86/entry/vdso/vdso2c.c) program from the different source code files, represent different approaches to call a system call like `int 0x80`, `sysenter` and etc. The full set of the images depends on the kernel configuration.

For example for the `x86_64` Linux kernel it will contain `vdso_image_64`:

```C
#ifdef CONFIG_X86_64
extern const struct vdso_image vdso_image_64;
#endif
```

But for the `x86` - `vdso_image_32`:

```C
#ifdef CONFIG_X86_X32
extern const struct vdso_image vdso_image_x32;
#endif
```

If our kernel is configured for the `x86` architecture or for the `x86_64` and compability mode, we will have ability to call a system call with the `int 0x80` interrupt, if compability mode is enabled, we will be able to call a system call with the native `syscall instruction` or `sysenter` instruction in other way:

```C
#if defined CONFIG_X86_32 || defined CONFIG_COMPAT
  extern const struct vdso_image vdso_image_32_int80;
#ifdef CONFIG_COMPAT
  extern const struct vdso_image vdso_image_32_syscall;
#endif
 extern const struct vdso_image vdso_image_32_sysenter;
#endif
```

As we can understand from the name of the `vdso_image` structure, it represents image of the `vDSO` for the certain mode of the system call entry. This structure contains information about size in bytes of the `vDSO` area that always a multiple of `PAGE_SIZE` (`4096` bytes), pointer to the text mapping, start and end address of the `alternatives` (set of instructions with better alternatives for the certain type of the processor) and etc. For example `vdso_image_64` looks like this:

```C
const struct vdso_image vdso_image_64 = {
	.data = raw_data,
	.size = 8192,
	.text_mapping = {
		.name = "[vdso]",
		.pages = pages,
	},
	.alt = 3145,
	.alt_len = 26,
	.sym_vvar_start = -8192,
	.sym_vvar_page = -8192,
	.sym_hpet_page = -4096,
};
```

Where the `raw_data` contains raw binary code of the 64-bit `vDSO` system calls which are `2` page size:

```C
static struct page *pages[2];
```

or 8 Kilobytes.

The `init_vdso_image` function is defined in the same source code file and just initializes the `vdso_image.text_mapping.pages`. First of all this function calculates the number of pages and initializes each `vdso_image.text_mapping.pages[number_of_page]` with the `virt_to_page` macro that converts given address to the `page` structure:

```C
void __init init_vdso_image(const struct vdso_image *image)
{
	int i;
	int npages = (image->size) / PAGE_SIZE;

	for (i = 0; i < npages; i++)
		image->text_mapping.pages[i] =
			virt_to_page(image->data + i*PAGE_SIZE);
	...
	...
	...
}
```

The `init_vdso` function passed to the `subsys_initcall` macro adds the given function to the `initcalls` list. All functions from this list will be called in the `do_initcalls` function from the [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) source code file:

```C
subsys_initcall(init_vdso);
```

Ok, we just saw initialization of the `vDSO` and initialization of `page` structures that are related to the memory pages that contain `vDSO` system calls. But to where do their pages map? Actually they are mapped by the kernel, when it loads binary to the memory. The Linux kernel calls the `arch_setup_additional_pages` function from the [arch/x86/entry/vdso/vma.c](https://github.com/torvalds/linux/blob/master/arch/x86/entry/vdso/vma.c) source code file that checks that `vDSO` enabled for the `x86_64` and calls the `map_vdso` function:

```C
int arch_setup_additional_pages(struct linux_binprm *bprm, int uses_interp)
{
	if (!vdso64_enabled)
		return 0;

	return map_vdso(&vdso_image_64, true);
}
```

The `map_vdso` function is defined in the same source code file and maps pages for the `vDSO` and for the shared `vDSO` variables. That's all. The main differences between the `vsyscall` and the `vDSO` concepts is that `vsyscal` has a static address of `ffffffffff600000` and implements `3` system calls, whereas the `vDSO` loads dynamically and implements four system calls:

* `__vdso_clock_gettime`;
* `__vdso_getcpu`;
* `__vdso_gettimeofday`;
* `__vdso_time`.


That's all.

Conclusion
--------------------------------------------------------------------------------

This is the end of the third part about the system calls concept in the Linux kernel. In the previous [part](http://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/linux-syscall-2.html) we discussed the implementation of the preparation from the Linux kernel side, before a system call will be handled and implementation of the `exit` process from a system call handler. In this part we continued to dive into the stuff which is related to the system call concept and learned two new concepts that are very similar to the system call - the `vsyscall` and the `vDSO`.

After all of these three parts, we know almost all things that are related to system calls, we know what system call is and why user applications need them.  We also know what occurs when a user application calls a system call and how the kernel handles system calls.

The next part will be the last part in this [chapter](http://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/index.html) and we will see what occurs when a user runs the program.

If you have questions or suggestions, feel free to ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create [issue](https://github.com/MintCN/linux-insides-zh/issues/new).

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/MintCN/linux-insides-zh).**

Links
--------------------------------------------------------------------------------

* [x86_64 memory map](https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/mm.txt)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [context switching](https://en.wikipedia.org/wiki/Context_switch)
* [ABI](https://en.wikipedia.org/wiki/Application_binary_interface)
* [virtual address](https://en.wikipedia.org/wiki/Virtual_address_space)
* [Segmentation](https://en.wikipedia.org/wiki/Memory_segmentation)
* [enum](https://en.wikipedia.org/wiki/Enumerated_type)
* [fix-mapped addresses](http://xinqiu.gitbooks.io/linux-insides-cn/content/MM/linux-mm-2.html)
* [glibc](https://en.wikipedia.org/wiki/GNU_C_Library)
* [BUILD_BUG_ON](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-1.html)
* [Processor register](https://en.wikipedia.org/wiki/Processor_register)
* [Page fault](https://en.wikipedia.org/wiki/Page_fault)
* [segementation fault](https://en.wikipedia.org/wiki/Segmentation_fault)
* [instruction pointer](https://en.wikipedia.org/wiki/Program_counter)
* [stack pointer](https://en.wikipedia.org/wiki/Stack_register)
* [uname](https://en.wikipedia.org/wiki/Uname)
* [Linkers](https://xinqiu.gitbooks.io/linux-insides-cn/content/Misc/linux-misc-3.html)
* [Previous part](http://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/linux-syscall-2.html)
