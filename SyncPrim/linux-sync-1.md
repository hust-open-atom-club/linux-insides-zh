Linux 内核中的同步原语. 第一部分.
================================================================================

Introduction
--------------------------------------------------------------------------------
这一部分为 [linux-insides](https://xinqiu.gitbooks.io/linux-insides-cn/content/) 这本书开启了新的章节。定时器和时间管理相关的概念在上一个[章节](https://xinqiu.gitbooks.io/linux-insides-cn/content/Timers/index.html)已经描述过了。现在是时候继续了。就像你可能从这一部分的标题所了解的那样，本章节将会描述 Linux 内核中的[同步](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)原语。

像往常一样，在考虑一些同步相关的事情之前，我们会尝试去概括地了解什么是`同步原语`。事实上，同步原语是一种软件机制，提供了两个或者多个[并行](https://en.wikipedia.org/wiki/Parallel_computing)进程或者线程在不同时刻执行一段相同的代码段的能力。例如下面的代码片段：

```C
mutex_lock(&clocksource_mutex);
...
...
...
clocksource_enqueue(cs);
clocksource_enqueue_watchdog(cs);
clocksource_select();
...
...
...
mutex_unlock(&clocksource_mutex);
```
出自 [kernel/time/clocksource.c](https://github.com/torvalds/linux/master/kernel/time/clocksource.c) 源文件。这段代码来自于 `__clocksource_register_scale` 函数，此函数添加给定的 [clocksource](https://xinqiu.gitbooks.io/linux-insides-cn/content/Timers/linux-timers-2.html) 到时钟源列表中。这个函数在注册时钟源列表中生成两个不同的操作。例如 `clocksource_enqueue` 函数就是添加给定时钟源到注册时钟源列表——`clocksource_list` 中。注意这几行代码被两个函数所包围：`mutex_lock` 和 `mutex_unlock`，这两个函数都带有一个参数——在本例中为 `clocksource_mutex`。

这些函数展示了基于[互斥锁 (mutex)](https://en.wikipedia.org/wiki/Mutual_exclusion) 同步原语的加锁和解锁。当 `mutex_lock` 被执行，允许我们阻止两个或两个以上线程执行这段代码，而 `mute_unlock` 还没有被互斥锁的处理拥有者锁执行。换句话说，就是阻止在 `clocksource_list`上的并行操作。为什么在这里需要使用`互斥锁`？ 如果两个并行处理尝试去注册一个时钟源会怎样。正如我们已经知道的那样，其中具有最大的等级（其具有最高的频率在系统中注册的时钟源）的列表中选择一个时钟源后，`clocksource_enqueue` 函数立即将一个给定的时钟源到 `clocksource_list` 列表：

```C
static void clocksource_enqueue(struct clocksource *cs)
{
	struct list_head *entry = &clocksource_list;
	struct clocksource *tmp;

	list_for_each_entry(tmp, &clocksource_list, list)
		if (tmp->rating >= cs->rating)
			entry = &tmp->list;
	list_add(&cs->list, entry);
}
```

如果两个并行处理尝试同时去执行这个函数，那么这两个处理可能会找到相同的 `入口 (entry)` 可能发生[竞态条件 (race condition)](https://en.wikipedia.org/wiki/Race_condition) 或者换句话说，第二个执行 `list_add` 的处理程序，将会重写第一个线程写入的时钟源。

除了这个简答的例子，同步原语在 Linux 内核无处不在。如果再翻阅之前的[章节] (https://xinqiu.gitbooks.io/linux-insides-cn/content/Timers/index.html) 或者其他章节或者如果大概看看 Linux 内核源码，就会发现许多地方都使用同步原语。我们不考虑 `mutex` 在 Linux 内核是如何实现的。事实上，Linux 内核提供了一系列不同的同步原语：

* `mutex`;
* `semaphores`;
* `seqlocks`;
* `atomic operations`;
* 等等。

现在从`自旋锁 (spinlock)` 这个章节开始。

Linux 内核中的自旋锁。
--------------------------------------------------------------------------------

自旋锁简单来说是一种低级的同步机制，表示了一个变量可能的两个状态：

* `acquired`;
* `released`.

每一个想要获取`自旋锁`的处理，必须为这个变量写入一个表示`自旋锁获取 (spinlock acquire)`状态的值，并且为这个变量写入`锁释放 (spinlock released)`状态。如果一个处理程序尝试执行受`自旋锁`保护的代码，那么代码将会被锁住，直到占有锁的处理程序释放掉。在本例中，所有相关的操作必须是
[原子的 (atomic)](https://en.wikipedia.org/wiki/Linearizability)，来阻止[竞态条件](https://en.wikipedia.org/wiki/Race_condition)状态。`自旋锁`在 Linux 内核中使用 `spinlock_t` 类型来表示。如果我们查看 Linux 内核代码，我们会看到，这个类型被[广泛地 (widely)](http://lxr.free-electrons.com/ident?i=spinlock_t) 使用。`spinlock_t` 的定义如下：

```C
typedef struct spinlock {
        union {
              struct raw_spinlock rlock;

#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define LOCK_PADSIZE (offsetof(struct raw_spinlock, dep_map))
                struct {
                        u8 __padding[LOCK_PADSIZE];
                        struct lockdep_map dep_map;
                };
#endif
        };
} spinlock_t;
```

这段代码在 [include/linux/spinlock_types.h](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h) 头文件中定义。可以看出，它的实现依赖于 `CONFIG_DEBUG_LOCK_ALLOC` 内核配置选项这个状态。现在我们跳过这一块，因为所有的调试相关的事情都将会在这一部分的最后。所以，如果 `CONFIG_DEBUG_LOCK_ALLOC` 内核配置选项不可用，那么 `spinlock_t` 则包含[联合体 (union)](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B)，这个联合体有一个字段——`raw_spinlock`：

```C
typedef struct spinlock {
        union {
              struct raw_spinlock rlock;
        };
} spinlock_t;
```
`raw_spinlock` 结构的定义在[相同的](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h)头文件中并且表达了`普通 (normal)` 自旋锁的实现。让我们看看 `raw_spinlock`结构是如何定义的：

```C
typedef struct raw_spinlock {
        arch_spinlock_t raw_lock;
#ifdef CONFIG_GENERIC_LOCKBREAK
        unsigned int break_lock;
#endif
} raw_spinlock_t;
```

这里的 `arch_spinlock_t` 表示了体系结构指定的`自旋锁`实现并且 `break_lock` 字段持有值—— 为`1`，当一个处理器开始等待而锁被另一个处理器持有时，使用的[对称多处理器 (SMP)](https://en.wikipedia.org/wiki/Symmetric_multiprocessing) 系统的例子中。这样就可以防止长时间加锁。考虑本书的 [x86_64](https://en.wikipedia.org/wiki/X86-64) 架构，因此 `arch_spinlock_t` 被定义在 [arch/x86/include/asm/spinlock_types.h](https://github.com/torvalds/linux/master/arch/x86/include/asm/spinlock_types.h) 头文件中，并且看上去是这样：

```C
#ifdef CONFIG_QUEUED_SPINLOCKS
#include <asm-generic/qspinlock_types.h>
#else
typedef struct arch_spinlock {
        union {
                __ticketpair_t head_tail;
                struct __raw_tickets {
                        __ticket_t head, tail;
                } tickets;
        };
} arch_spinlock_t;
```

正如我们所看到的，`arch_spinlock` 结构的定义依赖于 `CONFIG_QUEUED_SPINLOCKS` 内核配置选项的值。这个 Linux内核配置选项支持使用队列的 `自旋锁`。这个`自旋锁`的特殊类型替代了 `acquired` 和 `released` [原子](https://en.wikipedia.org/wiki/Linearizability)值，在`队列`上使用`原子`操作。如果 `CONFIG_QUEUED_SPINLOCKS` 内核配置选项启动，那么 `arch_spinlock_t` 将会被表示成如下的结构：

```C
typedef struct qspinlock {
	atomic_t	val;
} arch_spinlock_t;
```

来自于 [include/asm-generic/qspinlock_types.h](https://github.com/torvalds/linux/master/include/asm-generic/qspinlock_types.h) 头文件。

目前我们不会在这个结构上停止探索，在考虑 `arch_spinlock` 和 `qspinlock` 之前，先看看自旋锁上的操作。 Linux内核在`自旋锁`上提供了一下主要的操作：

* `spin_lock_init` ——给定的`自旋锁`进行初始化；
* `spin_lock` ——获取给定的`自旋锁`；
* `spin_lock_bh` ——禁止软件[中断](https://en.wikipedia.org/wiki/Interrupt)并且获取给定的`自旋锁`。
* `spin_lock_irqsave` 和 `spin_lock_irq`——禁止本地处理器上的中断，并且保存／不保存之前的中断状态的`标识 (flag)`；
* `spin_unlock` ——释放给定的`自旋锁`;
* `spin_unlock_bh` ——释放给定的`自旋锁`并且启动软件中断；
* `spin_is_locked` - 返回给定的`自旋锁`的状态；
* 等等。

来看看 `spin_lock_init` 宏的实现。就如我已经写过的一样，这个宏和其他宏定义都在 [include/linux/spinlock.h](https://github.com/torvalds/linux/master/include/linux/spinlock.h) 头文件里，并且 `spin_lock_init` 宏如下所示：

```C
#define spin_lock_init(_lock)		\
do {							                \
	spinlock_check(_lock);				        \
	raw_spin_lock_init(&(_lock)->rlock);		\
} while (0)
```

正如所看到的，`spin_lock_init` 宏有一个`自旋锁`，执行两步操作：检查我们看到的给定的`自旋锁`和执行 `raw_spin_lock_init`。`spinlock_check`的实现相当简单，实现的函数仅仅返回已知的`自旋锁`的 `raw_spinlock_t`，来确保我们精确获得`正常 (normal)` 原生自旋锁：

```C
static __always_inline raw_spinlock_t *spinlock_check(spinlock_t *lock)
{
	return &lock->rlock;
}
```

`raw_spin_lock_init` 宏:

```C
# define raw_spin_lock_init(lock)		\
do {                                                  \
    *(lock) = __RAW_SPIN_LOCK_UNLOCKED(lock);         \
} while (0)                                           \
```

用 `__RAW_SPIN_LOCK_UNLOCKED` 的值和给定的`自旋锁`赋值给给定的 `raw_spinlock_t`。就像我们能从 `__RAW_SPIN_LOCK_UNLOCKED` 宏的名字中了解的那样，这个宏为给定的`自旋锁`执行初始化操作，并且将锁设置为`释放 (released)` 状态。宏的定义在 [include/linux/spinlock_types.h](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h) 头文件中，并且扩展了一下的宏：

```C
#define __RAW_SPIN_LOCK_UNLOCKED(lockname)      \
         (raw_spinlock_t) __RAW_SPIN_LOCK_INITIALIZER(lockname)

#define __RAW_SPIN_LOCK_INITIALIZER(lockname)   \
         {                                                      \
             .raw_lock = __ARCH_SPIN_LOCK_UNLOCKED,             \
             SPIN_DEBUG_INIT(lockname)                          \
             SPIN_DEP_MAP_INIT(lockname)                        \
         }
```

正如之前所写的一样，我们不考虑同步原语调试相关的东西。在本例中也不考虑 `SPIN_DEBUG_INIT` 和 `SPIN_DEP_MAP_INIT` 宏。于是 `__RAW_SPINLOCK_UNLOCKED` 宏被扩展成：

```C
*(&(_lock)->rlock) = __ARCH_SPIN_LOCK_UNLOCKED;
```

而 `__ARCH_SPIN_LOCK_UNLOCKED` 宏是:

```C
#define __ARCH_SPIN_LOCK_UNLOCKED       { { 0 } }
```

还有:

```C
#define __ARCH_SPIN_LOCK_UNLOCKED       { ATOMIC_INIT(0) }
```

这是对于 [x86_64] 架构，如果 `CONFIG_QUEUED_SPINLOCKS` 内核配置选项启用的情况。那么，在 `spin_lock_init` 宏的扩展之后，给定的`自旋锁`将会初始化并且状态变为——`解锁 (unlocked)`。

从这一时刻起我们了解了如何去初始化一个`自旋锁`，现在考虑 Linux 内核为`自旋锁`的操作提供的 [API](https://en.wikipedia.org/wiki/Application_programming_interface)。首先是：

```C
static __always_inline void spin_lock(spinlock_t *lock)
{
	raw_spin_lock(&lock->rlock);
}
```

此函数允许我们`获取`一个自旋锁。`raw_spin_lock` 宏定义在同一个头文件中，并且扩展了 `_raw_spin_lock` 函数的调用：

```C
#define raw_spin_lock(lock)	_raw_spin_lock(lock)
```

就像在 [include/linux/spinlock.h](https://github.com/torvalds/linux/blob/master/include/linux/spinlock.h) 头文件所了解的那样，`_raw_spin_lock` 宏的定义依赖于 `CONFIG_SMP` 内核配置参数：

```C
#if defined(CONFIG_SMP) || defined(CONFIG_DEBUG_SPINLOCK)
# include <linux/spinlock_api_smp.h>
#else
# include <linux/spinlock_api_up.h>
#endif
```

因此，如果在 Linux内核中 [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing) 启用了，那么 `_raw_spin_lock` 宏就在 [arch/x86/include/asm/spinlock.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/spinlock.h) 头文件中定义，并且看起来像这样：

```C
#define _raw_spin_lock(lock) __raw_spin_lock(lock)
```

`__raw_spin_lock` 函数的定义:

```C
static inline void __raw_spin_lock(raw_spinlock_t *lock)
{
        preempt_disable();
        spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
        LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```
就像你们可能了解的那样， 首先我们禁用了[抢占](https://en.wikipedia.org/wiki/Preemption_%28computing%29)，通过 [include/linux/preempt.h](https://github.com/torvalds/linux/blob/master/include/linux/preempt.h) (在 Linux 内核初始化进程章节的第九[部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-9.html)会了解到更多关于抢占)中的 `preempt_disable` 调用实现禁用。当我们将要解开给定的`自旋锁`，抢占将会再次启用：

```C
static inline void __raw_spin_unlock(raw_spinlock_t *lock)
{
        ...
        ...
        ...
        preempt_enable();
}
```
当程序正在自旋锁时，这个已经获取锁的程序必须阻止其他程序方法的抢占。`spin_acquire` 宏通过其他宏宏展调用实现：

```C
#define spin_acquire(l, s, t, i)                lock_acquire_exclusive(l, s, t, NULL, i)
#define lock_acquire_exclusive(l, s, t, n, i)           lock_acquire(l, s, t, 0, 1, n, i)
```

`lock_acquire` 函数:

```C
void lock_acquire(struct lockdep_map *lock, unsigned int subclass,
                  int trylock, int read, int check,
                  struct lockdep_map *nest_lock, unsigned long ip)
{
         unsigned long flags;

         if (unlikely(current->lockdep_recursion))
                return;

         raw_local_irq_save(flags);
         check_flags(flags);

         current->lockdep_recursion = 1;
         trace_lock_acquire(lock, subclass, trylock, read, check, nest_lock, ip);
         __lock_acquire(lock, subclass, trylock, read, check,
                        irqs_disabled_flags(flags), nest_lock, ip, 0, 0);
         current->lockdep_recursion = 0;
         raw_local_irq_restore(flags);
}
```

就像之前所写的，我们不考虑这些调试或跟踪相关的东西。`lock_acquire` 函数的主要是通过 `raw_local_irq_save` 宏调用禁用硬件中断，因为给定的自旋锁可能被启用的硬件中断所获取。以这样的方式获取的话程序将不会被抢占。注意 `lock_acquire` 函数的最后将使用 `raw_local_irq_restore` 宏的帮助再次启动硬件中断。正如你们可能猜到的那样，主要工作将在 `__lock_acquire` 函数中定义，这个函数在 [kernel/locking/lockdep.c](https://github.com/torvalds/linux/blob/master/kernel/locking/lockdep.c) 源代码文件中。

`__lock_acquire` 函数看起来很大。我们将试图去理解这个函数要做什么，但不是在这一部分。事实上这个函数于 Linux内核[锁验证器 (lock validator)](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt) 密切相关，而这也不是此部分的主题。如果我们要返回 `__raw_spin_lock` 函数的定义，我们将会发现最终这个定义包含了以下的定义：

```C
LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
```

`LOCK_CONTENDED` 宏的定义在  [include/linux/lockdep.h](https://github.com/torvalds/linux/blob/master/include/linux/lockdep.h) 头文件中，而且只是使用给定`自旋锁`调用已知函数:

```C
#define LOCK_CONTENDED(_lock, try, lock) \
         lock(_lock)
```

在本例中，`lock` 就是 [include/linux/spinlock.h](https://github.com/torvalds/linux/blob/master/include/linux/spnlock.h) 头文件中的 `do_raw_spin_lock`，而`_lock` 就是给定的 `raw_spinlock_t`：

```C
static inline void do_raw_spin_lock(raw_spinlock_t *lock) __acquires(lock)
{
        __acquire(lock);
         arch_spin_lock(&lock->raw_lock);
}
```
这里的 `__acquire` 只是[稀疏(sparse)]相关宏，并且当前我们也对这些不感兴趣。`arch_spin_lock` 函数定义的位置依赖于两件事：第一是系统架构，第二是我们是否使用了`队列自旋锁(queued spinlocks)`。本例中我们仅以 `x86_64` 架构为例介绍，因此 `arch_spin_lock` 的定义的宏表示源自 [include/asm-generic/qspinlock.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlocks.h) 头文件中：

```C
#define arch_spin_lock(l)               queued_spin_lock(l)
```

如果使用 `队列自旋锁`，或者其他例子中，`arch_spin_lock` 函数定在 [arch/x86/include/asm/spinlock.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/spinlock.h) 头文件中，如何处理？现在我们只考虑`普通的自旋锁`，`队列自旋锁`相关的信息将在以后了解。来再看看 `arch_spinlock` 结构的定义，理解以下 `arch_spin_lock` 函数的实现：

```C
typedef struct arch_spinlock {
         union {
                __ticketpair_t head_tail;
                struct __raw_tickets {
                        __ticket_t head, tail;
                } tickets;
        };
} arch_spinlock_t;
```
这个`自旋锁`的变体被称为——`标签自旋锁 (ticket spinlock)`。 就像我们锁了解的，标签自旋锁包括两个部分。当锁被获取，如果有程序想要获取`自旋锁`它就会将`尾部(tail)`值加1。如果`尾部`不等于`头部`， 那么程序就会被锁住，直到这些变量的值不再相等。来看看`arch_spin_lock`函数上的实现：

```C
static __always_inline void arch_spin_lock(arch_spinlock_t *lock)
{
        register struct __raw_tickets inc = { .tail = TICKET_LOCK_INC };

        inc = xadd(&lock->tickets, inc);

        if (likely(inc.head == inc.tail))
                goto out;

        for (;;) {
                 unsigned count = SPIN_THRESHOLD;

                 do {
                       inc.head = READ_ONCE(lock->tickets.head);
                       if (__tickets_equal(inc.head, inc.tail))
                                goto clear_slowpath;
                        cpu_relax();
                 } while (--count);
                 __ticket_lock_spinning(lock, inc.tail);
         }
clear_slowpath:
        __ticket_check_and_clear_slowpath(lock, inc.head);
out:
        barrier();
}
```

`arch_spin_lock` 函数在一开始能够使用`尾部`—— `1` 对 `__raw_tickets` 结构初始化：

```C
#define __TICKET_LOCK_INC       1
```

在`inc` 和 `lock->tickets` 的下一行执行 [xadd](http://x86.renejeschke.de/html/file_module_x86_id_327.html) 操作。这个操作之后 `inc`将存储给定`标签 (tickets)` 的值，然后 `tickets.tail` 将增加 `inc` 或 `1`。`尾部`值增加 `1` 意味着一个程序开始尝试持有锁。下一步做检查，检查`头部`和`尾部`是否有相同的值。如果值相等，这意味着没有程序持有锁并且我们去到了 `out` 标签。在 `arch_spin_lock` 函数的最后，我们可能了解了 `barrier` 宏表示 `屏障指令 (barrier instruction)`，该指令保证了编译器将不更改进入内存操作的顺序(更多关于内存屏障的知识可以阅读内核[文档 (documentation)](https://www.kernel.org/doc/Documentation/memory-barriers.txt))。

如果前一个程序持有锁而第二个程序开始执行 `arch_spin_lock` 函数，那么 `头部`将不会`等于``尾部`，因为`尾部`比`头部`大`1`。这样，程序将循环发生。在每次循坏迭代的时候`头部`和`尾部`的值进行比较。如果值不相等，`cpu_relax` ，也就是 [NOP](https://en.wikipedia.org/wiki/NOP) 指令将会被调用：

```C
#define cpu_relax()     asm volatile("rep; nop")
```

然后将开始循环的下一次迭代。如果值相等，这意味着持有锁的程序，释放这个锁并且下一个程序获取这个锁。

`spin_unlock` 操作遍布所有有 `spin_lock` 的宏或函数中，当然，使用的是 `unlock` 前缀。最后，`arch_spin_unlock` 函数将会被调用。如果看看 `arch_spin_lock` 函数的实现，我们将了解到这个函数增加了 `lock tickets` 列表的`头部`：

```C
__add(&lock->tickets.head, TICKET_LOCK_INC, UNLOCK_LOCK_PREFIX);
```
在 `spin_lock` 和 `spin_unlock` 的组合使用中，我们得到一个队列，其`头部`包含了一个索引号，映射了当前执行的持有锁的程序，而`尾部`包含了一个索引号，映射了最后尝试持有锁的程序：

```
     +-------+       +-------+
     |       |       |       |
head |   7   | - - - |   7   | tail
     |       |       |       |
     +-------+       +-------+
                         |
                     +-------+
                     |       |
                     |   8   |
                     |       |
                     +-------+
                         |
                     +-------+
                     |       |
                     |   9   |
                     |       |
                     +-------+
```

目前这就是全部。这一部分不涵盖所有的`自旋锁` API，但我认为这个概念背后的主要思想现在一定清楚了。

结论
--------------------------------------------------------------------------------

涵盖 Linux 内核中的同步原语的第一部分到此结束。在这一部分，我们遇见了第一个 Linux 内核提供的同步原语`自旋锁`。下一部分将会继续深入这个有趣的主题，而且将会了解到其他`同步`相关的知识。

如果您有疑问或者建议，请在twitter [0xAX](https://twitter.com/0xAX) 上联系我，通过 [email](anotherworldofworld@gmail.com) 联系我，或者创建一个 [issue](https://github.com/MintCN/linux-insides-zh/issues/new)。

**友情提示：英语不是我的母语，对于译文给您带来了的不便我感到非常抱歉。如果您发现任何错误请给我发送PR到 [linux-insides](https://github.com/0xAX/linux-insides)。**

链接
--------------------------------------------------------------------------------

* [Concurrent computing](https://en.wikipedia.org/wiki/Concurrent_computing)
* [Synchronization](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)
* [Clocksource framework](https://xinqiu.gitbooks.io/linux-insides-cn/content/Timers/linux-timers-2.html)
* [Mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)
* [Race condition](https://en.wikipedia.org/wiki/Race_condition)
* [Atomic operations](https://en.wikipedia.org/wiki/Linearizability)
* [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [Interrupts](https://en.wikipedia.org/wiki/Interrupt)
* [Preemption](https://en.wikipedia.org/wiki/Preemption_%28computing%29)
* [Linux kernel lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)
* [Sparse](https://en.wikipedia.org/wiki/Sparse)
* [xadd instruction](http://x86.renejeschke.de/html/file_module_x86_id_327.html)
* [NOP](https://en.wikipedia.org/wiki/NOP)
* [Memory barriers](https://www.kernel.org/doc/Documentation/memory-barriers.txt)
* [Previous chapter](https://xinqiu.gitbooks.io/linux-insides-cn/content/Timers/index.html)
