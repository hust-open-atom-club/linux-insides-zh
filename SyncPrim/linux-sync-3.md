
内核同步原语. 第三部分.
================================================================================

信号量
--------------------------------------------------------------------------------

这是本章的第三部分 [chapter](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/index.html)，本章描述了内核中的同步原语,在之前的部分我们见到了特殊的 [自旋锁](https://en.wikipedia.org/wiki/Spinlock) - `排队自旋锁`。 在更前的 [部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/linux-sync-2.html) 是和 `自旋锁` 相关的描述。我们将描述更多同步原语。

在 `自旋锁` 之后的下一个我们将要讲到的 [内核同步原语](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)是 [信号量](https://en.wikipedia.org/wiki/Semaphore_%28programming%29)。我们会从理论角度开始学习什么是 `信号量`， 然后我们会像前几章一样讲到Linux内核是如何实现信号量的。

好吧，现在我们开始。

介绍Linux内核中的信号量
--------------------------------------------------------------------------------

那么究竟什么是 `信号量` ？就像你可以猜到那样 - `信号量` 是另外一种支持线程或者进程的同步机制。Linux内核已经提供了一种同步机制 - `自旋锁`， 为什么我们还需要另外一种呢？为了回答这个问题，我们需要理解这两种机制。我们已经熟悉了 `自旋锁` ,因此我们从 `信号量` 机制开始。

`自旋锁` 的设计理念是它仅会被持有非常短的时间。 但持有自旋锁的时候我们不可以进入睡眠模式因为其他的进程在等待我们。为了防止 [死锁](https://en.wikipedia.org/wiki/Deadlock) [上下文交换](https://en.wikipedia.org/wiki/Context_switch) 也是不允许的。

当需要长时间持有一个锁的时候 [信号量](https://en.wikipedia.org/wiki/Semaphore_%28programming%29) 就是一个很好的解决方案。从另一个方面看，这个机制对于需要短期持有锁的应用并不是最优。为了理解这个问题，我们需要知道什么是 `信号量`。

就像一般的同步原语，`信号量` 是基于变量的。这个变量可以变大或者减少，并且这个变量的状态代表了获取锁的能力。注意这个变量的值并不限于 `0` 和 `1`。有两种类型的 `信号量`：

* `二值信号量`;
* `普通信号量`.

第一种 `信号量` 的值可以为 `1` 或者 `0`。第二种 `信号量` 的值可以为任何非负数。如果 `信号量` 的值大于 `1` 那么它被叫做 `计数信号量`，并且它允许多于 `1` 个进程获取它。这种机制允许我们记录现有的资源，而 `自旋锁` 只允许我们为一个任务上锁。除了所有这些之外，另外一个重要的点是 `信号量` 允许进入睡眠状态。 另外当某进程在等待一个被其他进程获取的锁时， [调度器](https://en.wikipedia.org/wiki/Scheduling_%28computing%29) 也许会切换别的进程。

信号量 API
--------------------------------------------------------------------------------

因此，我们从理论方面了解一些 `信号量`的知识，我们来看看它在Linux内核中是如何实现的。所有 `信号量` 相关的 [API](https://en.wikipedia.org/wiki/Application_programming_interface) 都在名为 [include/linux/semaphore.h](https://github.com/torvalds/linux/blob/master/include/linux/semaphore.h) 的头文件中

我们看到 `信号量` 机制是有以下的结构体表示的：

```C
struct semaphore {
	raw_spinlock_t		lock;
	unsigned int		count;
	struct list_head	wait_list;
};
```

在内核中， `信号量` 结构体由三部分组成：

* `lock` - 保护 `信号量` 的 `自旋锁`;
* `count` - 现有资源的数量;
* `wait_list` - 等待获取此锁的进程序列.

在我们考虑Linux内核的的 `信号量` [API](https://en.wikipedia.org/wiki/Application_programming_interface) 之前，我们需要知道如何初始化一个 `信号量`。事实上， Linux内核提供了两个 `信号量` 的初始函数。这些函数允许初始化一个 `信号量` 为：

* `静态`;
* `动态`.

我们来看看第一个种初始化静态 `信号量`。我们可以使用 `DEFINE_SEMAPHORE` 宏将 `信号量` 静态初始化。

```C
#define DEFINE_SEMAPHORE(name)  \
         struct semaphore name = __SEMAPHORE_INITIALIZER(name, 1)
```

就像我们看到这样，`DEFINE_SEMAPHORE` 宏只提供了初始化 `二值` 信号量。 `DEFINE_SEMAPHORE` 宏展开到 `信号量` 结构体的定义。结构体通过 `__SEMAPHORE_INITIALIZER` 宏初始化。我们来看看这个宏的实现
```C
#define __SEMAPHORE_INITIALIZER(name, n)              \
{                                                                       \
        .lock           = __RAW_SPIN_LOCK_UNLOCKED((name).lock),        \
        .count          = n,                                            \
        .wait_list      = LIST_HEAD_INIT((name).wait_list),             \
}
```

`__SEMAPHORE_INITIALIZER` 宏传入了 `信号量` 结构体的名字并且初始化这个结构体的各个域。首先我们使用 `__RAW_SPIN_LOCK_UNLOCKED` 宏对给予的 `信号量` 初始化一个 `自旋锁`。就像你从 [之前](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/linux-sync-1.html) 的部分看到那样，`__RAW_SPIN_LOCK_UNLOCKED` 宏是在 [include/linux/spinlock_types.h](https://github.com/torvalds/linux/blob/master/include/linux/spinlock_types.h) 头文件中定义，它展开到 `__ARCH_SPIN_LOCK_UNLOCKED` 宏，而 `__ARCH_SPIN_LOCK_UNLOCKED` 宏又展开到零或者无锁状态

```C
#define __ARCH_SPIN_LOCK_UNLOCKED       { { 0 } }
```

 `信号量` 的最后两个域 `count` 和 `wait_list` 是通过现有资源的数量和空 [链表](https://xinqiu.gitbooks.io/linux-insides-cn/content/DataStructures/linux-datastructures-1.html)来初始化。
第二种初始化 `信号量` 的方式是将 `信号量` 和现有资源数目传送给 `sema_init` 函数。 这个函数是在 [include/linux/semaphore.h](https://github.com/torvalds/linux/blob/master/include/linux/semaphore.h) 头文件中定义的。

```C
static inline void sema_init(struct semaphore *sem, int val)
{
       static struct lock_class_key __key;
       *sem = (struct semaphore) __SEMAPHORE_INITIALIZER(*sem, val);
       lockdep_init_map(&sem->lock.dep_map, "semaphore->lock", &__key, 0);
}
```

我们来看看这个函数是如何实现的。它看起来很简单。函数使用我们刚看到的 `__SEMAPHORE_INITIALIZER` 宏对传入的 `信号量` 进行初始化。就像我们在之前 [部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/index.html) 写的那样，我们将会跳过Linux内核关于 [锁验证](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt) 的部分。
从现在开始我们知道如何初始化一个 `信号量`，我们看看如何上锁和解锁。Linux内核提供了如下操作 `信号量` 的 [API](https://en.wikipedia.org/wiki/Application_programming_interface) 

```
void down(struct semaphore *sem);
void up(struct semaphore *sem);
int  down_interruptible(struct semaphore *sem);
int  down_killable(struct semaphore *sem);
int  down_trylock(struct semaphore *sem);
int  down_timeout(struct semaphore *sem, long jiffies);
```

前两个函数： `down` 和 `up` 是用来获取或释放 `信号量`。 `down_interruptible`函数试图去获取一个 `信号量`。如果被成功获取，`信号量` 的计数就会被减少并且锁也会被获取。同时当前任务也会被调度到受阻状态，也就是说 `TASK_INTERRUPTIBLE` 标志将会被至位。`TASK_INTERRUPTIBLE` 表示这个进程也许可以通过 [信号](https://en.wikipedia.org/wiki/Unix_signal) 退回到销毁状态。

`down_killable` 函数和 `down_interruptible` 函数提供类似的功能，但是它还将当前进程的 `TASK_KILLABLE` 标志置位。这表示等待的进程可以被杀死信号中断。

`down_trylock` 函数和 `spin_trylock` 函数相似。这个函数试图去获取一个锁并且退出如果这个操作是失败的。在这个例子中，想获取锁的进程不会等待。最后的 `down_timeout`函数试图去获取一个锁。当前进程将会被中断进入到等待状态当超过传入的可等待时间。除此之外你也许注意到，这个等待的时间是以 [jiffies](https://xinqiu.gitbooks.io/linux-insides-cn/content/Timers/linux-timers-1.html)计数。

我们刚刚看了 `信号量` [API](https://en.wikipedia.org/wiki/Application_programming_interface)的定义。我们从 `down` 函数开始看。这个函数是在 [kernel/locking/semaphore.c](https://github.com/torvalds/linux/blob/master/kernel/locking/semaphore.c) 源代码定义的。我们来看看函数实现：

```C
void down(struct semaphore *sem)
{
        unsigned long flags;

        raw_spin_lock_irqsave(&sem->lock, flags);
        if (likely(sem->count > 0))
                sem->count--;
        else
                __down(sem);
        raw_spin_unlock_irqrestore(&sem->lock, flags);
}
EXPORT_SYMBOL(down);
```

我们先看在 `down` 函数起始处定义的 `flags` 变量。这个变量将会传入到 `raw_spin_lock_irqsave` 和 `raw_spin_lock_irqrestore` 宏定义。这些宏是在 [include/linux/spinlock.h](https://github.com/torvalds/linux/blob/master/include/linux/spinlock.h)头文件定义的。这些宏用来保护当前 `信号量` 的计数器。事实上这两个宏的作用和 `spin_lock` 和 `spin_unlock` 宏相似。只不过这组宏会存储/重置当前中断标志并且禁止 [中断](https://en.wikipedia.org/wiki/Interrupt)。

就像你猜到那样， `down` 函数的主要就是通过 `raw_spin_lock_irqsave` 和 `raw_spin_unlock_irqrestore` 宏来实现的。我们通过将 `信号量` 的计数器和零对比，如果计数器大于零，我们可以减少这个计数器。这表示我们已经获取了这个锁。否则如果计数器是零，这表示所以的现有资源都已经被占用，我们需要等待以获取这个锁。就像我们看到那样， `__down` 函数将会被调用。
 `__down` 函数是在 [相同](https://github.com/torvalds/linux/blob/master/kernel/locking/semaphore.c))的源代码定义的，它的实现看起来如下：
```C
static noinline void __sched __down(struct semaphore *sem)
{
        __down_common(sem, TASK_UNINTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}
```

 `__down` 函数仅仅调用了 `__down_common` 函数，并且传入了三个参数

* `semaphore`;
* `flag` - 对当前任务;
* `timeout` - 最长等待 `信号量` 的时间.

在我们看  `__down_common` 函数之前，注意 `down_trylock`, `down_timeout` 和 `down_killable` 的实现也都是基于 `__down_common` 函数。

```C
static noinline int __sched __down_interruptible(struct semaphore *sem)
{
        return __down_common(sem, TASK_INTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}
```

`__down_killable` 函数：

```C
static noinline int __sched __down_killable(struct semaphore *sem)
{
        return __down_common(sem, TASK_KILLABLE, MAX_SCHEDULE_TIMEOUT);
}
```

`__down_timeout` 函数:

```C
static noinline int __sched __down_timeout(struct semaphore *sem, long timeout)
{
        return __down_common(sem, TASK_UNINTERRUPTIBLE, timeout);
}
```

现在我们来看看 `__down_common` 函数的实现。这个函数是在 [kernel/locking/semaphore.c](https://github.com/torvalds/linux/blob/master/kernel/locking/semaphore.c)源文件中定义的。这个函数的定义从以下两个本地变量开始。

```C
struct task_struct *task = current;
struct semaphore_waiter waiter;
```

第一个变量表示当前想获取本地处理器锁的任务。 `current` 宏是在 [arch/x86/include/asm/current.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/current.h) 头文件中定义的。

```C
#define current get_current()
```

`get_current` 函数返回 `current_task` [per-cpu](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html) 变量的值。


```C
DECLARE_PER_CPU(struct task_struct *, current_task);

static __always_inline struct task_struct *get_current(void)
{
        return this_cpu_read_stable(current_task);
}
```

第二个变量是 `waiter` 表示了一个 `semaphore.wait_list` 列表的入口：

```C
struct semaphore_waiter {
        struct list_head list;
        struct task_struct *task;
        bool up;
};
```

下一步我们将当前进程加入到 `wait_list` 并且在定义如下变量后填充 `waiter` 域

```C
list_add_tail(&waiter.list, &sem->wait_list);
waiter.task = task;
waiter.up = false;
```

下一步我们进入到如下的无限循环：

```C
for (;;) {
        if (signal_pending_state(state, task))
            goto interrupted;

        if (unlikely(timeout <= 0))
            goto timed_out;

        __set_task_state(task, state);

        raw_spin_unlock_irq(&sem->lock);
        timeout = schedule_timeout(timeout);
        raw_spin_lock_irq(&sem->lock);

        if (waiter.up)
            return 0;
}
```

在之前的代码中我们将 `waiter.up` 设置为 `false`。所以当 `up` 没有设置为 `true` 任务将会在这个无限循环中循环。这个循环从检查当前的任务是否处于 `pending` 状态开始，也就是说此任务的标志包含 `TASK_INTERRUPTIBLE` 或者 `TASK_WAKEKILL` 标志。我之前写到当一个任务在等待获取一个信号的时候任务也许可以被 ［信号］(https://en.wikipedia.org/wiki/Unix_signal) 中断。`signal_pending_state` 函数是在 [include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h)原文件中定义的，它看起来如下：

```C
static inline int signal_pending_state(long state, struct task_struct *p)
{
         if (!(state & (TASK_INTERRUPTIBLE | TASK_WAKEKILL)))
                 return 0;
         if (!signal_pending(p))
                 return 0;
 
         return (state & TASK_INTERRUPTIBLE) || __fatal_signal_pending(p);
}
```
我们先会检测 `state` [位掩码](https://en.wikipedia.org/wiki/Mask_%28computing%29) 包含 `TASK_INTERRUPTIBLE` 或者 `TASK_WAKEKILL` 位，如果不包含这两个位，函数退出。下一步我们检测当前任务是否有一个挂起信号，如果没有挂起信号函数退出。最后我们就检测 `state` 位掩码的 `TASK_INTERRUPTIBLE` 位。如果，我们任务包含一个挂起信号，我们将会跳转到 `interrupted` 标签：

```C
interrupted:
    list_del(&waiter.list);
    return -EINTR;
```

在这个标签中，我们会删除等待锁的列表，然后返回 `-EINTR` [错误码](https://en.wikipedia.org/wiki/Errno.h)。 如果一个任务没有挂起信号，我们检测超时是否小于等于零。

```C
if (unlikely(timeout <= 0))
    goto timed_out;
```

我们跳转到 `timed_out` 标签：

```C
timed_out:
    list_del(&waiter.list);
    return -ETIME;
```

在这个标签里，我们继续做和 `interrupted` 一样的事情。我们将任务从锁等待者中删除，但是返回  `-ETIME` 错误码。如果一个任务没有挂起信号而且给予的超时也没有过期，当前的任务将会被设置为传入的 `state`：

```C
__set_task_state(task, state);
```

然后调用 `schedule_timeout` 函数：

```C
raw_spin_unlock_irq(&sem->lock);
timeout = schedule_timeout(timeout);
raw_spin_lock_irq(&sem->lock);
```

这个函数是在 [kernel/time/timer.c](https://github.com/torvalds/linux/blob/master/kernel/time/timer.c) 代码中定义的。`schedule_timeout` 函数将当前的任务置为休眠到设置的超时为止。 

这就是所有关于 `__down_common` 函数。如果一个函数想要获取一个已经被其它任务获取的锁，它将会转入到无限循环。并且它不能被信号中断，当前设置的超时不会过期或者当前持有锁的任务不释放它。现在我们来看看 `up` 函数的实现。

`up` 函数和 `down` 函数定义在[同一个](https://github.com/torvalds/linux/blob/master/kernel/locking/semaphore.c) 原文件。这个函数的主要功能是释放锁，这个函数看起来：

```C
void up(struct semaphore *sem)
{
        unsigned long flags;

        raw_spin_lock_irqsave(&sem->lock, flags);
        if (likely(list_empty(&sem->wait_list)))
                sem->count++;
        else
                __up(sem);
        raw_spin_unlock_irqrestore(&sem->lock, flags);
}
EXPORT_SYMBOL(up);
```

它看起来和 `down` 函数相似。这里有两个不同点。首先我们增加 `semaphore` 的计数。如果等待列表是空的，我们调用在当前原文件中定义的 `__up` 函数。如果等待列表不是空的，我们需要允许列表中的第一个任务去获取一个锁：

```C
static noinline void __sched __up(struct semaphore *sem)
{
        struct semaphore_waiter *waiter = list_first_entry(&sem->wait_list,
                                                struct semaphore_waiter, list);
        list_del(&waiter->list);
        waiter->up = true;
        wake_up_process(waiter->task);
}
```

在此我们获取待序列中的第一个任务，将它从列表中删除，将它的 `waiter-up` 设置为真。从此刻起 `__down_common` 函数中的无限循环将会被停止。 `wake_up_process` 函数将会在 `__up` 函数的结尾调用。我们从 `__down_common` 函数调用的 `schedule_timeout` 函数调用了  `schedule_timeout` 函数。`schedule_timeout` 函数将当前任务置于睡眠状态直到超时等待。现在我们进程也许会睡眠，我们需要唤醒。这就是为什么我们需要从 [kernel/sched/core.c](https://github.com/torvalds/linux/blob/master/kernel/sched/core.c) 源代码中调用 `wake_up_process` 函数

这就是所有的信息了。

小结
--------------------------------------------------------------------------------

这就是Linux内核中关于 [同步原语](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29) 的第三部分的终结。在之前的两个部分，我们已经见到了第一个Linux内核的同步原语 `自旋锁`，它是使用 `ticket spinlock` 实现并且用于很短时间的锁。在这个部分我们见到了另外一种同步原语 － [信号量](https://en.wikipedia.org/wiki/Semaphore_%28programming%29)，信号量用于长时间的锁，因为它会导致 [上下文切换](https://en.wikipedia.org/wiki/Context_switch)。 在下一部分，我们将会继续深入Linux内核的同步原语并且讨论另一个同步原语 － [互斥量](https://en.wikipedia.org/wiki/Mutual_exclusion)。

如果你有问题或者建议，请在twitter [0xAX](https://twitter.com/0xAX)上联系我，通过 [email](anotherworldofworld@gmail.com)联系我，或者创建一个 [issue](https://github.com/MintCN/linux-insides-zh/issues/new)




链接
--------------------------------------------------------------------------------

* [spinlocks](https://en.wikipedia.org/wiki/Spinlock)
* [synchronization primitive](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)
* [semaphore](https://en.wikipedia.org/wiki/Semaphore_%28programming%29)
* [context switch](https://en.wikipedia.org/wiki/Context_switch)
* [preemption](https://en.wikipedia.org/wiki/Preemption_%28computing%29)
* [deadlocks](https://en.wikipedia.org/wiki/Deadlock)
* [scheduler](https://en.wikipedia.org/wiki/Scheduling_%28computing%29)
* [Doubly linked list in the Linux kernel](https://xinqiu.gitbooks.io/linux-insides-cn/content/DataStructures/linux-datastructures-1.html)
* [jiffies](https://xinqiu.gitbooks.io/linux-insides-cn/content/Timers/linux-timers-1.html)
* [interrupts](https://en.wikipedia.org/wiki/Interrupt)
* [per-cpu](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html)
* [bitmask](https://en.wikipedia.org/wiki/Mask_%28computing%29)
* [SIGKILL](https://en.wikipedia.org/wiki/Unix_signal#SIGKILL)
* [errno](https://en.wikipedia.org/wiki/Errno.h)
* [API](https://en.wikipedia.org/wiki/Application_programming_interface)
* [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)
* [Previous part](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/linux-sync-2.html)

