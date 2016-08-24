Linux 内核中的同步原语. 第一部分.
================================================================================

Introduction
--------------------------------------------------------------------------------
这一部分为[linux-insides](http://0xax.gitbooks.io/linux-insides/content/) 这本书开启了新的章节。定时器和时间管理相关的观念在上一个[章节](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html) 已经描述过了。现在是时候继续了。正如你可能从这个部分的标题所理解的一样，这个章节将会描述 Linux 内核中的[同步](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29) 原语。

像往常一样，在将要考虑一些同步相关的事情之前，我们将会尝试去概括地了解什么是`同步原语`。事实上，同步原语是一种提供两个或者多个[并行](https://en.wikipedia.org/wiki/Parallel_computing)进程或者线程不同时执行一个相同的代码段这样一种能力的软件机制。例如让我们看看下面的代码片段：

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
出自 [kernel/time/clocksource.c](https://github.com/torvalds/linux/master/kernel/time/clocksource.c) 源文件。这段代码来自于 `__clocksource_register_scale` 函数，此函数添加给定的 [clocksource](https://0xax.gitbooks.io/linux-insides/content/Timers/timers-2.html) 到时钟源列表中。这个函数在注册时钟源列表中生成两个不同的操作。例如 `clocksource_enqueue` 函数就是添加给定时钟源到注册时钟源列表——`clocksource_list` 中。注意这几行代码闭两个函数所包围：`mutex_lock` 和 `mutex_unlock`，这两个函数都带有一个参数——在本例中为`clocksource_mutex`。

这些函数表达了基于[互斥锁 (mutex)](https://en.wikipedia.org/wiki/Mutual_exclusion) 同步原语的加锁和解锁。当 `mutex_lock` 被执行，则允许我们阻止两个或两个以上线程执行这段代码，而`mute_unlock`还没有被互斥锁的处理拥有者锁执行。换句话说，就是阻止在`clocksource_list`上的并行操作。为什么在这里需要使用 `互斥锁`？ 如果两个并行处理尝试去注册一个时钟源会怎样。正如我们已经知道的那样，其中具有最大的等级（其具有最高的频率在系统中注册的时钟源）的列表中选择一个时钟源后，`clocksource_enqueue` 函数立即将一个给定的时钟源到`clocksource_list`列表：

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
如果两个并行处理尝试同时去执行这个函数，那么这两个处理可能会找到相同的 `入口 (entry)` 可能发生[竞态条件(race condition)](https://en.wikipedia.org/wiki/Race_condition) 或者换句话说，第二个执行 `list_add` 的处理程序，将会重写第一个线程写入的时钟源。

除了这个简答的例子，同步原语在 Linux 内核无处不在。如果再翻阅之前的[章节] (https://0xax.gitbooks.io/linux-insides/content/Timers/index.html) 或者其他章节或者如果大概看看 Linux 内核源码，就会发现许多地方都使用同步原语。我们不考虑 `mutex` 在 Linux内核是如何实现的。事实上，Linux内核提供了一系列不同的同步原语：

* `mutex`;
* `semaphores`;
* `seqlocks`;
* `atomic operations`;
* 等等。

现在从`自旋锁(spinlock)` 这个章节开始。

Linux 内核中的自旋锁。
--------------------------------------------------------------------------------

自旋锁简单来说是一种低级的同步机制，表示了一个变量可能的两个状态：

* `acquired`;
* `released`.

每一个想要获取`自旋锁`的处理，必须为这个变量写入一个表示`自旋锁获取 (spinlock acquire)`状态的值，并且为这个变量写入`所释放 (spinlock released)`状态。如果一个处理程序尝试执行受`自旋锁`保护的代码，那么代码将会被锁住，直到占有锁的处理程序释放掉。在本例中，所有相关的操作必须是
[原子的 (atomic)](https://en.wikipedia.org/wiki/Linearizability)，来阻止 [竞态条件](https://en.wikipedia.org/wiki/Race_condition) 状态。`自旋锁`在 Linux内核中使用 `spinlock_t` 类型来表示。如果我们查看 Linux 内核代码，我们会看到，这个类型被[广泛地 (widely)](http://lxr.free-electrons.com/ident?i=spinlock_t)使用。`spinlock_t` 的定义如下：

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

这段代码在 [include/linux/spinlock_types.h](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h) 头文件中。可以看出，它的实现依赖于 `CONFIG_DEBUG_LOCK_ALLOC` 内核配置选项这个状态。现在我们跳过这一块，因为所有的调试相关的事情都将会在这一部分的最后。所以，如果 `CONFIG_DEBUG_LOCK_ALLOC` 内核配置选项不可用，那么`spinlock_t` 则包含 [联合体(union)](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B)，这个联合体有一个字段——`raw_spinlock`：

```C
typedef struct spinlock {
        union {
              struct raw_spinlock rlock;
        };
} spinlock_t;
```

The `raw_spinlock` structure defined in the [same](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h) header file and represents implementation of `normal` spinlock. Let's look how the `raw_spinlock` structure is defined:

```C
typedef struct raw_spinlock {
        arch_spinlock_t raw_lock;
#ifdef CONFIG_GENERIC_LOCKBREAK
        unsigned int break_lock;
#endif
} raw_spinlock_t;
```

where the `arch_spinlock_t` represents architecture-specific `spinlock` implementation and the `break_lock` field which holds value - `1` in a case when  one processor starts to wait while the lock is held on another processor on [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing) systems. This allows prevent long time locking. As consider the [x86_64](https://en.wikipedia.org/wiki/X86-64) architecture in this books, so the `arch_spinlock_t` is defined in the [arch/x86/include/asm/spinlock_types.h](https://github.com/torvalds/linux/master/arch/x86/include/asm/spinlock_types.h) header file and looks:

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

As we may see, the definition of the `arch_spinlock` structure depends on the value of the `CONFIG_QUEUED_SPINLOCKS` kernel configuration option. This configuration option the Linux kernel supports `spinlocks` with queue. This special type of `spinlocks` which instead of `acquired` and `released` [atomic](https://en.wikipedia.org/wiki/Linearizability) values used `atomic` operation on a `queue`. If the `CONFIG_QUEUED_SPINLOCKS` kernel configuration option is enabled, the `arch_spinlock_t` will be represented by the following structure:

```C
typedef struct qspinlock {
	atomic_t	val;
} arch_spinlock_t;
```

from the [include/asm-generic/qspinlock_types.h](https://github.com/torvalds/linux/master/include/asm-generic/qspinlock_types.h) header file.

We will not stop on this structures for now and before we will consider both `arch_spinlock` and the `qspinlock`, let's look at the operations on a spinlock. The Linux kernel provides following main operations on a `spinlock`:

* `spin_lock_init` - produces initialization of the given `spinlock`;
* `spin_lock` - acquires given `spinlock`;
* `spin_lock_bh` - disables software [interrupts](https://en.wikipedia.org/wiki/Interrupt) and acquire given `spinlock`.
* `spin_lock_irqsave` and `spin_lock_irq` - disable interrupts on local processor and preserve/not preserve previous interrupt state in the `flags`;
* `spin_unlock` - releases given `spinlock`;
* `spin_unlock_bh` - releases given `spinlock` and enables software interrupts;
* `spin_is_locked` - returns the state of the given `spinlock`;
* and etc.

Let's look on the implementation of the `spin_lock_init` macro. As I already wrote, this and other macro are defined in the [include/linux/spinlock.h](https://github.com/torvalds/linux/master/include/linux/spinlock.h) header file and the `spin_lock_init` macro looks:

```C
#define spin_lock_init(_lock)		\
do {							                \
	spinlock_check(_lock);				        \
	raw_spin_lock_init(&(_lock)->rlock);		\
} while (0)
```

As we may see, the `spin_lock_init` macro takes a `spinlock` and executes two operations: check the given `spinlock` and execute the `raw_spin_lock_init`. The implementation of the `spinlock_check` is pretty easy, this function just returns the `raw_spinlock_t` of the given `spinlock` to be sure that we got exactly `normal` raw spinlock:

```C
static __always_inline raw_spinlock_t *spinlock_check(spinlock_t *lock)
{
	return &lock->rlock;
}
```

The `raw_spin_lock_init` macro:

```C
# define raw_spin_lock_init(lock)		\
do {                                                  \
    *(lock) = __RAW_SPIN_LOCK_UNLOCKED(lock);         \
} while (0)                                           \
```

assigns the value of the `__RAW_SPIN_LOCK_UNLOCKED` with the given `spinlock` to the given `raw_spinlock_t`. As we may understand from the name of the `__RAW_SPIN_LOCK_UNLOCKED` macro, this macro does initialization of the given `spinlock` and set it to `released` state. This macro defined in the [include/linux/spinlock_types.h](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h) header file and expands to the following macros:

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

As I already wrote above, we will not consider stuff which is related to debugging of synchronization primitives. In this case we will not consider the `SPIN_DEBUG_INIT` and the `SPIN_DEP_MAP_INIT` macros. So the `__RAW_SPINLOCK_UNLOCKED` macro will be expanded to the:

```C
*(&(_lock)->rlock) = __ARCH_SPIN_LOCK_UNLOCKED;
```

where the `__ARCH_SPIN_LOCK_UNLOCKED` is:

```C
#define __ARCH_SPIN_LOCK_UNLOCKED       { { 0 } }
```

and:

```C
#define __ARCH_SPIN_LOCK_UNLOCKED       { ATOMIC_INIT(0) }
```

for the [x86_64](https://en.wikipedia.org/wiki/X86-64) architecture. if the `CONFIG_QUEUED_SPINLOCKS` kernel configuration option is enabled. So, after the expansion of the `spin_lock_init` macro, a given `spinlock` will be initialized and its state will be - `unlocked`.

From this moment we know how to initialize a `spinlock`, now let's consider [API](https://en.wikipedia.org/wiki/Application_programming_interface) which Linux kernel provides for manipulations of `spinlocks`. The first is:

```C
static __always_inline void spin_lock(spinlock_t *lock)
{
	raw_spin_lock(&lock->rlock);
}
```

function which allows us to `acquire` a spinlock. The `raw_spin_lock` macro is defined in the same header file and expands to the call of the `_raw_spin_lock` function:

```C
#define raw_spin_lock(lock)	_raw_spin_lock(lock)
```

As we may see in the [include/linux/spinlock.h](https://github.com/torvalds/linux/blob/master/include/linux/spinlock.h) header file, definition of the `_raw_spin_lock` macro depends on the `CONFIG_SMP` kernel configuration parameter:

```C
#if defined(CONFIG_SMP) || defined(CONFIG_DEBUG_SPINLOCK)
# include <linux/spinlock_api_smp.h>
#else
# include <linux/spinlock_api_up.h>
#endif
```

So, if the [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing) is enabled in the Linux kernel, the `_raw_spin_lock` macro is defined in the [arch/x86/include/asm/spinlock.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/spinlock.h) header file and looks like:

```C
#define _raw_spin_lock(lock) __raw_spin_lock(lock)
```

The `__raw_spin_lock` function looks:

```C
static inline void __raw_spin_lock(raw_spinlock_t *lock)
{
        preempt_disable();
        spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
        LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```

As you may see, first of all we disable [preemption](https://en.wikipedia.org/wiki/Preemption_%28computing%29) by the call of the `preempt_disable` macro from the [include/linux/preempt.h](https://github.com/torvalds/linux/blob/master/include/linux/preempt.h) (more about this you may read in the ninth [part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-9.html) of the Linux kernel initialization process chapter). When we will unlock the given `spinlock`, preemption will be enabled again:

```C
static inline void __raw_spin_unlock(raw_spinlock_t *lock)
{
        ...
        ...
        ...
        preempt_enable();
}
```

We need to do this while a process is spinning on a lock, other processes must be prevented to preempt the process which acquired a lock. The `spin_acquire` macro which through a chain of other macros expands to the call of the:

```C
#define spin_acquire(l, s, t, i)                lock_acquire_exclusive(l, s, t, NULL, i)
#define lock_acquire_exclusive(l, s, t, n, i)           lock_acquire(l, s, t, 0, 1, n, i)
```

`lock_acquire` function:

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

As I wrote above, we will not consider stuff here which is related to debugging or tracing. The main point of the `lock_acquire` function is to disable hardware interrupts by the call of the `raw_local_irq_save` macro, because the given spinlock might be acquired with enabled hardware interrupts. In this way the process will not be preempted. Note that in the end of the `lock_acquire` function we will enable hardware interrupts again with the help of the `raw_local_irq_restore` macro. As you already may guess, the main work will be in the `__lock_acquire` function which is defined in the [kernel/locking/lockdep.c](https://github.com/torvalds/linux/blob/master/kernel/locking/lockdep.c) source code file.

The `__lock_acquire` function looks big. We will try to understand what does this function do, but not in this part. Actually this function mostly related to the Linux kernel [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt) and it is not topic of this part. If we will return to the definition of the `__raw_spin_lock` function, we will see that it contains the following definition in the end:

```C
LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
```

The `LOCK_CONTENDED` macro is defined in the [include/linux/lockdep.h](https://github.com/torvalds/linux/blob/master/include/linux/lockdep.h) header file and just calls the given function with the given `spinlock`:

```C
#define LOCK_CONTENDED(_lock, try, lock) \
         lock(_lock)
```

In our case, the `lock` is `do_raw_spin_lock` function from the [include/linux/spinlock.h](https://github.com/torvalds/linux/blob/master/include/linux/spnlock.h) header file and the `_lock` is the given `raw_spinlock_t`:

```C
static inline void do_raw_spin_lock(raw_spinlock_t *lock) __acquires(lock)
{
        __acquire(lock);
         arch_spin_lock(&lock->raw_lock);
}
```

The `__acquire` here is just [sparse](https://en.wikipedia.org/wiki/Sparse) related macro and we are not interesting in it in this moment. Location of the definition of the `arch_spin_lock` function depends on two things: the first is architecture of system and the second do we use `queued spinlocks` or not. In our case we consider only `x86_64` architecture, so the definition of the `arch_spin_lock` is represented as the macro from the [include/asm-generic/qspinlock.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlocks.h) header file:

```C
#define arch_spin_lock(l)               queued_spin_lock(l)
```

if we are using `queued spinlocks`. Or in other case, the `arch_spin_lock` function is defined in the [arch/x86/include/asm/spinlock.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/spinlock.h) header file. Now we will consider only `normal spinlock` and information related to `queued spinlocks` we will see later. Let's look again on the definition of the `arch_spinlock` structure, to understand implementation of the `arch_spin_lock` function:

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

This variant of `spinlock` is called - `ticket spinlock`. As we may see, it consists from two parts. When lock is acquired, it increments a `tail` by one every time when a process wants to hold a `spinlock`. If the `tail` is not equal to `head`, the process will be locked, until values of these variables will not be equal. Let's look on the implementation of the `arch_spin_lock` function:

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

At the beginning of the `arch_spin_lock` function we can initialization of the `__raw_tickets` structure with `tail` - `1`:

```C
#define __TICKET_LOCK_INC       1
```

In the next line we execute [xadd](http://x86.renejeschke.de/html/file_module_x86_id_327.html) operation on the `inc` and `lock->tickets`. After this operation the `inc` will store value of the `tickets` of the given `lock` and the `tickets.tail` will be increased on `inc` or `1`. The `tail` value was increased on `1` which means that one process started to try to hold a lock. In the next step we do the check that checks that `head` and `tail` have the same value. If these values are equal, this means that nobody holds lock and we go to the `out` label. In the end of the `arch_spin_lock` function we may see the `barrier` macro which represents `barrier instruction` which guarantees that compiler will not change order of operations that access memory (more about memory barriers you can read in the kernel [documentation](https://www.kernel.org/doc/Documentation/memory-barriers.txt)).

If one process held a lock and a second process started to execute the `arch_spin_lock` function, the `head` will not be `equal` to `tail`, because the `tail` will be greater than `head` on `1`. In this way, process will occur in the loop. There will be comparison between `head` and the `tail` values at each loop iteration. If these values are not equal, the `cpu_relax` will be called which is just [NOP](https://en.wikipedia.org/wiki/NOP) instruction:


```C
#define cpu_relax()     asm volatile("rep; nop")
```

and the next iteration of the loop will be started. If these values will be equal, this means that the process which held this lock, released this lock and the next process may acquire the lock.

The `spin_unlock` operation goes through the all macros/function as `spin_lock`, of course with `unlock` prefix. In the end the `arch_spin_unlock` function will be called. If we will look at the implementation of the `arch_spin_lock` function, we will see that it increases `head` of the `lock tickets` list:

```C
__add(&lock->tickets.head, TICKET_LOCK_INC, UNLOCK_LOCK_PREFIX);
```

In a combination of the `spin_lock` and `spin_unlock`, we get kind of queue where `head` contains an index number which maps currently executed process which holds a lock and the `tail` which contains an index number which maps last process which tried to hold the lock:

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

That's all for now. We didn't cover `spinlock` API in full in this part, but I think that the main idea behind this concept must be clear now.

Conclusion
--------------------------------------------------------------------------------

This concludes the first part covering synchronization primitives in the Linux kernel. In this part, we met first synchronization primitive `spinlock` provided by the Linux kernel. In the next part we will continue to dive into this interesting theme and will see other `synchronization` related stuff.

If you have questions or suggestions, feel free to ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create [issue](https://github.com/0xAX/linux-insides/issues/new).

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [Concurrent computing](https://en.wikipedia.org/wiki/Concurrent_computing)
* [Synchronization](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)
* [Clocksource framework](https://0xax.gitbooks.io/linux-insides/content/Timers/timers-2.html)
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
* [Previous chapter](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)
