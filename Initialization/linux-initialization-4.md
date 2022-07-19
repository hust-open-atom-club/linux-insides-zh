内核初始化. Part 4.
================================================================================

Kernel entry point
================================================================================

还记得上一章的内容吗 - [跳转到内核入口之前的最后准备](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-3.md)？你应该还记得我们已经完成一系列初始化操作，并停在了调用位于`init/main.c`中的`start_kernel`函数之前.`start_kernel`函数是与体系架构无关的通用处理入口函数，尽管我们在此初始化过程中要无数次的返回arch/ 文件夹。如果你仔细看看`start_kernel`函数的内容，你将发现此函数涉及内容非常广泛。在此过程中约包含了86个调用函数，是的，你发现它真的是非常庞大但是此部分并不是全部的初始化过程，在当前阶段我们只看这些就可以了。此章节以及后续所有在[内核初始化过程](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/README.md)章节的内容将涉及并详述它。

`start_kernel`函数的主要目的是完成内核初始化并启动祖先进程(1号进程)。在祖先进程启动之前`start_kernel`函数做了很多事情，如[锁验证器](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt),根据处理器标识ID初始化处理器，开启cgroups子系统，设置每CPU区域环境，初始化[VFS](http://en.wikipedia.org/wiki/Virtual_file_system) Cache机制，初始化内存管理，rcu,vmalloc,scheduler(调度器),IRQs(中断向量表),ACPI(中断可编程控制器)以及其它很多子系统。只有经过这些步骤我们才看到本章最后一部分祖先进程启动的过程；同志们，如此复杂的内核子系统，有没有勾起你的学习欲望，有这么多的内核代码等着我们去征服，让我们开始吧。

**注意:在此大章节的所有内容 `Linux Kernel initialization process`，并不涉及内核调试相关，关于内核调试部分会有一个单独的章节来进行描述**

关于 `__attribute__` 
---------------------------------------------------------------------------------

正如我上述所写，`start_kernel`函数是定义在[init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c).从已知代码中我们能看到此函数使用了`__init`特性，你也许从其它地方了解过关于GCC `__attribute__`相关的内容。在内核初始化阶段这个机制在所有的函数中都是有必要的。

```C
	#define __init      __section(.init.text) __cold notrace
```

在初始化过程完成后，内核将通过调用`free_initmem`释放这些sections(段)。注意`__init`属性是通过`__cold`和`notrace`两个属性来定义的。第一个属性`cold`的目的是标记此函数很少使用所以编译器必须优化此函数的大小，第二个属性`notrace`定义如下：

```C
	#define notrace __attribute__((no_instrument_function))
```

含有`no_instrument_function`意思就是告诉编译器函数调用不产生环境变量(堆栈空间)。

在`start_kernel`函数的定义中，你也可以看到`__visible` 属性的扩展：

```
	#define __visible __attribute__((externally_visible))
```

含有`externally_visible`意思就是告诉编译器有一些过程在使用该函数或者变量，为了放至标记这个函数/变量是`unusable`。你可以在此[include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h)处查到这些属性表达式的含义。

start_kernel 初始化
--------------------------------------------------------------------------------
在start_kernel的初始之初你可以看到这两个变量：

```C
char *command_line;
char *after_dashes;
```
第一个变量表示内核命令行的全局指针，第二个变量将包含`parse_args`函数通过输入字符串中的参数'name=value'，寻找特定的关键字和调用正确的处理程序。我们不想在这个时候参与这两个变量的相关细节，但是会在接下来的章节看到。我们接着往下走，下一步我们看到了此函数:

```C
lockdep_init();
```

`lockdep_init` 初始化 [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt). 其实现是相当简单的，它只是初始化了两个哈希表 [list_head](https://github.com/MintCN/linux-insides-zh/blob/master/DataStructures/linux-datastructures-1.md)并设置`lockdep_initialized` 全局变量为`1`。
关于自旋锁 [spinlock](http://en.wikipedia.org/wiki/Spinlock)以及互斥锁[mutex](http://en.wikipedia.org/wiki/Mutual_exclusion) 如何获取请参考链接.

下一个函数是`set_task_stack_end_magic`，参数为`init_task`和设置`STACK_END_MAGIC` (`0x57AC6E9D`)。`init_task`代表初始化进程(任务)数据结构:

```C
struct task_struct init_task = INIT_TASK(init_task);
```
`task_struct` 存储了进程的所有相关信息。因为它很庞大，我在这本书并不会去介绍，详细信息你可以查看调度相关数据结构定义头文件 [include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h#L1278)。在此刻`task_sreuct`包含了超过`100`个字段！虽然你不会看到`task_struct`是在这本书中的解释，但是我们会经常使用它，因为它是介绍在Linux内核`进程`的基本知识。我将描述这个结构中字段的一些含义，因为我们在后面的实践中见到它们。

你也可以查看`init_task`的相关定义以及宏指令`INIT_TASK`的初始化流程。这个宏指令来自于[include/linux/init_task.h](https://github.com/torvalds/linux/blob/master/include/linux/init_task.h)在此刻只是设置和初始化了第一个进程来(0号进程)的值。例如这么设置：
* 初始化进程状态为 zero 或者 `runnable`. 一个可运行进程即为等待CPU去运行;
* 初始化仅存的标志位 - `PF_KTHREAD` 意思为 - 内核线程;
* 一个可运行的任务列表;
* 进程地址空间;
* 初始化进程堆栈 `&init_thread_info`  - `init_thread_union.thread_info` 和 `initthread_union` 使用共用体 - `thread_union` 包含了 `thread_info`进程信息以及进程栈:。


```C
union thread_union {
	struct thread_info thread_info;
    unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```
每个进程都有其自己的堆栈，`x86_64`架构的CPU一般支持的页表是16KB or 4个页框大小。我们注意stack变量被定义为数据并且类型是`unsigned long`。`thread_union`结构的下一个字段为`thread_union` 定义如下：

```C
struct thread_info {
        struct task_struct      *task;
        struct exec_domain      *exec_domain;
        __u32                   flags; 
        __u32                   status;
        __u32                   cpu;
        int                     saved_preempt_count;
        mm_segment_t            addr_limit;
        struct restart_block    restart_block;
        void __user             *sysenter_return;
        unsigned int            sig_on_uaccess_error:1;
        unsigned int            uaccess_err:1;
};
```
此结构占用52个字节。`thread_info`结构包含了特定体系架构相关的线程信息，我们都知道在`X86_64`架构上内核栈是逆生成而`thread_union.thread_info`结构则是正生长。所以进程进程栈是16KB并且`thread_info`是在栈底。还需我们处理`16 kilobytes - 62 bytes = 16332 bytes`.注意 `thread_union`代表一个联合体[union](http://en.wikipedia.org/wiki/Union_type)而不是结构体，用一张图来描述栈内存空间。
如下图所示:

```C
+-----------------------+
|                       |
|                       |
|        stack          |
|                       |
|_______________________|
|          |            |
|          |            |
|          |            |
|__________↓____________|             +--------------------+
|                       |             |                    |
|      thread_info      |<----------->|     task_struct    |
|                       |             |                    |
+-----------------------+             +--------------------+
```

http://www.quora.com/In-Linux-kernel-Why-thread_info-structure-and-the-kernel-stack-of-a-process-binds-in-union-construct

所以`INIT_TASK`宏指令就是`task_struct's`'结构。正如我上述所写，我并不会去描述这些字段的含义和值，在`INIT_TASK`赋值处理的时候我们很快能看到这些。

现在让我们回到`set_task_stack_end_magic`函数，这个函数被定义在[kernel/fork.c](https://github.com/torvalds/linux/blob/master/kernel/fork.c#L297)功能为设置[canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow) `init` 进程堆栈以检测堆栈溢出。


```C
void set_task_stack_end_magic(struct task_struct *tsk)
{
	unsigned long *stackend;
	stackend = end_of_stack(tsk);
	*stackend = STACK_END_MAGIC; /* for overflow detection */
}
```

上述函数比较简单，`set_task_stack_end_magic`函数的作用是先通过`end_of_stack`函数获取堆栈并赋给 `task_struct`。
关于检测配置需要打开内核配置宏`CONFIG_STACK_GROWSUP`。因为我们学习的是x86架构的初始化，堆栈是逆生成，所以堆栈底部为：

```C
(unsigned long *)(task_thread_info(p) + 1);
```

`task_thread_info`的定义如下，返回一个当前的堆栈；

```C
#define task_thread_info(task)  ((struct thread_info *)(task)->stack)
```

进程的栈底，我们写`STACK_END_MAGIC`这个值。如果设置`canary`，我们可以像这样子去检测堆栈：

```C
if (*end_of_stack(task) != STACK_END_MAGIC) {
        //
        // handle stack overflow here
		//
}
```

`set_task_stack_end_magic` 初始化完毕后的下一个函数是 `smp_setup_processor_id`.此函数在`x86_64`架构上是空函数：

```C
void __init __weak smp_setup_processor_id(void)
{
}
```

在此架构上没有实现此函数，但在别的体系架构的实现可以参考[s390](http://en.wikipedia.org/wiki/IBM_ESA/390) and [arm64](http://en.wikipedia.org/wiki/ARM_architecture#64.2F32-bit_architecture).

我们接着往下走，下一个函数是`debug_objects_early_init`。此函数的执行几乎和`lockdep_init`是一样的，但是填充的哈希对象是调试相关。上述我已经表明，关于内核调试部分会在后续专门有一个章节来完成。

`debug_object_early_init`函数之后我们看到调用了`boot_init_stack_canary`函数。`task_struct->canary` 的值利用了GCC特性，但是此特性需要先使能内核`CONFIG_CC_STACKPROTECTOR`宏后才可以使用。
 `boot_init_stack_canary` 什么也没有做， 否则基于随机数和随机池产生 [TSC](http://en.wikipedia.org/wiki/Time_Stamp_Counter):

```C
get_random_bytes(&canary, sizeof(canary));
tsc = __native_read_tsc();
canary += tsc + (tsc << 32UL);
```

我们要获取随机数, 我们可以给`stack_canary` 字段 `task_struct`赋值：

```C
current->stack_canary = canary;
```

然后将此值写入IRQ堆栈的顶部:

```C
this_cpu_write(irq_stack_union.stack_canary, canary); // read below about this_cpu_write
```

关于IRQ的章节我们这里也不会详细剖析, 关于这部分介绍看这里[IRQs](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29).如果canary被设置, 关闭本地中断注册bootstrap CPU以及CPU maps. 我们关闭本地中断 (interrupts for current CPU) 使用 `local_irq_disable` 函数，展开后原型为 `arch_local_irq_disable` 函数[include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h):

```C
static inline notrace void arch_local_irq_enable(void)
{
        native_irq_enable();
}
```
如果`native_irq_enable`通过`cli`指令判断架构，这里是`X86_64`，
Where `native_irq_enable` is `cli` instruction for `x86_64`.中断的关闭(屏蔽)我们可以通过注册当前CPU ID到CPU bitmap来实现。 

激活第一个CPU
---------------------------------------------------------------------------------

当前已经走到`start_kernel`函数中的`boot_cpu_init`函数，此函数主要为了通过掩码初始化每一个CPU。首先我们需要获取当前处理器的ID通过下面函数：

```C
int cpu = smp_processor_id();
```

现在是0. 如果`CONFIG_DEBUG_PREEMPT` 宏配置了那么 `smp_processor_id` 的值就来自于 `raw_smp_processor_id` 函数，原型如下:

```C
#define raw_smp_processor_id() (this_cpu_read(cpu_number))
```

`this_cpu_read` 函数与其它很多函数一样如(`this_cpu_write`, `this_cpu_add` 等等...) 被定义在[include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h) 此部分函数主要为对 `this_cpu` 进行操作. 这些操作提供不同的对每cpu[per-cpu](http://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html) 变量相关访问方式. 譬如让我们来看看这个函数 `this_cpu_read`:

```
__pcpu_size_call_return(this_cpu_read_, pcp)
```
还记得上面我们所写，每cpu变量`cpu_number` 的值是`this_cpu_read`通过`raw_smp_processor_id`来得到，现在让我们看看 `__pcpu_size_call_return`的执行：

```C
#define __pcpu_size_call_return(stem, variable)                         \
({                                                                      \
        typeof(variable) pscr_ret__;                                    \
        __verify_pcpu_ptr(&(variable));                                 \
        switch(sizeof(variable)) {                                      \
        case 1: pscr_ret__ = stem##1(variable); break;                  \
        case 2: pscr_ret__ = stem##2(variable); break;                  \
        case 4: pscr_ret__ = stem##4(variable); break;                  \
        case 8: pscr_ret__ = stem##8(variable); break;                  \
        default:                                                        \
                __bad_size_call_parameter(); break;                     \
        }                                                               \
        pscr_ret__;                                                     \
}) 
```
是的，此函数虽然看起起奇怪但是它的实现是简单的，我们看到`pscr_ret__` 变量的定义是`int`类型，为什么是int类型呢？好吧，`变量`是`common_cpu` 它声明了每cpu(per-cpu)变量:

```C
DECLARE_PER_CPU_READ_MOSTLY(int, cpu_number);
```

在下一个步骤中我们调用了`__verify_pcpu_ptr`通过使用一个有效的每cpu变量指针来取地址得到`cpu_number`。之后我们通过`pscr_ret__` 函数设置变量的大小，`common_cpu`变量是`int`,所以它的大小是4字节。意思就是我们通过`this_cpu_read_4(common_cpu)`获取cpu变量其大小被`pscr_ret__`决定。在`__pcpu_size_call_return`的结束 我们调用了__pcpu_size_call_return：

```C
#define this_cpu_read_4(pcp)       percpu_from_op("mov", pcp)
```

需要调用`percpu_from_op` 并且通过`mov`指令来传递每cpu变量，`percpu_from_op`的内联扩展如下：


```C
asm("movl %%gs:%1,%0" : "=r" (pfo_ret__) : "m" (common_cpu))
```

让我们尝试理解此函数是如果工作的，`gs`段寄存器包含每个CPU区域的初始值，这里我们通过`mov`指令copy `common_cpu`到内存中去，此函数还有另外的形式：

```C
this_cpu_read(common_cpu)
```

等价于:

```C
movl %gs:$common_cpu, $pfo_ret__
```

由于我们没有设置每个CPU的区域,我们只有一个 - 为当前CPU的值`zero` 通过此函数 `smp_processor_id`返回.

返回的ID表示我们处于哪一个CPU上, `boot_cpu_init` 函数设置了CPU的在线, 激活, 当前的设置为:

```C
set_cpu_online(cpu, true);
set_cpu_active(cpu, true);
set_cpu_present(cpu, true);
set_cpu_possible(cpu, true);
```

上述我们所有使用的这些CPU的配置我们称之为- CPU掩码`cpumask`. `cpu_possible` 则是设置支持CPU热插拔时候的CPU ID. `cpu_present` 表示当前热插拔的CPU. `cpu_online`表示当前所有在线的CPU以及通过 `cpu_present` 来决定被调度出去的CPU. CPU热插拔的操作需要打开内核配置宏`CONFIG_HOTPLUG_CPU`并且将 `possible == present` 以及`active == online`选项禁用。这些功能都非常相似，每个函数都需要检查第二个参数，如果设置为`true`，需要通过调用`cpumask_set_cpu` or `cpumask_clear_cpu`来改变状态。

譬如我们可以通过true或者第二个参数来这么调用：

```C
cpumask_set_cpu(cpu, to_cpumask(cpu_possible_bits));
```

 让我们继续尝试理解`to_cpumask`宏指令，此宏指令转化为一个位图通过`struct cpumask *`，CPU掩码提供了位图集代表了当前系统中所有的CPU's，每CPU都占用1bit，CPU掩码相关定义通过`cpu_mask`结构定义:

```C
typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t;
```
在来看下面一组函数定义了位图宏指令。

```C
#define DECLARE_BITMAP(name, bits) unsigned long name[BITS_TO_LONGS(bits)]
```

正如我们看到的定义一样， `DECLARE_BITMAP`宏指令的原型是一个`unsigned long`的数组，现在让我们查看如何执行`to_cpumask`:

```C
#define to_cpumask(bitmap)                                              \
        ((struct cpumask *)(1 ? (bitmap)                                \
                            : (void *)sizeof(__check_is_bitmap(bitmap))))
```

我不知道你是怎么想的, 但是我是这么想的，我看到此函数其实就是一个条件判断语句当条件为真的时候，但是为什么执行`__check_is_bitmap`？让我们看看`__check_is_bitmap`的定义：

```C
static inline int __check_is_bitmap(const unsigned long *bitmap)
{
        return 1;
}
```

原来此函数始终返回1，事实上我们需要这样的函数才达到我们的目的： 它在编译时给定一个`bitmap`，换句话将就是检查`bitmap`的类型是否是`unsigned long *`,因此我们仅仅通过`to_cpumask`宏指令将类型为`unsigned long`的数组转化为`struct cpumask *`。现在我们可以调用`cpumask_set_cpu` 函数，这个函数仅仅是一个 `set_bit`给CPU掩码的功能函数。所有的这些`set_cpu_*`函数的原理都是一样的。

如果你还不确定`set_cpu_*`这些函数的操作并且不能理解 `cpumask`的概念，不要担心。你可以通过读取这些章节[cpumask](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-2.html) or [documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt).来继续了解和学习这些函数的原理。

现在我们已经激活第一个CPU，我们继续接着start_kernel函数往下走，下面的函数是`page_address_init`,但是此函数不执行任何操作，因为只有当所有内存不能直接映射的时候才会执行。

Linux 内核的第一条打印信息
---------------------------------------------------------------------------------

下面调用了pr_notice函数。

```C
#define pr_notice(fmt, ...) \
    printk(KERN_NOTICE pr_fmt(fmt), ##__VA_ARGS__)
```


pr_notice其实是printk的扩展，这里我们使用它打印了Linux 的banner。

```C
pr_notice("%s", linux_banner);
```

打印的是内核的版本号以及编译环境信息:

```
Linux version 4.0.0-rc6+ (alex@localhost) (gcc version 4.9.1 (Ubuntu 4.9.1-16ubuntu6) ) #319 SMP
```

依赖于体系结构的初始化部分
---------------------------------------------------------------------------------

下个步骤我们就要进入到指定的体系架构的初始函数，Linux 内核初始化体系架构相关调用`setup_arch`函数，这又是一个类型于`start_kernel`的庞大函数，这里我们仅仅简单描述，在下一个章节我们将继续深入。指定体系架构的内容，我们需要再一次阅读`arch/`目录，`setup_arch`函数定义在[arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c) 文件中，此函数就一个参数-内核命令行。

此函数解析内核的段`_text`和`_data`来自于`_text`符号和`_bss_stop`(你应该还记得此文件[arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S#L46))。我们使用`memblock`来解析内存块。

```C
memblock_reserve(__pa_symbol(_text), (unsigned long)__bss_stop - (unsigned long)_text);
```

你可以阅读关于`memblock`的相关内容在[Linux kernel memory management Part 1.](http://xinqiu.gitbooks.io/linux-insides-cn/content/MM/linux-mm-1.html)，你应该还记得`memblock_reserve`函数的两个参数：

* base physical address of a memory block;
* size of a memory block.

我们可以通过`__pa_symbol`宏指令来获取符号表`_text`段中的物理地址

```C
#define __pa_symbol(x) \
	__phys_addr_symbol(__phys_reloc_hide((unsigned long)(x)))
```

上述宏指令调用 `__phys_reloc_hide` 宏指令来填充参数，`__phys_reloc_hide`宏指令在`x86_64`上返回的参数是给定的。宏指令 `__phys_addr_symbol`的执行是简单的，只是减去从`_text`符号表中读到的内核的符号映射地址并且加上物理地址的基地址。

```C
#define __phys_addr_symbol(x) \
 ((unsigned long)(x) - __START_KERNEL_map + phys_base)
```

`memblock_reserve`函数对内存页进行分配。


保留可用内存初始化initrd
---------------------------------------------------------------------------------

之后我们保留替换内核的text和data段用来初始化[initrd](http://en.wikipedia.org/wiki/Initrd),我们暂时不去了解initrd的详细信息，你仅仅只需要知道根文件系统就是通过这种方式来进行初始化，这就是`early_reserve_initrd` 函数的工作，此函数获取RAM DISK的基地址、RAM DISK的大小以及RAM DISK的结束地址。

```C
u64 ramdisk_image = get_ramdisk_image();
u64 ramdisk_size  = get_ramdisk_size();
u64 ramdisk_end   = PAGE_ALIGN(ramdisk_image + ramdisk_size);
```

如果你阅读过这些章节[Linux Kernel Booting Process](http://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/index.html)，你就知道所有的这些参数都来自于`boot_params`，时刻谨记`boot_params`在boot期间已经被赋值，内核启动头包含了一下几个字段用来描述RAM DISK：
```
Field name:	ramdisk_image
Type:		write (obligatory)
Offset/size:	0x218/4
Protocol:	2.00+

  The 32-bit linear address of the initial ramdisk or ramfs.  Leave at
  zero if there is no initial ramdisk/ramfs.
```


我们可以得到关于 `boot_params`的一些信息. 具体查看`get_ramdisk_image`:

```C
static u64 __init get_ramdisk_image(void)
{
        u64 ramdisk_image = boot_params.hdr.ramdisk_image;

        ramdisk_image |= (u64)boot_params.ext_ramdisk_image << 32;

        return ramdisk_image;
}
```

关于32位的ramdisk的地址，我们可以阅读此部分内容来获取[Documentation/x86/zero-page.txt](https://github.com/0xAX/linux/blob/master/Documentation/x86/zero-page.txt):

```
0C0/004	ALL	ext_ramdisk_image ramdisk_image high 32bits
```

32位变化后，我们获取64位的ramdisk原理一样，为此我们可以检查bootloader 提供的ramdisk信息：

```C
if (!boot_params.hdr.type_of_loader ||
    !ramdisk_image || !ramdisk_size)
	return;
```

并保留内存块将ramdisk传输到最终的内存地址，然后进行初始化：

```C
memblock_reserve(ramdisk_image, ramdisk_end - ramdisk_image);
```

结束语
---------------------------------------------------------------------------------

以上就是第四部分关于内核初始化的部分内容，我们从`start_kernel`函数开始一直到指定体系架构初始化`setup_arch`的过程中停止，那么在下一个章节我们将继续研究体系架构相关的初始化内容。

如果你有任何的问题或者建议，你可以留言，也可以直接发消息给我[twitter](https://twitter.com/0xAX)。

**很抱歉，英语并不是我的母语，非常抱歉给您阅读带来不便，如果你发现文中描述有任何问题，请提交一个 PR 到 [linux-insides](https://github.com/MintCN/linux-insides-zh).**

链接
--------------------------------------------------------------------------------

* [GCC function attributes](https://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html)
* [this_cpu operations](https://www.kernel.org/doc/Documentation/this_cpu_ops.txt)
* [cpumask](http://www.crashcourse.ca/wiki/index.php/Cpumask)
* [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)
* [cgroups](http://en.wikipedia.org/wiki/Cgroups)
* [stack buffer overflow](http://en.wikipedia.org/wiki/Stack_buffer_overflow)
* [IRQs](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [initrd](http://en.wikipedia.org/wiki/Initrd)
* [Previous part](https://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-3.html)
