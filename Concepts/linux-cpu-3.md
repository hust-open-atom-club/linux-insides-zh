initcall 机制
================================================================================

介绍
--------------------------------------------------------------------------------


就像你从标题所理解的，这部分将涉及 Linux 内核中有趣且重要的概念，称之为 `initcall`。在 Linux 内核中，我们可以看到类似这样的定义：

```C
early_param("debug", debug_kernel);
```

或者

```C
arch_initcall(init_pit_clocksource);
```

在我们分析这个机制在内核中是如何实现的之前，我们必须了解这个机制是什么，以及在 Linux 内核中是如何使用它的。像这样的定义表示一个 [回调函数](https://en.wikipedia.org/wiki/Callback_%28computer_programming%29) ，它们会在 Linux 内核启动中或启动后调用。实际上 `initcall` 机制的要点是确定内置模块和子系统初始化的正确顺序。举个例子，我们来看看下面的函数：

```C
static int __init nmi_warning_debugfs(void)
{
    debugfs_create_u64("nmi_longest_ns", 0644,
                       arch_debugfs_dir, &nmi_longest_ns);
    return 0;
}
```

这个函数出自源码文件 [arch/x86/kernel/nmi.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/nmi.c)。我们可以看到，这个函数只是在 `arch_debugfs_dir` 目录中创建 `nmi_longest_ns` [debugfs](https://en.wikipedia.org/wiki/Debugfs) 文件。实际上，只有在 `arch_debugfs_dir` 创建后，才会创建这个 `debugfs` 文件。这个目录是在 Linux 内核特定架构的初始化期间创建的。实际上，该目录将在源码文件 [arch/x86/kernel/kdebugfs.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/kdebugfs.c) 的 `arch_kdebugfs_init` 函数中创建。注意 `arch_kdebugfs_init` 函数也被标记为 `initcall`。

```C
arch_initcall(arch_kdebugfs_init);
```

Linux 内核在调用 `fs` 相关的 `initcalls` 之前调用所有特定架构的 `initcalls`。因此，只有在 `arch_kdebugfs_dir` 目录创建以后才会创建我们的 `nmi_longest_ns`。实际上，Linux 内核提供了八个级别的主 `initcalls`：

* `early`;
* `core`;
* `postcore`;
* `arch`;
* `susys`;
* `fs`;
* `device`;
* `late`.

它们的所有名称是由数组 `initcall_level_names` 来描述的，该数组定义在源码文件 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 中：

```C
static char *initcall_level_names[] __initdata = {
	"early",
	"core",
	"postcore",
	"arch",
	"subsys",
	"fs",
	"device",
	"late",
};
```

所有用这些标识符标记为 `initcall` 的函数将会以相同的顺序被调用，或者说，`early initcalls` 会首先被调用，其次是 `core initcalls`，以此类推。现在，我们对 `initcall` 机制了解点了，所以我们可以开始潜入 Linux 内核源码，来看看这个机制是如何实现的。

initcall 机制在 Linux 内核中的实现
--------------------------------------------------------------------------------

Linux 内核提供了一组来自头文件 [include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h) 的宏，来标记给定的函数为 `initcall`。所有这些宏都相当简单：

```C
#define early_initcall(fn)		__define_initcall(fn, early)
#define core_initcall(fn)		__define_initcall(fn, 1)
#define postcore_initcall(fn)		__define_initcall(fn, 2)
#define arch_initcall(fn)		__define_initcall(fn, 3)
#define subsys_initcall(fn)		__define_initcall(fn, 4)
#define fs_initcall(fn)			__define_initcall(fn, 5)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define late_initcall(fn)		__define_initcall(fn, 7)
```

我们可以看到，这些宏只是从同一个头文件的 `__define_initcall` 宏的调用扩展而来。此外，`__define_initcall` 宏有两个参数：

* `fn` - 在调用某个级别 `initcalls` 时调用的回调函数；
* `id` - 识别 `initcall` 的标识符，用来防止两个相同的 `initcalls` 指向同一个处理函数时出现错误。

`__define_initcall` 宏的实现如下所示：

```C
#define __define_initcall(fn, id) \
	static initcall_t __initcall_##fn##id __used \
	__attribute__((__section__(".initcall" #id ".init"))) = fn; \
	LTO_REFERENCE_INITCALL(__initcall_##fn##id)
```

要了解 `__define_initcall` 宏，首先让我们来看下 `initcall_t` 类型。这个类型定义在同一个 [头文件]() 中，它表示一个返回 [整形](https://en.wikipedia.org/wiki/Integer)指针的函数指针，这将是 `initcall` 的结果：

```C
typedef int (*initcall_t)(void);
```

现在让我们回到 `_-define_initcall` 宏。[##](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html) 提供了连接两个符号的能力。在我们的例子中，`__define_initcall` 宏的第一行产生了 `.initcall id .init` [ELF 部分](http://www.skyfree.org/linux/references/ELF_Format.pdf) 给定函数的定义，并标记以下 [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection) 属性： `__initcall_function_name_id` 和 `__used`。如果我们查看表示内核链接脚本数据的 [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/vmlinux.lds.h) 头文件，我们会看到所有的 `initcalls` 部分都将放在 `.data` 段：

```C
#define INIT_CALLS					\
		VMLINUX_SYMBOL(__initcall_start) = .;	\
		*(.initcallearly.init)					\
		INIT_CALLS_LEVEL(0)					    \
		INIT_CALLS_LEVEL(1)					    \
		INIT_CALLS_LEVEL(2)					    \
		INIT_CALLS_LEVEL(3)					    \
		INIT_CALLS_LEVEL(4)					    \
		INIT_CALLS_LEVEL(5)					    \
		INIT_CALLS_LEVEL(rootfs)				\
		INIT_CALLS_LEVEL(6)					    \
		INIT_CALLS_LEVEL(7)					    \
		VMLINUX_SYMBOL(__initcall_end) = .;

#define INIT_DATA_SECTION(initsetup_align)	\
	.init.data : AT(ADDR(.init.data) - LOAD_OFFSET) {	   \
        ...                                                \
        INIT_CALLS						                   \
        ...                                                \
	}

```

第二个属性 - `__used`，定义在 [include/linux/compiler-gcc.h](https://github.com/torvalds/linux/blob/master/include/linux/compiler-gcc.h) 头文件中，它扩展了以下 `gcc` 定义：

```C
#define __used   __attribute__((__used__))
```

它防止 `定义了变量但未使用` 的告警。宏 `__define_initcall` 最后一行是：

```C
LTO_REFERENCE_INITCALL(__initcall_##fn##id)
```

这取决于 `CONFIG_LTO` 内核配置选项，只为编译器提供[链接时间优化](https://gcc.gnu.org/wiki/LinkTimeOptimization)存根：

```
#ifdef CONFIG_LTO
#define LTO_REFERENCE_INITCALL(x) \
        static __used __exit void *reference_##x(void)  \
        {                                               \
                return &x;                              \
        }
#else
#define LTO_REFERENCE_INITCALL(x)
#endif
```

为了防止当模块中的变量没有引用时而产生的任何问题，它被移到了程序末尾。这就是关于 `__define_initcall` 宏的全部了。所以，所有的 `*_initcall` 宏将会在Linux内核编译时扩展，所有的 `initcalls` 会放置在它们的段内，并可以通过 `.data` 段来获取，Linux 内核在初始化过程中就知道在哪儿去找到 `initcall` 并调用它。

既然 Linux 内核可以调用 `initcalls`，我们就来看下 Linux 内核是如何做的。这个过程从 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 头文件的 `do_basic_setup` 函数开始：

```C
static void __init do_basic_setup(void)
{
    ...
    ...
    ...
   	do_initcalls();
    ...
    ...
    ...
}
```

该函数在 Linux 内核初始化过程中调用，调用时机是主要的初始化步骤，比如内存管理器相关的初始化、`CPU` 子系统等完成之后。`do_initcalls` 函数只是遍历 `initcall` 级别数组，并调用每个级别的 `do_initcall_level` 函数：

```C
static void __init do_initcalls(void)
{
	int level;

	for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
		do_initcall_level(level);
}
```

`initcall_levels` 数组在同一个源码[文件](https://github.com/torvalds/linux/blob/master/init/main.c)中定义，包含了定义在 `__define_initcall` 宏中的那些段的指针：

```C
static initcall_t *initcall_levels[] __initdata = {
	__initcall0_start,
	__initcall1_start,
	__initcall2_start,
	__initcall3_start,
	__initcall4_start,
	__initcall5_start,
	__initcall6_start,
	__initcall7_start,
	__initcall_end,
};
```

如果你有兴趣，你可以在 Linux 内核编译后生成的链接器脚本 `arch/x86/kernel/vmlinux.lds` 中找到这些段：

```
.init.data : AT(ADDR(.init.data) - 0xffffffff80000000) {
    ...
    ...
    ...
    ...
    __initcall_start = .;
    *(.initcallearly.init)
    __initcall0_start = .;
    *(.initcall0.init)
    *(.initcall0s.init)
    __initcall1_start = .;
    ...
    ...
}
```

如果你对这些不熟，可以在本书的某些[部分](https://0xax.gitbooks.io/linux-insides/content/Misc/linkers.html)了解更多关于[链接器](https://en.wikipedia.org/wiki/Linker_%28computing%29)的信息。

正如我们刚看到的，`do_initcall_level` 函数有一个参数 - `initcall` 的级别，做了以下两件事：首先这个函数拷贝了 `initcall_command_line`，这是通常内核包含了各个模块参数的[命令行](https://www.kernel.org/doc/Documentation/kernel-parameters.txt)的副本，并用 [kernel/params.c](https://github.com/torvalds/linux/blob/master/kernel/params.c)源码文件的 `parse_args` 函数解析它，然后调用各个级别的 `do_on_initcall` 函数：

```C
for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
		do_one_initcall(*fn);
```

`do_on_initcall` 为我们做了主要的工作。我们可以看到，这个函数有一个参数表示 `initcall` 回调函数，并调用给定的回调函数：

```C
int __init_or_module do_one_initcall(initcall_t fn)
{
	int count = preempt_count();
	int ret;
	char msgbuf[64];

	if (initcall_blacklisted(fn))
		return -EPERM;

	if (initcall_debug)
		ret = do_one_initcall_debug(fn);
	else
		ret = fn();

	msgbuf[0] = 0;

	if (preempt_count() != count) {
		sprintf(msgbuf, "preemption imbalance ");
		preempt_count_set(count);
	}
	if (irqs_disabled()) {
		strlcat(msgbuf, "disabled interrupts ", sizeof(msgbuf));
		local_irq_enable();
	}
	WARN(msgbuf[0], "initcall %pF returned with %s\n", fn, msgbuf);

	return ret;
}
```

让我们来试着理解 `do_on_initcall` 函数做了什么。首先我们增加 [preemption](https://en.wikipedia.org/wiki/Preemption_%28computing%29) 计数，以便我们稍后进行检查，确保它不是不平衡的。这步以后，我们可以看到 `initcall_backlist` 函数的调用，这个函数遍历包含了 `initcalls` 黑名单的 `blacklisted_initcalls` 链表，如果 `initcall` 在黑名单里就释放它：

```C
list_for_each_entry(entry, &blacklisted_initcalls, next) {
	if (!strcmp(fn_name, entry->buf)) {
		pr_debug("initcall %s blacklisted\n", fn_name);
		kfree(fn_name);
		return true;
	}
}
```

黑名单的 `initcalls` 保存在 `blacklisted_initcalls` 链表中，这个链表是在早期 Linux 内核初始化时由 Linux 内核命令行来填充的。

处理完进入黑名单的 `initcalls`，接下来的代码直接调用 `initcall`：

```C
if (initcall_debug)
	ret = do_one_initcall_debug(fn);
else
	ret = fn();
```

取决于 `initcall_debug` 变量的值，`do_one_initcall_debug` 函数将调用 `initcall`，或直接调用 `fn()`。`initcall_debug` 变量定义在[同一个源码文件](https://github.com/torvalds/linux/blob/master/init/main.c)：

```C
bool initcall_debug;
```

该变量提供了向内核[日志缓冲区](https://en.wikipedia.org/wiki/Dmesg)打印一些信息的能力。可以通过 `initcall_debug` 参数从内核命令行中设置这个变量的值。从Linux内核命令行[文档](https://www.kernel.org/doc/Documentation/kernel-parameters.txt)可以看到：

```
initcall_debug	[KNL] Trace initcalls as they are executed.  Useful
                      for working out where the kernel is dying during
                      startup.
```

确实如此。如果我们看下 `do_one_initcall_debug` 函数的实现，我们会看到它与 `do_one_initcall` 函数做了一样的事，也就是说，`do_one_initcall_debug` 函数调用了给定的 `initcall`，并打印了一些和 `initcall` 相关的信息（比如当前任务的 [pid](https://en.wikipedia.org/wiki/Process_identifier)、`initcall` 的持续时间等）：

```C
static int __init_or_module do_one_initcall_debug(initcall_t fn)
{
	ktime_t calltime, delta, rettime;
	unsigned long long duration;
	int ret;

	printk(KERN_DEBUG "calling  %pF @ %i\n", fn, task_pid_nr(current));
	calltime = ktime_get();
	ret = fn();
	rettime = ktime_get();
	delta = ktime_sub(rettime, calltime);
	duration = (unsigned long long) ktime_to_ns(delta) >> 10;
	printk(KERN_DEBUG "initcall %pF returned %d after %lld usecs\n",
		 fn, ret, duration);

	return ret;
}
```

由于 `initcall` 被 `do_one_initcall` 或 `do_one_initcall_debug` 调用，我们可以看到在 `do_one_initcall` 函数末尾做了两次检查。第一个检查在initcall执行内部 `__preempt_count_add` 和 `__preempt_count_sub` 可能的执行次数，如果这个值和之前的可抢占计数不相等，我们就把 `preemption imbalance` 字符串添加到消息缓冲区，并设置正确的可抢占计数：

```C
if (preempt_count() != count) {
	sprintf(msgbuf, "preemption imbalance ");
	preempt_count_set(count);
}
```

稍后这个错误字符串就会被打印出来。最后检查本地 [IRQs](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 的状态，如果它们被禁用了，我们就将 `disabled interrupts` 字符串添加到我们的消息缓冲区，并为当前处理器使能 `IRQs`，以防出现 `IRQs` 被 `initcall` 禁用了但不再使能的情况出现：

```C
if (irqs_disabled()) {
	strlcat(msgbuf, "disabled interrupts ", sizeof(msgbuf));
	local_irq_enable();
}
```

这就是全部了。通过这种方式，Linux 内核以正确的顺序完成了很多子系统的初始化。现在我们知道 Linux 内核的 `initcall` 机制是怎么回事了。在这部分中，我们介绍了 `initcall` 机制的主要部分，但遗留了一些重要的概念。让我们来简单看下这些概念。

首先，我们错过了一个级别的 `initcalls`，就是 `rootfs initcalls`。和我们在本部分看到的很多宏类似，你可以在 [include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h) 头文件中找到 `rootfs_initcall` 的定义：

```C
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
```

从这个宏的名字我们可以理解到，它的主要目的是保存和 [rootfs](https://en.wikipedia.org/wiki/Initramfs) 相关的回调。除此之外，只有在与设备相关的东西没被初始化时，在文件系统级别初始化以后再初始化一些其它东西时才有用。例如，发生在源码文件 [init/initramfs.c](https://github.com/torvalds/linux/blob/master/init/initramfs.c) 中 `populate_rootfs` 函数里的解压  [initramfs](https://en.wikipedia.org/wiki/Initramfs)：

```C
rootfs_initcall(populate_rootfs);
```

在这里，我们可以看到熟悉的输出：

```
[    0.199960] Unpacking initramfs...
```

除了 `rootfs_initcall` 级别，还有其它的 `console_initcall`、 `security_initcall` 和其他辅助的  `initcall` 级别。我们遗漏的最后一件事，是 `*_initcall_sync` 级别的集合。在这部分我们看到的几乎每个 `*_initcall` 宏，都有 `_sync` 前缀的宏伴随：

```C
#define core_initcall_sync(fn)		__define_initcall(fn, 1s)
#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)
#define arch_initcall_sync(fn)		__define_initcall(fn, 3s)
#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)
#define fs_initcall_sync(fn)		__define_initcall(fn, 5s)
#define device_initcall_sync(fn)	__define_initcall(fn, 6s)
#define late_initcall_sync(fn)		__define_initcall(fn, 7s)
```

这些附加级别的主要目的是，等待所有某个级别的与模块相关的初始化例程完成。

这就是全部了。

结论
--------------------------------------------------------------------------------

在这部分中，我们看到了 Linux 内核的一项重要机制，即在初始化期间允许调用依赖于 Linux 内核当前状态的函数。

如果你有问题或建议，可随时在 twitter [0xAX](https://twitter.com/0xAX) 上联系我，给我发 [email](anotherworldofworld@gmail.com)，或者创建 [issue](https://github.com/0xAX/linux-insides/issues/new)。

**请注意英语不是我的母语，对此带来的不便，我很抱歉。如果你发现了任何错误，都可以给我发 PR 到[linux-insides](https://github.com/0xAX/linux-insides)。**.

链接
--------------------------------------------------------------------------------

* [callback](https://en.wikipedia.org/wiki/Callback_%28computer_programming%29)
* [debugfs](https://en.wikipedia.org/wiki/Debugfs)
* [integer type](https://en.wikipedia.org/wiki/Integer)
* [symbols concatenation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html)
* [GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [Link time optimization](https://gcc.gnu.org/wiki/LinkTimeOptimization)
* [Introduction to linkers](https://0xax.gitbooks.io/linux-insides/content/Misc/linkers.html)
* [Linux kernel command line](https://www.kernel.org/doc/Documentation/kernel-parameters.txt)
* [Process identifier](https://en.wikipedia.org/wiki/Process_identifier)
* [IRQs](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [rootfs](https://en.wikipedia.org/wiki/Initramfs)
* [previous part](https://0xax.gitbooks.io/linux-insides/content/Concepts/cpumask.html)
