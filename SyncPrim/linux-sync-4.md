内核同步原语. 第四部分.
================================================================================

引言
--------------------------------------------------------------------------------

这是本章的第四部分 [chapter](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/index.html) ，本章描述了内核中的同步原语，且在之前的部分我们介绍完了 [自旋锁](https://en.wikipedia.org/wiki/Spinlock) 和 [信号量](https://en.wikipedia.org/wiki/Semaphore_%28programming%29) 两种不同的同步原语。我们将在这个章节持续学习 [同步原语](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)，并考虑另一简称 [互斥锁 (mutex)](https://en.wikipedia.org/wiki/Mutual_exclusion) 的同步原语，全名 `MUTual EXclusion`。


与 [本书](https://xinqiu.gitbooks.io/linux-insides-cn/content) 前面所有章节一样，我们将先尝试从理论面来探究此同步原语，在此基础上再探究Linux内核所提供用来操作 `互斥锁` 的 [API](https://en.wikipedia.org/wiki/Application_programming_interface)。

让我们开始吧。

`互斥锁` 的概念
--------------------------------------------------------------------------------

从前一 [部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/linux-sync-3.html)，我们已经很熟悉同步原语 [信号量](https://en.wikipedia.org/wiki/Semaphore_%28programming%29)。其表示为：

```C
struct semaphore {
	raw_spinlock_t		lock;
	unsigned int		count;
	struct list_head	wait_list;
};
```

此结构体包含 [锁](https://en.wikipedia.org/wiki/Lock_%28computer_science%29) 的状态和一由等待者所构成的列表。根据 `count` 字段的值，`semaphore` 可以向希望访问该资源的多个进程提供对该资源的访问。[互斥锁](https://en.wikipedia.org/wiki/Mutual_exclusion) 的概念与 [信号量](https://en.wikipedia.org/wiki/Semaphore_%28programming%29) 相似，却有着一些差异。`信号量` 和 `互斥锁` 同步原语的主要差异在于 `互斥锁` 具有更严格的语义。不像 `信号量`，单一时间一 `互斥锁` 只能由一个 [进程](https://en.wikipedia.org/wiki/Process_%28computing%29) 持有，并且只有该 `互斥锁` 的 `持有者` 能够对其进行释放或解锁的动作。另外的差异是 `锁` 的 [API](https://en.wikipedia.org/wiki/Application_programming_interface) 实现，`信号量` 同步原语强置重新调度 (重新调度) 在等待列表中的进程。`互斥锁 API` 的实现则允许避免这种状况，进而减少这种昂贵的 [上下文切换](https://en.wikipedia.org/wiki/Context_switch) 操作。

`互斥锁` 同步原语由如下结构呈现在Linux内核中：

```C
struct mutex {
        atomic_t                count;
        spinlock_t              wait_lock;
        struct list_head        wait_list;
#if defined(CONFIG_DEBUG_MUTEXES) || defined(CONFIG_MUTEX_SPIN_ON_OWNER)
        struct task_struct      *owner;
#endif
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
        struct optimistic_spin_queue osq;
#endif
#ifdef CONFIG_DEBUG_MUTEXES
        void                    *magic;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
        struct lockdep_map      dep_map;
#endif
};
```

此结构被定义在 [include/linux/mutex.h](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h) 头文件并且包含着与 `信号量` 结构体类似的字段。`互斥锁` 结构的第一个字段是 - `count`。这个字段的值代表着该 `互斥锁` 的状态。在 `count` 字段为 `1` 的情况下，说明该 `互斥锁` 处于 `无锁` 状态。当 `count` 字段值处于 `零`，表示该 `互斥锁` 处于 `上锁` 状态。此外，`count` 字段的值可能是 `负` 的，在这种情况下表该 `互斥锁` 处于 `上锁` 状态且可能有其他的等待者。

`互斥锁` 结构的下面两个字段 - `wait_lock` 和 `wait_list` 分别是用来保护 `等待队列` 的 [自旋锁](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h)，以及由某个锁的等待者们所构成的 `等待队列` 列表。你可能注意到了，`互斥锁` 和 `信号量` 结构体的相似处到此结束。剩余的 `互斥锁` 结构体字段，则如同我们所见，取决于Linux内核的不同配置选项。

第一个字段 - `owner` 代表持有该锁的 [进程](https://en.wikipedia.org/wiki/Process_%28computing%29)。正如我们看到的，此字段是否存在于 `mutex` 结构体中取决于 `CONFIG_DEBUG_MUTEXES` 或 `CONFIG_MUTEX_SPIN_ON_OWNER` 的内核配置选项。这个字段和下个 `osq` 字段的主要用途在于支援我们之后会介绍的`乐观自旋 (optimistic spinning)` 功能。最后两个字段 - `magic` 和 `dep_map` 仅被用于 [调试](https://en.wikipedia.org/wiki/Debugging) 模式。`magic` 字段用来存储跟 `互斥锁` 用来调试的相关资讯，而第二个字段 - `lockdep_map` 则在Linux内核的 [锁验证器 (lock validator)](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt) 中被使用。

我们已探究完 `互斥锁` 的结构，现在，我们可以开始思考此同步原语是如何在Linux内核中运作的。你可能会猜，一个想获取锁的进程，在允许的情况下，就直接对 `mutex->count` 字段做递减的操作来试图获取锁。而如果进程希望释放一个锁，则递增该字段的值即可。大致上没错，但也正如你可能猜到的那样，在Linux内核中这可能不会那么简单。

事实上，当某进程试图获取 `互斥锁` 时，三条可能路径的选择策略要根据当前 mutex 的状态而定。

* `fastpath`;
* `midpath`;
* `slowpath`.

第一条路径或 `fastpath` 是最快的，正如你从名称中可以领会到的那样。这种情况下的所有事情都很简单。因 `互斥锁` 还没被任何人获取，其 `mutex` 结构中的`count`字段可以直接被进行递减操作。在释放该 `互斥锁` 的情况下，算法是类似的，进程对该 `互斥锁` 结构中的 `count` 字段进行递增的操作即可。当然，所有的这些操作都必须是 [原子](https://en.wikipedia.org/wiki/Linearizability) 的。

是的，这看起来很简单。但如果一个进程想要获取一个已经被其他进程持有的 `互斥锁` 时，会发生什么事？在这种情况下，控制流程将被转由第二条路径决定 - `midpath`。`midpath` 或 `乐观自旋` 会在锁的持有者仍在运行时，对我们已经熟悉的 [MCS lock](http://www.cs.rochester.edu/~scott/papers/1991_TOCS_synch.pdf) 进行 [循环](https://en.wikipedia.org/wiki/Spinlock) 操作。这条路径只有在没有其他更高优先级的进程准备运行时执行。这条路径被称作 `乐观` 是因为等待的进程不会被睡眠或是调度。这可以避免掉昂贵的 [上下文切换](https://en.wikipedia.org/wiki/Context_switch).

在最后一种情况，当 `fastpath` 和 `midpath` 都不被执行时，就会执行最后一条路径 - `slowpath`。这条路径的行为跟 [信号量](https://en.wikipedia.org/wiki/Semaphore_%28programming%29) 的锁操作相似。如果该锁无法被进程获取，那么此进程将被以如下的结构表示加入 `等待队列`：

```C
struct mutex_waiter {
        struct list_head        list;
        struct task_struct      *task;
#ifdef CONFIG_DEBUG_MUTEXES
        void                    *magic;
#endif
};
```

此结构定义在[include/linux/mutex.h](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h) 头文件，且使用上可能会被睡眠。在研究Linux内核提供用来操作 `互斥锁` 的 [API](https://en.wikipedia.org/wiki/Application_programming_interface) 前，让我们先研究 `mutex_waiter` 结构。如果你有阅读此章节的 [前一部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/linux-sync-3.html)，你可以能会注意到 `mutex_waiter` 结构与 [kernel/locking/semaphore.c](https://github.com/torvalds/linux/blob/master/kernel/locking/semaphore.c) 源代码中的 `semaphore_waiter` 结构相似：

```C
struct semaphore_waiter {
        struct list_head list;
        struct task_struct *task;
        bool up;
};
```

它也包含 `list` 和 `task` 字段，用来表示该互斥锁的等待队列。这之间有一个差异是 `mutex_waiter` 没有 `up` 字段，但却有一个可以根据内核设定`CONFIG_DEBUG_MUTEXES` 选项而存在的 `magic` 字段，它能用来存储一些在调试`互斥锁` 问题时有用的资讯。

现在我们知道了什么是 `互斥锁` 以及他是如何在Linux内核中呈现的。在这种情况下，我们可以开始一窥Linux内核所提供用来操作 `互斥锁` 的 [API](https://en.wikipedia.org/wiki/Application_programming_interface) 。

互斥锁 API
--------------------------------------------------------------------------------

至此，在前面的章节我们已经了解什么是 `互斥锁` 原语，而且看过了Linux内核中用来呈现 `互斥锁` 的 `互斥锁` 结构。现在是时后开始研究用来操弄互斥锁的 [API](https://en.wikipedia.org/wiki/Application_programming_interface) 了。详细的 `互斥锁` API 被记录在 [include/linux/mutex.h](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h) 头文件。一如既往，在考虑如何获取和释放一 `互斥锁` 之前，我们需要知道如何初始化它。

初始化 `互斥锁` 有两种方法，第一种是静态初始化。为此，Linux内核提供以下宏：

```C
#define DEFINE_MUTEX(mutexname) \
        struct mutex mutexname = __MUTEX_INITIALIZER(mutexname)
```

让我们来研究此宏的实现。正如我们所见，`DEFINE_MUTEX` 宏接受新定义的 `互斥锁` 名称，并将其扩展成一个新的 `互斥锁` 解构。此外新 `mutex` 结构由 `__MUTEX_INITIALIZER` 宏初始化。让我们看看 `__MUTEX_INITIALIZER`的实现：

```C
#define __MUTEX_INITIALIZER(lockname)         \
{                                                             \
       .count = ATOMIC_INIT(1),                               \
       .wait_lock = __SPIN_LOCK_UNLOCKED(lockname.wait_lock), \
       .wait_list = LIST_HEAD_INIT(lockname.wait_list)        \
}
```

这个宏被定义在 [相同的](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h) 头文件，并且我们可以了解到它初始化了 `mutex` 解构体中的字段。`count` 被初始化为 `1` ，这代表该互斥锁状态为 `无锁` 。`wait_lock` [自旋锁](https://en.wikipedia.org/wiki/Spinlock) 被初始化为无锁状态，而最后的栏位 `wait_list` 被初始化为空的 [双向列表](https://xinqiu.gitbooks.io/linux-insides-cn/content/DataStructures/linux-datastructures-1.html)。

第二种做法允许我们动态初始化一个 `互斥锁`。为此我们需要调用在 [kernel/locking/mutex.c](https://github.com/torvalds/linux/blob/master/kernel/locking/mutex.c) 源码文件的 `__mutex_init` 函数。事实上，大家很少直接调用 `__mutex_init` 函数。取而代之，我们使用下面的 `mutex_init` :

```C
# define mutex_init(mutex)                \
do {                                                    \
        static struct lock_class_key __key;             \
                                                        \
        __mutex_init((mutex), #mutex, &__key);          \
} while (0)
```

我们可以看到 `mutex_init` 宏定义了 `lock_class_key` 并且调用 `__mutex_init` 函数。让我们看看这个函数的实现:

```C
void
__mutex_init(struct mutex *lock, const char *name, struct lock_class_key *key)
{
        atomic_set(&lock->count, 1);
        spin_lock_init(&lock->wait_lock);
        INIT_LIST_HEAD(&lock->wait_list);
        mutex_clear_owner(lock);
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
        osq_lock_init(&lock->osq);
#endif
        debug_mutex_init(lock, name, key);
}
```

如我们所见，`__mutex_init` 函数接收三个参数:

* `lock` - 互斥锁本身;
* `name` - 调试用的互斥锁名称;
* `key`  -  [锁验证器](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)用的key.

在 `__mutex_init` 函数的一开始，我们可以看到 `互斥锁` 状态被初始化。我们通过 `atomic_set` 函数原子地将给定的变量赋予给定的值来将 `互斥锁` 的状态设为 `无锁`。之后我们可以看到 `自旋锁` 被初始化为解锁状态，这将保护 `互斥锁` 的`等待队列`，之后始化 `互斥锁` 的 `等待队列`。 之后我们清除该 `锁` 的持有者并且通过调用 [include/linux/osq_lock.h](https://github.com/torvalds/linux/blob/master/include/linux/osq_lock.h) 头文件中的 `osq_lock_init` 函数来初始化乐观队列(optimistic queue)。 而此函数也只是将乐观队列的的tail设定为无锁状态:

```C
static inline bool osq_is_locked(struct optimistic_spin_queue *lock)
{
        return atomic_read(&lock->tail) != OSQ_UNLOCKED_VAL;
}
```

在 `__mutex_init` 函数的结尾阶段，我们可以看到它调用了 `debug_mutex_init` 函数，不过就如同我在此 [章节](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/) 前面提到的那样，我们并不会在此章节讨论调试相关的内容。

在 `互斥锁` 结构被初始化后，我们可以继续研究 `互斥锁` 同步原语的 `上锁` 和 `解锁` API。`mutex_lock` 和 `mutex_unlock` 函数的实现位于 [kernel/locking/mutex.c](https://github.com/torvalds/linux/blob/master/kernel/locking/mutex.c) 源码文件。首先，让我们从 `mutex_lock` 的实现开始吧。代码如下:

```C
void __sched mutex_lock(struct mutex *lock)
{
        might_sleep();
        __mutex_fastpath_lock(&lock->count, __mutex_lock_slowpath);
        mutex_set_owner(lock);
}
```

我们可以从 [include/linux/kernel.h](https://github.com/torvalds/linux/blob/master/include/linux/kernel.h) 头文件的 `mutex_lock` 函数开头看到 `might_sleep` 宏被调用。该宏的实现取决于 `CONFIG_DEBUG_ATOMIC_SLEEP` 内核配置选项，如果这个选项被启用，该宏会在 [原子](https://en.wikipedia.org/wiki/Linearizability) 上下文中执行时打印栈追踪。此宏为调试用途上的帮手，除此之外，这个宏什么都没做。

在 `might_sleep` 宏之后，我们可以看到 `__mutex_fastpath_lock` 函数被调用。此函数是体系结构相关的，并且因为我们此书探讨的是 [x86_64](https://en.wikipedia.org/wiki/X86-64) 体系结构， \ `__mutex_fastpath_lock` 的实现部分位于 [arch/x86/include/asm/mutex_64.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/mutex_64.h) 头文件。正如我们从`__mutex_fastpath_lock` 函数名称可以理解的那样，此函数将尝试通过 fast path 试图获取一个锁，或者换句话说此函数试图递减一个给定互斥锁的 `count` 字段。

`__mutex_fastpath_lock` 函数的实现由两部分组成。第一部分是 [内联汇编](https://xinqiu.gitbooks.io/linux-insides-cn/content/Theory/linux-theory-3.html) 。让我们看看:

```C
asm_volatile_goto(LOCK_PREFIX "   decl %0\n"
                              "   jns %l[exit]\n"
                              : : "m" (v->counter)
                              : "memory", "cc"
                              : exit);
```

首先，让我们将注意力放在 `asm_volatile_goto`。这个宏被定义在 [include/linux/compiler-gcc.h](https://github.com/torvalds/linux/blob/master/include/linux/compiler-gcc.h) 头文件，并且它只是扩展成两个内联汇编：

```C
#define asm_volatile_goto(x...) do { asm goto(x); asm (""); } while (0)
```

第一个汇编包含 `goto` 特性，而第二个空的内联汇编是 [屏障 (barrier)](https://en.wikipedia.org/wiki/Memory_barrier)。现在回去看看我们的内联汇编。正如我们所看到的，它从被 `LOCK前缀` 的宏定义开始，该宏仅是扩展 [lock](http://x86.renejeschke.de/html/file_module_x86_id_159.html) 指令：

```C
#define LOCK_PREFIX LOCK_PREFIX_HERE "\n\tlock; "
```

正如我们从前面部分已经知道的那样，该指令允许 [原子地](https://en.wikipedia.org/wiki/Linearizability) 执行被它前缀修饰的指令 。所以，在我们的汇编语句的第一步中，我们尝试将给定的 `mutex->counter` 的字段递减。再下一步，如 `mutex->counter` 在被递减完后的值为非负数，那么 [jns](http://unixwiz.net/techtips/x86-jumps.html) 指令将会执行跳转到 `exit` 标签所在处。`exit` 标签是 `__mutex_fastpath_lock` 函数的第二部分，而它仅仅是指到该函数的出口：

```C
exit:
        return;
```

就目前为止， `__mutex_fastpath_lock` 函数的实现看起来还挺简单。但 `mutex->counter` 的字段有可能在递减后是负的，在这种情况下： 

```C
fail_fn(v);
```

将在内联汇编后被调用。`fail_fn` 是`__mutex_fastpath_lock` 函数的第二个参数，其为一指针指到用来获取给定锁的 `midpath/slowpath` 路径函数。我们的这个例子中`fail_fn` 是 `__mutex_lock_slowpath` 函数。在我们一窥 `__mutex_lock_slowpath` 函数的实现前，我们先把 `mutex_lock` 函数的实现看完。在最简单的情况下，锁会被某进程通过执行 `__mutex_fastpath_lock` 路径后成功被获取。这种情况下，我们仅在 mutex_lock 结尾调用即可。

```C
mutex_set_owner(lock);
```

`mutex_set_owner` 函数被定义在 [kernel/locking/mutex.h](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h) 头文件，这将一个锁的持有者设为当前进程：

```C
static inline void mutex_set_owner(struct mutex *lock)
{
        lock->owner = current;
}
```

另外一种情况，让我们研究一进程因为锁已被其它进程持有，而无法顺利获得的情况，我们已经知道在这种情境下 `__mutex_lock_slowpath` 函数会被调用，让我们开始研究这个函数。此函数被定义在 [kernel/locking/mutex.c](https://github.com/torvalds/linux/blob/master/kernel/locking/mutex.c) 源码文件，并且这个函数由 `container_of` 宏开头，意图通过 `__mutex_fastpath_lock` 给的互斥锁状态变量来获得互斥锁本身：

```C
__visible void __sched
__mutex_lock_slowpath(atomic_t *lock_count)
{
        struct mutex *lock = container_of(lock_count, struct mutex, count);

        __mutex_lock_common(lock, TASK_UNINTERRUPTIBLE, 0,
                            NULL, _RET_IP_, NULL, 0);
}
```

之后将获得的 `互斥锁` 带入函数 `__mutex_lock_common` 进行调用。  `__mutex_lock_common` 函数一开始会关闭 [抢占](https://en.wikipedia.org/wiki/Preemption_%28computing%29) 直到下次的重新调度：

```C
preempt_disable();
```

之后我们进到乐观自旋阶段。如同我们知道的，这个阶段取决于`CONFIG_MUTEX_SPIN_ON_OWNER` 内核配置选项。如该选项被禁用，我们就跳过这个阶段并迈入最后一条路径 - 获取 `互斥锁` 的 `slowpath`：

```C
if (mutex_optimistic_spin(lock, ww_ctx, use_ww_ctx)) {
        preempt_enable();
        return 0;
}
```

首先 `mutex_optimistic_spin` 函数检查我们需不需要被重新调度，或者换句话说，没有其他优先级更高的任务可以运行。 如果这个检查成立我们就将当前 循环者(spinner) 更新至 `MCS` 锁的等待队列。这种情况下互斥锁的获取在某个时间只会由一个循环者获得：

```C
osq_lock(&lock->osq)
```

之后下个步奏，我们在下面的迭代中循环:

```C
while (true) {
    owner = READ_ONCE(lock->owner);

    if (owner && !mutex_spin_on_owner(lock, owner))
        break;

    if (mutex_try_to_acquire(lock)) {
        lock_acquired(&lock->dep_map, ip);

        mutex_set_owner(lock);
        osq_unlock(&lock->osq);
        return true;
    }
}
```

并试图获取该锁。首先我们尝试获取该锁的持有者资讯，如果持有者存在 (在进程已释放互斥锁的情况下可能不存在)，我们就在持有者释放锁前于 `mutex_spin_on_owner` 函数中等待。如果在等待锁持有者的过程中遭遇了高优先级任務，我们就离开循环进入睡眠。在另外一种情况下，该锁被进程释放，那我们就试图通过 `mutex_try_to_acquired` 来获取该锁。如果这个操作顺利完成，我们就为该互斥锁设定新的持有者，将我们自身从`MCS` 等待队列中移除，并从 `mutex_optimistic_spin` 函数中离开。至此锁就被一进程获取完成，我们接着开启 [抢占](https://en.wikipedia.org/wiki/Preemption_%28computing%29) 并从 `__mutex_lock_common` 函数中离开：

```C
if (mutex_optimistic_spin(lock, ww_ctx, use_ww_ctx)) {
    preempt_enable();
    return 0;
}

```

以上就是这种情况的全部了。

在其他情况下，一切可能都不那么顺利。例如，在我们于 `mutex_optimistic_spin` 迭代中循环的同时可能有新任务出现，更甚我们可能在进入 `mutex_optimistic_spin`循环前，系统就存在着更高优先级别的任务了。又或者 `CONFIG_MUTEX_SPIN_ON_OWNER` 内核选项根本是禁用的，在这种情况下 `mutex_optimistic_spin` 什么都不会做：
```C
#ifndef CONFIG_MUTEX_SPIN_ON_OWNER
static bool mutex_optimistic_spin(struct mutex *lock,
                                  struct ww_acquire_ctx *ww_ctx, const bool use_ww_ctx)
{
    return false;
}
#endif
```

在所有这种情况下，`__mutex_lock_common` 函数的行为就像 `信号量`。我们就再次尝试获取锁，因为锁的持有者在此之前可能已经将它释放：

```C
if (!mutex_is_locked(lock) &&
   (atomic_xchg_acquire(&lock->count, 0) == 1))
      goto skip_wait;
```

在失败的情况下，希望获取锁的进程将被添加到等待者列表中

```C
list_add_tail(&waiter.list, &lock->wait_list);
waiter.task = task;
```

在成功的情况下，我们更新该锁的持有者、允许抢占，并从`__mutex_lock_common` 函数离开：

```C
skip_wait:
        mutex_set_owner(lock);
        preempt_enable();
        return 0;
```

在这个情况下锁将被获取。如果到这边都无法获取锁，我们将进入下面的迭代中：

```C
for (;;) {

    if (atomic_read(&lock->count) >= 0 && (atomic_xchg_acquire(&lock->count, -1) == 1))
        break;

    if (unlikely(signal_pending_state(state, task))) {
        ret = -EINTR;
        goto err;
    }

    __set_task_state(task, state);

     schedule_preempt_disabled();
}
```

这边我们再尝试获取一次锁，如果成功的话就离开。是的，我们会在循环前的那次尝试失败后立再一次的尝试获取锁。我们需要这样做，以确保一旦锁在之后被解锁时，我们会被唤醒。 除此之外，它还允许我们在睡眠后获得锁。在另一种情况下，我们检查当前进程是否有pending [信号](https://en.wikipedia.org/wiki/Unix_signal)，如果进程在等待锁获取期间被 `信号` 中断，则退出。在迭代的结尾因为我们未能成功获取锁，所以我们将任务的状态设为 `TASK_UNINTERRUPTIBLE` 并通过调用 `schedule_preempt_disabled` 函数去睡眠。

这就是全部，我们已经考虑了进程获取锁时可能通过的所有三条路径。现在让我们来研究下 `mutex_unlock` 是如何被实现的。将被一个希望释放锁的进程调用 `mutex_unlock`， `__mutex_fastpath_unlock` 也将被从  [arch/x86/include/asm/mutex_64.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/mutex_64.h) 头文件中调用：

```C
void __sched mutex_unlock(struct mutex *lock)
{
    __mutex_fastpath_unlock(&lock->count, __mutex_unlock_slowpath);
}
```

`__mutex_fastpath_unlock` 函数的实现与`__mutex_fastpath_lock` 函数非常相似：

```C
static inline void __mutex_fastpath_unlock(atomic_t *v,
                                           void (*fail_fn)(atomic_t *))
{
       asm_volatile_goto(LOCK_PREFIX "   incl %0\n"
                         "   jg %l[exit]\n"
                         : : "m" (v->counter)
                         : "memory", "cc"
                         : exit);
       fail_fn(v);
exit:
       return;
}
```

事实上, 其中只有一个区别，我们递增 `mutex->count` 字段。这个操作过后锁将呈现 `无锁` 状态。随着 `互斥锁` 释放，如等待队列有条目的话我们必须将其更新。在这种情况下，`fail_fn` 函数会被调用，也就是 `__mutex_unlock_slowpath`。`__mutex_unlock_slowpath` 函数仅是从给定的`mutex->count` 来获取 `mutex` 实例，并调用 `__mutex_unlock_common_slowpath` 函数：

```C
__mutex_unlock_slowpath(atomic_t *lock_count)
{
      struct mutex *lock = container_of(lock_count, struct mutex, count);

      __mutex_unlock_common_slowpath(lock, 1);
}
```

在 `__mutex_unlock_common_slowpath` 函数中，如果等待队列非空，我们将从中获取第一个条目，并唤醒相关的进程：

```C
if (!list_empty(&lock->wait_list)) {
    struct mutex_waiter *waiter =
           list_entry(lock->wait_list.next, struct mutex_waiter, list);
                wake_up_process(waiter->task);
}
```

在此之后，前一个进程将释放互斥锁，并由另一个在等待队列中的进程获取。

这就是全部了. 我们已经研究完两个主要用来操作 `互斥锁` 的 API `mutex_lock` 和 `mutex_unlock`。除此之外，Linux内核还提供以下 API：

* `mutex_lock_interruptible`;
* `mutex_lock_killable`;
* `mutex_trylock`.

以及对应相同前缀的 `unlock` 函数. 我们就不在此解释这些 `API` 了, 因为他们跟 `信号量` 所提供的 `API` 类似。你可以通过阅读 [前一部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/linux-sync-3.html) 来了解更多。

总结
--------------------------------------------------------------------------------

至此我们结束了Linux内核 [同步原语](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29) 章节的第四部分。在这部分我们见到了新的同步原语 - `互斥锁`。就理论上来说，此同步原语与 [信号量](https://en.wikipedia.org/wiki/Semaphore_%28programming%29) 非常相似。事实上，`互斥锁` 代表着二进制信号量。但它的实现与Linux内核中 `信号量` 实现并不同。在下一部分中，我们将继续深入研究Linux内核中的同步原语。

如果你有问题或者建议，请在twitter [0xAX](https://twitter.com/0xAX)上联系我，通过 [email](mailto:anotherworldofworld@gmail.com) 联系我，或者创建一个[issue](https://github.com/MintCN/linux-insides-zh/issues/new)。

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

链接
--------------------------------------------------------------------------------

* [Mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)
* [Spinlock](https://en.wikipedia.org/wiki/Spinlock)
* [Semaphore](https://en.wikipedia.org/wiki/Semaphore_%28programming%29)
* [Synchronization primitives](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)
* [API](https://en.wikipedia.org/wiki/Application_programming_interface)
* [Locking mechanism](https://en.wikipedia.org/wiki/Lock_%28computer_science%29)
* [Context switches](https://en.wikipedia.org/wiki/Context_switch)
* [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)
* [Atomic](https://en.wikipedia.org/wiki/Linearizability)
* [MCS lock](http://www.cs.rochester.edu/~scott/papers/1991_TOCS_synch.pdf)
* [Doubly linked list](https://xinqiu.gitbooks.io/linux-insides-cn/content/DataStructures/linux-datastructures-1.html)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [Inline assembly](https://xinqiu.gitbooks.io/linux-insides-cn/content/Theory/linux-theory-3.html)
* [Memory barrier](https://en.wikipedia.org/wiki/Memory_barrier)
* [Lock instruction](http://x86.renejeschke.de/html/file_module_x86_id_159.html)
* [JNS instruction](http://unixwiz.net/techtips/x86-jumps.html)
* [preemption](https://en.wikipedia.org/wiki/Preemption_%28computing%29)
* [Unix signals](https://en.wikipedia.org/wiki/Unix_signal)
* [Previous part](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/linux-sync-3.html)
