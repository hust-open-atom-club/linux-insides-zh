中断和中断处理。 第九部分。
================================================================================

延后中断(软中断，Tasklets和工作队列)介绍
--------------------------------------------------------------------------------

这是[linux内核揭密](https://www.gitbook.com/book/xinqiu/linux-insides-cn/details)中断部分的第九小节，在[之前章节](https://www.gitbook.com/book/xinqiu/linux-insides-cn/content/interrupts/interrupts-8.html)我们了解了源文件[arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/irqinit.c)中`init_IRQ`的实现。接下来的这一节我们将继续深入学习和外部硬件中断相关的初始化。

在[init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c)中我们可以看到在`init_IRQ`函数后面调用了`softirq_init`函数。这个函数在源文件[kernel/softirq.c](https://github.com/torvalds/linux/blob/master/kernel/softirq.c)中定义，从名字我们可以看出，它的作用是初始化`软中断`或者也可以说是初始化`延后中断`。那么什么是延后中断？在讲解内核初始化过程的[部分](https://www.gitbook.com/book/xinqiu/linux-insides-cn/content/Initialization/linux-initialization-9.html)第九小结我们已经对他有了一些了解，Linux内核中一共有三种'延后中断'：

* `软中断`;
* `tasklets`;
* `工作队列`;

在这一小节我们将详细介绍这三种实现。就像我说的，我们对这个主题有一些了解。那么，现在是时间深入了解一下了。

延后中断
----------------------------------------------------------------------------------

对中断处理有一些严格的要求，总的来说有两种：

* 中断处理必须快速执行完毕
* 有时中断处理必须做很多冗长的事情

就像你所想到的，我们几乎不可能同时做到这两点，之前的中断被分为两部分：

* 前半部
* 后半部

`后半部`曾经是Linux内核延后中断执行的一种方式，但现在的实际情况已经不是这样了。这种遗留称谓现在作为名词代表所有延后中断执行的方式。伴随着内核对并行处理的支持，出于性能考虑，所有新的下半部实现方案都基于被称之为`ksoftirqd`(稍后将详细讨论)的内核线程。`ksoftirqd`中断处理方式几乎和硬件中断处理一样重要。中断延后处理会在系统负载较低的时候才执行一个中断的具体实现行为。如你所知，中断处理代码运行于禁止响应后续中断的中断处理上下文中，所以要避免长时间执行。但有时中断处理却又有很多的工作需要执行，所以中断处理有时会被分为两部分。第一部分中，中断处理先只做少量的最重要工作，接下来提交第二部分到内核调度，然后就结束运行。当系统比较空闲并且处理器上下文允许处理中断时，第二部分就会开始执行被延后的剩余中断任务。以上是对延后中断处理的简要介绍。

就像上面说的，延后中断(或者叫`软中断`)和`tasklets`是由一些内核线程(每个处理器一个线程)来执行的。每个处理器都有自己的内核线程，名字叫做`ksoftirqd/n`，n是处理器的编号。我们可以通过系统命令`systemd-cgls`看到它们：

```
$ systemd-cgls -k | grep ksoft
├─   3 [ksoftirqd/0]
├─  13 [ksoftirqd/1]
├─  18 [ksoftirqd/2]
├─  23 [ksoftirqd/3]
├─  28 [ksoftirqd/4]
├─  33 [ksoftirqd/5]
├─  38 [ksoftirqd/6]
├─  43 [ksoftirqd/7]
```

由`spawn_ksoftirqd`函数启动这些线程。就像我们看到的，这个函数在早期的[initcall](http://www.compsoc.man.ac.uk/~moz/kernelnewbies/documents/initcall/index.html)被调用。

```C
early_initcall(spawn_ksoftirqd);
```

延后中断在Linux内核编译时就静态的确定了，`open_softirq`函数负责`softirq`初始化。`open_softirq`在[kernel/softirq.c](https://github.com/torvalds/linux/blob/master/kernel/softirq.c)中定义：

```C
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
	softirq_vec[nr].action = action;
}
```

这个函数有两个参数：

* `softirq_vec`数组的索引序号
* 一个指向软中断处理函数的指针

我们首先来看`softirq_vec`数组：

```C
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;
```

它在同一个源文件中定义。`softirq_vec`数组包含了`NR_SOFTIRQS`(其值为10)个不同`softirq`类型的`softirq_action`。当前版本的Linux内核定义了十种软中断向量。其中两个tasklet相关，两个网络相关，两个块处理层相关，两个定时器相关，另外调度器和RCU也各占一个。所有这些都在一个枚举中定义：

```C
enum
{
        HI_SOFTIRQ=0,
        TIMER_SOFTIRQ,
        NET_TX_SOFTIRQ,
        NET_RX_SOFTIRQ,
        BLOCK_SOFTIRQ,
        BLOCK_IOPOLL_SOFTIRQ,
        TASKLET_SOFTIRQ,
        SCHED_SOFTIRQ,
        HRTIMER_SOFTIRQ,
        RCU_SOFTIRQ,
        NR_SOFTIRQS
};
```

以上软中断的名字在如下的数组中定义：

```C
const char * const softirq_to_name[NR_SOFTIRQS] = {
        "HI", "TIMER", "NET_TX", "NET_RX", "BLOCK", "BLOCK_IOPOLL",
        "TASKLET", "SCHED", "HRTIMER", "RCU"
};
```

我们也可以在'/proc/softirqs'的输出中看到他们：

```
~$ cat /proc/softirqs
                    CPU0       CPU1       CPU2       CPU3       CPU4       CPU5       CPU6       CPU7
          HI:          5          0          0          0          0          0          0          0
       TIMER:     332519     310498     289555     272913     282535     279467     282895     270979
      NET_TX:       2320          0          0          2          1          1          0          0
      NET_RX:     270221        225        338        281        311        262        430        265
       BLOCK:     134282         32         40         10         12          7          8          8
BLOCK_IOPOLL:          0          0          0          0          0          0          0          0
     TASKLET:     196835          2          3          0          0          0          0          0
       SCHED:     161852     146745     129539     126064     127998     128014     120243     117391
     HRTIMER:          0          0          0          0          0          0          0          0
         RCU:     337707     289397     251874     239796     254377     254898     267497     256624
```

可以看到`softirq_vec`数组的元素类型为`softirq_action`。这是软中断机制里一个重要的数据结构，它只有一个指向中断处理函数的成员：

```C
struct softirq_action
{
         void    (*action)(struct softirq_action *);
};
```

现在我们可以理解到`open_softirq`函数实际上用`softirq_action`参数填充了`softirq_vec`数组。由`open_softirq`注册的延后中断处理函数会由`raise_softirq`调用。这个函数只有一个参数 -- 软中断序号`nr`。来看下它的实现：

```C
void raise_softirq(unsigned int nr)
{
        unsigned long flags;

        local_irq_save(flags);
        raise_softirq_irqoff(nr);
        local_irq_restore(flags);
}
```

可以看到在`local_irq_save`和`local_irq_restore`两个宏中间调用了`raise_softirq_irqoff`函数。`local_irq_save`的定义位于[include/linux/irqflags.h](https://github.com/torvalds/linux/blob/master/include/linux/irqflags.h)头文件，它保存了[eflags](https://en.wikipedia.org/wiki/FLAGS_register)寄存器中的[IF](https://en.wikipedia.org/wiki/Interrupt_flag)标志位并且禁用了当前处理器的中断。`local_irq_restore`宏定义于同一头文件中，它做了完全相反的事情：装回之前保存的中断标志位然后允许中断。这里之所以要禁用中断是因为`softirq`中断运行于中断上下文中，并且????????????????????

`raise_softirq_irqoff`函数设置当前处理器上和nr参数对应的软中断标志位(`__softirq_pending`)。这是通过以下代码做到的：

```C
__raise_softirq_irqoff(nr);
```

然后，通过`in_interrupt`函数获得`irq_count`值。我们在这一章的第一[小节](https://www.gitbook.com/book/xinqiu/linux-insides-cn/content/interrupts/interrupts-1.html)已经知道它是用来检测一个cpu是否已经有软中断需要处理。如果我们不处于中断上下文中，我们就退出`raise_softirq_irqoff`函数，装回`IF`标志位并允许当前处理器的中断。如果在中断上下文中，就会调用`wakeup_softirqd`函数：

```C
if (!in_interrupt())
	wakeup_softirqd();
```

`wakeup_softirqd`函数会激活当前处理器上的`ksoftirqd`内核线程：

```C
static void wakeup_softirqd(void)
{
	struct task_struct *tsk = __this_cpu_read(ksoftirqd);

    if (tsk && tsk->state != TASK_RUNNING)
        wake_up_process(tsk);
}
```

每个`ksoftirqd`内核线程都运行`run_ksoftirqd`函数来检测是否有延后中断需要处理，如果有的话就会调用`__do_softirq`函数。`__do_softirq`读取当前处理器对应的`__softirq_pending`软中断标记，并调用所有已被标记中断对应的处理函数。在执行一个延后函数的同时，可能会发生新的软中断。这会导致用户态代码由于`__do_softirq`要处理很多延后中断而很长时间不能返回。为了解决这个问题，系统限制了延后中断处理的最大耗时：

```C
unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
...
...
...
restart:
while ((softirq_bit = ffs(pending))) {
	...
	h->action(h);
	...
}
...
...
...
pending = local_softirq_pending();
if (pending) {
	if (time_before(jiffies, end) && !need_resched() &&
		--max_restart)
            goto restart;
}
...
```

除周期性检测是否有延后中断需要执行之外，系统还会在一些关键时间点上检测。一个主要的检测时间点就是当定义在[arch/x86/kernel/irq.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/irq.c)的函数`do_IRQ`被调用时，这是Linux内核中执行延后中断的主要时机。在这个函数要完成中断处理时它会调用[arch/x86/include/asm/apic.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/apic.h)中定义的`exiting_irq`函数，`exiting_irq`又调用了`irq_exit`。`irq_exit`函数会检测当前处理器上下文是否有延后中断，有的话就会调用`invoke_softirq`：

```C
if (!in_interrupt() && local_softirq_pending())
    invoke_softirq();
```

这样就调用到了我们上面提到的`__do_softirq`。每个`softirq`都有如下的阶段：通过`open_softirq`函数注册一个软中断，通过`raise_softirq`函数标记一个延后中断来激活它，然后所有被标记的软中断将会在Linux内核下一次执行周期性延后中断检测时得以调度，对应此类型中断的处理函数也就得以执行。

如上所讲，软中断是静态分配的，但这对于后期加载的内核模块是一个问题。基于软中断的`tasklets`解决了这个问题。

Tasklets
--------------------------------------------------------------------------------



工作队列
--------------------------------------------------------------------------------
