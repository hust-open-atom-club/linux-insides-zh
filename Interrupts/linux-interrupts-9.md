中断和中断处理(九)
================================================================================

延后中断(软中断，Tasklets 和工作队列)介绍
--------------------------------------------------------------------------------

这是 Linux 内核[中断和中断处理](https://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/index.html)的第九节，在[上一节](https://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/linux-interrupts-8.html)我们分析了源文件 [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/irqinit.c) 中的 `init_IRQ` 实现。接下来的这一节我们将继续深入学习外部硬件中断的初始化。

中断处理会有一些特点，其中最主要的两个是：

* 中断处理必须快速执行完毕
* 有时中断处理必须做很多冗长的事情

就像你所想到的，我们几乎不可能同时做到这两点，之前的中断被分为两部分：

* 前半部
* 后半部

`后半部` 曾经是 Linux 内核延后中断执行的一种方式，但现在的实际情况已经不是这样了。现在它已作为一个遗留称谓代表内核中所有延后中断的机制。如你所知，中断处理代码运行于中断处理上下文中，此时禁止响应后续的中断，所以要避免中断处理代码长时间执行。但有些中断却又需要执行很多工作，所以中断处理有时会被分为两部分。第一部分中，中断处理先只做尽量少的重要工作，接下来提交第二部分给内核调度，然后就结束运行。当系统比较空闲并且处理器上下文允许处理中断时，第二部分被延后的剩余任务就会开始执行。

当前实现延后中断的有如下三种途径：

* `软中断`
* `tasklets`
* `工作队列`

在这一小节我们将详细介绍这三种实现，现在是时间深入了解一下了。

软中断
----------------------------------------------------------------------------------

伴随着内核对并行处理的支持，出于性能考虑，所有新的下半部实现方案都基于被称之为 `ksoftirqd` (稍后将详细讨论)的内核线程。每个处理器都有自己的内核线程，名字叫做 `ksoftirqd/n`，n是处理器的编号。我们可以通过系统命令 `systemd-cgls` 看到它们：

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

由 `spawn_ksoftirqd` 函数启动这些线程。就像我们看到的，这个函数在早期的 [initcall](http://www.compsoc.man.ac.uk/~moz/kernelnewbies/documents/initcall/index.html) 被调用。

```C
early_initcall(spawn_ksoftirqd);
```

软中断在 Linux 内核编译时就静态地确定了。`open_softirq` 函数负责 `softirq` 初始化，它在 [kernel/softirq.c](https://github.com/torvalds/linux/blob/master/kernel/softirq.c) 中定义：

```C
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
	softirq_vec[nr].action = action;
}
```

这个函数有两个参数：

* `softirq_vec` 数组的索引序号
* 一个指向软中断处理函数的指针

我们首先来看 `softirq_vec` 数组：

```C
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;
```

它在同一源文件中定义。`softirq_vec` 数组包含了 `NR_SOFTIRQS` (其值为10)个不同 `softirq` 类型的 `softirq_action`。当前版本的 Linux 内核定义了十种软中断向量。其中两个 tasklet 相关，两个网络相关，两个块处理相关，两个定时器相关，另外调度器和 RCU 也各占一个。所有这些都在一个枚举中定义：

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

我们也可以在 `/proc/softirqs` 的输出中看到他们：

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

可以看到 `softirq_vec` 数组的类型为 `softirq_action`。这是软中断机制里一个重要的数据结构，它只有一个指向中断处理函数的成员：

```C
struct softirq_action
{
         void    (*action)(struct softirq_action *);
};
```

现在我们可以理解到 `open_softirq` 函数实际上用 `softirq_action` 参数填充了 `softirq_vec` 数组。由 `open_softirq` 注册的延后中断处理函数会由 `raise_softirq` 调用。这个函数只有一个参数 -- 软中断序号 `nr`。来看下它的实现：

```C
void raise_softirq(unsigned int nr)
{
        unsigned long flags;

        local_irq_save(flags);
        raise_softirq_irqoff(nr);
        local_irq_restore(flags);
}
```

可以看到在 `local_irq_save` 和 `local_irq_restore` 两个宏中间调用了 `raise_softirq_irqoff` 函数。`local_irq_save` 的定义位于 [include/linux/irqflags.h](https://github.com/torvalds/linux/blob/master/include/linux/irqflags.h) 头文件，它保存了 [eflags](https://en.wikipedia.org/wiki/FLAGS_register) 寄存器中的 [IF](https://en.wikipedia.org/wiki/Interrupt_flag) 标志位并且禁用了当前处理器的中断。`local_irq_restore` 宏定义于相同头文件中，它做了完全相反的事情：装回之前保存的中断标志位然后允许中断。这里之所以要禁用中断是因为将要运行的 `softirq` 中断处理运行于中断上下文中。

`raise_softirq_irqoff` 函数设置当前处理器上和nr参数对应的软中断标志位(`__softirq_pending`)。这是通过以下代码做到的：

```C
__raise_softirq_irqoff(nr);
```

然后，通过 `in_interrupt` 函数获得 `irq_count` 值。我们在这一章的第一[小节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/linux-interrupts-1.html)已经知道它是用来检测一个 cpu 是否处于中断环境。如果我们处于中断上下文中，我们就退出 `raise_softirq_irqoff` 函数，装回 `IF` 标志位并允许当前处理器的中断。如果不在中断上下文中，就会调用 `wakeup_softirqd` 函数：

```C
if (!in_interrupt())
	wakeup_softirqd();
```

`wakeup_softirqd` 函数会激活当前处理器上的 `ksoftirqd` 内核线程：

```C
static void wakeup_softirqd(void)
{
	struct task_struct *tsk = __this_cpu_read(ksoftirqd);

    if (tsk && tsk->state != TASK_RUNNING)
        wake_up_process(tsk);
}
```

每个 `ksoftirqd` 内核线程都运行 `run_ksoftirqd` 函数来检测是否有延后中断需要处理，如果有的话就会调用 `__do_softirq` 函数。`__do_softirq` 读取当前处理器对应的 `__softirq_pending` 软中断标记，并调用所有已被标记中断对应的处理函数。在执行一个延后函数的同时，可能会发生新的软中断。这会导致用户态代码由于 `__do_softirq` 要处理很多延后中断而很长时间不能返回。为了解决这个问题，系统限制了延后中断处理的最大耗时：

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

除周期性检测是否有延后中断需要执行之外，系统还会在一些关键时间点上检测。一个主要的检测时间点就是当定义在 [arch/x86/kernel/irq.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/irq.c) 的 `do_IRQ` 函数被调用时，这是 Linux 内核中执行延后中断的主要时机。在这个函数将要完成中断处理时它会调用 [arch/x86/include/asm/apic.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/apic.h) 中定义的 `exiting_irq` 函数，`exiting_irq` 又调用了 `irq_exit`。`irq_exit` 函数会检测当前处理器上下文是否有延后中断，有的话就会调用 `invoke_softirq`：

```C
if (!in_interrupt() && local_softirq_pending())
    invoke_softirq();
```

这样就调用到了我们上面提到的 `__do_softirq`。每个 `softirq` 都有如下的阶段：通过 `open_softirq` 函数注册一个软中断，通过 `raise_softirq` 函数标记一个软中断来激活它，然后所有被标记的软中断将会在 Linux 内核下一次执行周期性软中断检测时得以调度，对应此类型软中断的处理函数也就得以执行。

从上述可看出，软中断是静态分配的，这对于后期加载的内核模块将是一个问题。基于软中断实现的 `tasklets` 解决了这个问题。

Tasklets
--------------------------------------------------------------------------------

如果你阅读 Linux 内核源码中软中断相关的代码，你会发现它很少会被用到。内核中实现延后中断的主要途径是 `tasklets`。正如上面说的，`tasklets` 构建于 `softirq` 中断之上，他是基于下面两个软中断实现的：

* `TASKLET_SOFTIRQ`;
* `HI_SOFTIRQ`.

简而言之，`tasklets` 是运行时分配和初始化的软中断。和软中断不同的是，同一类型的 `tasklets` 不能同时运行在多个处理器上。我们已经了解到一些关于软中断的知识，当然上面的文字并不能详细讲解所有的细节，但我们现在可以通过直接阅读代码一步步的更深入了解软中断。我们返回到开始部分讨论的 `softirq_init` 函数实现，这个函数在 [kernel/softirq.c](https://github.com/torvalds/linux/blob/master/kernel/softirq.c) 中定义如下：

```C
void __init softirq_init(void)
{
        int cpu;

        for_each_possible_cpu(cpu) {
                per_cpu(tasklet_vec, cpu).tail =
                        &per_cpu(tasklet_vec, cpu).head;
                per_cpu(tasklet_hi_vec, cpu).tail =
                        &per_cpu(tasklet_hi_vec, cpu).head;
        }

        open_softirq(TASKLET_SOFTIRQ, tasklet_action);
        open_softirq(HI_SOFTIRQ, tasklet_hi_action);
}
```

可以看到在函数开头定义了一个名为 cpu 的 integer 类型变量。接下来他会作为参数传递给宏 `for_each_possible_cpu` 来获得系统中所有的处理器。如果 `possible_cpu` 对你来说是一个新的术语，你可以阅读 [CPU masks](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-2.html) 章节来了解更多知识。简单的说，`possible_cpu` 是系统运行期间插入的处理器集合。所有的 `possible processor` 存储在 `cpu_possible_bits` 位图中，你可以在 [kernel/cpu.c](https://github.com/torvalds/linux/blob/master/kernel/cpu.c) 中找到他的定义：

```C
static DECLARE_BITMAP(cpu_possible_bits, CONFIG_NR_CPUS) __read_mostly;
...
...
...
const struct cpumask *const cpu_possible_mask = to_cpumask(cpu_possible_bits);
```

好了，我们定义了 integer 类型变量 `cpu` 并且通过 `for_each_possible_cpu` 宏遍历了所有处理器，初始化了两个 `per-cpu` 变量：

* `tasklet_vec`;
* `tasklet_hi_vec`;

这两个 `per-cpu` 变量和 `softirq_init` 函数都定义在相同[代码](https://github.com/torvalds/linux/blob/master/kernel/softirq.c)中，他们被定义为 `tasklet_head` 类型：

```C
static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);
```

`tasklet_head` 结构代表一组 `Tasklets`，它包含两个成员，head 和 tail：

```C
struct tasklet_head {
        struct tasklet_struct *head;
        struct tasklet_struct **tail;
};
```

`tasklet_struct` 数据类型在 [include/linux/interrupt.h](https://github.com/torvalds/linux/blob/master/include/linux/interrupt.h) 中定义，它代表一个 `Tasklet`。这本书之前部分我们没有见过这个单词，那我们先试着理解一下 `Tasklet` 究竟为何物。实际上，`Tasklet` 是处理延后中断的一种机制，来看一下 `tasklet_struct` 的具体定义：

```C
struct tasklet_struct
{
        struct tasklet_struct *next;
        unsigned long state;
        atomic_t count;
        void (*func)(unsigned long);
        unsigned long data;
};
```

这个数据结构包含有下面5个成员：

* 调度队列中的下一个 `Tasklet`
* 当前这个 `Tasklet` 的状态
* 这个 `Tasklet` 是否处于活动状态
* `Tasklet` 的回调函数
* 回调函数的参数

上面代码中，在 `softirq_init` 函数中初始化了两个 tasklets 数组：`tasklet_vec` 和 `tasklet_hi_vec`。Tasklets 和高优先级 Tasklets 分别存储于这两个数组中。初始化完成后我们看到代码 [kernel/softirq.c](https://github.com/torvalds/linux/blob/master/kernel/softirq.c) 在 `softirq_init` 函数的最后又两次调用了 `open_softirq`：

```C
open_softirq(TASKLET_SOFTIRQ, tasklet_action);
open_softirq(HI_SOFTIRQ, tasklet_hi_action);
```

`open_softirq` 函数的主要作用是初始化软中断，接下来让我们看看它是怎么做的。和 Tasklets 相关的软中断处理函数有两个，分别是 `tasklet_action` 和 `tasklet_hi_action`。其中 `tasklet_hi_action` 和 `HI_SOFTIRQ` 关联在一起，`tasklet_action` 和 `TASKLET_SOFTIRQ` 关联在一起。

Linux 内核提供一些 API 供操作 Tasklets 之用。首先是 `tasklet_init` 函数，它接受一个 `task_struct` 数据结构，一个处理函数，和另外一个参数，并利用这些参数来初始化所给的 `task_struct` 结构：

```C
void tasklet_init(struct tasklet_struct *t,
                  void (*func)(unsigned long), unsigned long data)
{
    t->next = NULL;
    t->state = 0;
    atomic_set(&t->count, 0);
    t->func = func;
    t->data = data;
}
```

另外还有如下两个宏可以静态地初始化一个 tasklet：

```C
DECLARE_TASKLET(name, func, data);
DECLARE_TASKLET_DISABLED(name, func, data);
```

Linux 内核提供三个函数标记一个 tasklet 已经准备就绪：

```C
void tasklet_schedule(struct tasklet_struct *t);
void tasklet_hi_schedule(struct tasklet_struct *t);
void tasklet_hi_schedule_first(struct tasklet_struct *t);
```

第一个函数使用普通优先级调度一个 tasklet，第二个使用高优先级，第三个则用更高优先级。所有这三个函数的实现都很类似，所以我们只看一下第一个 `tasklet_schedule` 的实现：

```C
static inline void tasklet_schedule(struct tasklet_struct *t)
{
    if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))
        __tasklet_schedule(t);
}

void __tasklet_schedule(struct tasklet_struct *t)
{
        unsigned long flags;

        local_irq_save(flags);
        t->next = NULL;
        *__this_cpu_read(tasklet_vec.tail) = t;
        __this_cpu_write(tasklet_vec.tail, &(t->next));
        raise_softirq_irqoff(TASKLET_SOFTIRQ);
        local_irq_restore(flags);
}
```

我们看到它检测并设置所给的 tasklet 为 `TASKLET_STATE_SCHED` 状态，然后以所给 tasklet 为参数执行了 `__tasklet_schedule` 函数。`__tasklet_schedule` 看起来和前面见到的 `raise_softirq` 很像。一开始它保存中断标志并禁用中断，继而将新的 tasklet 添加到 `tasklet_vec`，然后调用了我们前面见过的 `raise_softirq_irqoff` 函数。当 Linux 内核调度器决定去运行一个延后函数，`tasklet_action` 函数会被作为和 `TASKLET_SOFTIRQ` 相关联的延后函数调用。同样的，`tasklet_hi_action` 会被作为和 `HI_SOFTIRQ` 相关联的延后函数调用。这些函数之所以如此相似是因为他们之间只有一个地方不同 ---  `tasklet_action` 使用 `tasklet_vec` 而 `tasklet_hi_action` 使用 `tasklet_hi_vec`。

让我们看下 `tasklet_action` 函数的实现：

```C
static void tasklet_action(struct softirq_action *a)
{
    local_irq_disable();
    list = __this_cpu_read(tasklet_vec.head);
    __this_cpu_write(tasklet_vec.head, NULL);
    __this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head));
    local_irq_enable();

    while (list) {
		if (tasklet_trylock(t)) {
	        t->func(t->data);
            tasklet_unlock(t);
	    }
		...
		...
		...
    }
}
```

在 `tasklet_action` 开始时利用 `local_irq_disable` 宏禁用了当前处理器的中断(你可以阅读本书[第二部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/linux-interrupts-2.html)了解更多关于此宏的信息)。接下来获取到当前处理器对应的普通优先级 tasklet 列表并把它设置为 `NULL` ，这是因为所有的 tasklet 都将被执行。然后使能当前处理器的中断，循环遍历 tasklet 列表，每一次遍历都会对当前 tasklet 调用 `tasklet_trylock` 函数来更新它的状态为 `TASKLET_STATE_RUN`：

```C
static inline int tasklet_trylock(struct tasklet_struct *t)
{
    return !test_and_set_bit(TASKLET_STATE_RUN, &(t)->state);
}
```

如果这个操作成功了就会执行此 tasklet 的处理函数(我们在 `tasklet_init` 中所设置的)，然后调用 `tasklet_unlock` 函数清除他的 `TASKLET_STATE_RUN` 状态。

通常情况下，这就是 `tasklet` 的所有概念。当然这些还不足以覆盖所有的 `tasklets`，但是我想大家可以以此为切入点继续学习下去。

`tasklets` 在 Linux 内核中是一个[广泛](http://lxr.free-electrons.com/ident?i=tasklet_init)使用的概念，但就像我在本章开头所写的，还有第三个延后中断机制 -- 工作队列。接下来我们将会看看它又是怎样一种机制。


工作队列
--------------------------------------------------------------------------------

`工作队列`是另外一个处理延后函数的概念，它大体上和 `tasklets` 类似。工作队列运行于内核进程上下文，而 `tasklets` 运行于软中断上下文。这意味着`工作队列`函数不必像 `tasklets` 一样必须是原子性的。Tasklets 总是运行于它提交自的那个处理器，工作队列在默认情况下也是这样。`工作队列`在 Linux 内核代码 [kernel/workqueue.c](https://github.com/torvalds/linux/blob/master/kernel/workqueue.c) 中由如下的数据结构表示：

```C
struct worker_pool {
    spinlock_t              lock;
    int                     cpu;
    int                     node;
    int                     id;
    unsigned int            flags;

    struct list_head        worklist;
    int                     nr_workers;
...
...
...
```

因为这个结构有非常多的成员，这里就不把它们全部罗列出来，下面只讨论上面列出的这几个。

工作队列最基础的用法，是作为创建内核线程的接口来处理提交到队列里的工作任务。所有这些内核线程称之为 `worker thread`。工作队列内的任务是由代码 [include/linux/workqueue.h](https://github.com/torvalds/linux/blob/master/include/linux/workqueue.h) 中定义的 `work_struct` 表示的，起定义如下：

```C
struct work_struct {
    atomic_long_t data;
    struct list_head entry;
    work_func_t func;
#ifdef CONFIG_LOCKDEP
    struct lockdep_map lockdep_map;
#endif
};
```

这里有两个字段比较有意思：`func` --将被`工作队列`调度执行的函数，`data` --这个函数的参数。Linux 内核提供了称之为 `kworker` 的特定于每个 cpu 的内核线程：

```
systemd-cgls -k | grep kworker
├─    5 [kworker/0:0H]
├─   15 [kworker/1:0H]
├─   20 [kworker/2:0H]
├─   25 [kworker/3:0H]
├─   30 [kworker/4:0H]
...
...
...
```

这些线程会被用来调度执行工作队列的延后函数(就像 `ksoftirqd` 之于`软中断`)。除此之外我们还可以为一个`工作队列`创建一个新的工作线程。Linux 内核提供了如下宏静态创建一个队列任务：

```C
#define DECLARE_WORK(n, f) \
    struct work_struct n = __WORK_INITIALIZER(n, f)
```

它需要两个参数：工作队列的名字和工作队列的函数。我们还可以在运行时动态创建：

```C
#define INIT_WORK(_work, _func)       \
    __INIT_WORK((_work), (_func), 0)

#define __INIT_WORK(_work, _func, _onstack)                     \
    do {                                                        \
            __init_work((_work), _onstack);                     \
            (_work)->data = (atomic_long_t) WORK_DATA_INIT();   \
            INIT_LIST_HEAD(&(_work)->entry);                    \
             (_work)->func = (_func);                           \
    } while (0)
```

这个宏需要一个 `work_struct` 数据结构作为将要创建的队列任务，和一个将在这个任务里调度运行的函数。通过这两个宏的其中一个创建一个 `work` 后，我们需要把它放到`工作队列`中去。可以通过 `queue_work` 或者 `queue_delayed_work` 来做到这一点：

```C
static inline bool queue_work(struct workqueue_struct *wq,
                              struct work_struct *work)
{
    return queue_work_on(WORK_CPU_UNBOUND, wq, work);
}
```

`queue_work` 只是调用了 `queue_work_on` 函数指定相应的处理器。注意这里给 `queue_work_on` 函数传递了 `WORK_CPU_UNBOUND` 参数，它作为代表队列任务要绑定到哪一个处理器的枚举一员，定义于 [include/linux/workqueue.h](https://github.com/torvalds/linux/blob/master/include/linux/workqueue.h)。`queue_work_on` 函数测试并设置所给`任务`的 `WORK_STRUCT_PENDING_BIT` 标志位，然后以所给的工作队列和队列任务为参数执行 `__queue_work` 函数：

```C
bool queue_work_on(int cpu, struct workqueue_struct *wq,
           struct work_struct *work)
{
    bool ret = false;
    ...
    if (!test_and_set_bit(WORK_STRUCT_PENDING_BIT, work_data_bits(work))) {
        __queue_work(cpu, wq, work);
        ret = true;
    }
    ...
    return ret;
}
```

`__queue_work` 函数得到参数 `work poll`。是的，是 `work poll` 而不是 `workqueue`。实际上，所有的 `works` 都没有放在 `workqueue` 中，而是放在 Linux 内核中由 `worker_pool` 数据结构所定义的 `work poll`。如上所述，`workqueue_struct` 数据结构的 `pwqs` 成员是一个 `worker_pool` 列表。当我们创建一个 `workqueue`，他针对每一个处理器都创建了 `worker_pool`。每一个和 `worker_pool` 相关联的 `pool_workqueue` 都分配在相同的处理器上对应的优先级队列，`workqueue` 通过他们和 `worker_pool` 交互。在 `__queue_work` 函数里使用 `raw_smp_processor_id` 设置 cpu 为当前处理器在[第四章](https://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-4.html)你可以找到更多相关信息)，得到与所给 `work_struct` 对应的 `pool_workqueue` 并将 `work` 插入到 `workqueue`：

```C
static void __queue_work(int cpu, struct workqueue_struct *wq,
                         struct work_struct *work)
{
...
...
...
if (req_cpu == WORK_CPU_UNBOUND)
    cpu = raw_smp_processor_id();

if (!(wq->flags & WQ_UNBOUND))
    pwq = per_cpu_ptr(wq->cpu_pwqs, cpu);
else
    pwq = unbound_pwq_by_node(wq, cpu_to_node(cpu));
...
...
...
insert_work(pwq, work, worklist, work_flags);
```

现在我们可以创建 `works` 和 `workqueue`，我们需要知道他们究竟会在何时被执行。就像前面提到的，所有的 `works` 都会在内核线程中执行。当内核线程得到调度，它开始执行 `workqueue` 中的 `works`。每一个工作队列内核线程都会在 `worker_thread` 函数里执行一个循环。这些内核线程会做很多不同的事情，其中一些和本章前面提到的很类似。当开始执行时，所有的 `work_struct` 和 `works` 都会从他的 `workqueue` 移除。


总结
--------------------------------------------------------------------------------

现在结束了[中断和中断处理](https://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/index.html)的第九节。这一节中我们继续讨论了外部硬件中断。在之前部分我们看到了 `IRQs` 的初始化和 `irq_desc` 数据结构，在这一节我们看到了用于延后函数的三个概念：`软中断`，`tasklet` 和`工作队列`。

下一节将是 `中断和中断处理` 的最后一节。我们将会了解真正的硬件驱动，并试着学习它是怎样和中断子系统一起工作的。

如果你有任何问题或建议，请给我发评论或者给我发 [Twitter](https://twitter.com/0xAX)。

**请注意英语并不是我的母语，我为任何表达不清楚的地方感到抱歉。如果你发现任何错误请发 PR 到 [linux-insides](https://github.com/MintCN/linux-insides-zh)。(译者注：翻译问题请发 PR 到 [linux-insides-cn](https://www.gitbook.com/book/xinqiu/linux-insides-cn))**


链接
--------------------------------------------------------------------------------

* [initcall](http://www.compsoc.man.ac.uk/~moz/kernelnewbies/documents/initcall/index.html)
* [IF](https://en.wikipedia.org/wiki/Interrupt_flag)
* [eflags](https://en.wikipedia.org/wiki/FLAGS_register)
* [CPU masks](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-2.html)
* [per-cpu](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html)
* [Workqueue](https://github.com/torvalds/linux/blob/master/Documentation/workqueue.txt)
* [Previous part](http://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/linux-interrupts-8.html)
