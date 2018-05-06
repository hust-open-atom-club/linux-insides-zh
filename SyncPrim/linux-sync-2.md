Linux 内核的同步原语. 第二部分.
================================================================================

队列自旋锁
--------------------------------------------------------------------------------

这是本[章节](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/index.html)的第二部分，这部分描述 Linux 内核的和我们在本章的第一[部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/linux-sync-1.html)所见到的－－[自旋锁](https://en.wikipedia.org/wiki/Spinlock)的同步原语。在这个部分我们将继续学习自旋锁的同步原语。 如果阅读了上一部分的相关内容，你可能记得除了正常自旋锁，Linux 内核还提供`自旋锁`的一种特殊类型 - `队列自旋锁`。 在这个部分我们将尝试理解此概念锁代表的含义。

我们在上一[部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/linux-sync-1.html)已知`自旋锁`的 [API](https://en.wikipedia.org/wiki/Application_programming_interface):

* `spin_lock_init` - 为给定`自旋锁`进行初始化；
* `spin_lock` - 获取给定`自旋锁`；
* `spin_lock_bh` - 禁止软件[中断](https://en.wikipedia.org/wiki/Interrupt)并且获取给定`自旋锁`；
* `spin_lock_irqsave` 和 `spin_lock_irq` - 禁止本地处理器中断并且保存/不保存之前`标识位`的中断状态；
* `spin_unlock` - 释放给定的`自旋锁`；
* `spin_unlock_bh` - 释放给定的`自旋锁`并且启用软件中断；
* `spin_is_locked` - 返回给定`自旋锁`的状态；
* 等等。

而且我们知道所有这些宏都在 [include/linux/spinlock.h](https://github.com/torvalds/linux/blob/master/include/linux/spinlock.h) 头文件中所定义，都被扩展成针对 [x86_64](https://en.wikipedia.org/wiki/X86-64) 架构，来自于 [arch/x86/include/asm/spinlock.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/spinlock.h) 文件的 `arch_spin_.*` 前缀的函数调用。如果我们关注这个头文件，我们会发现这些函数(`arch_spin_is_locked`， `arch_spin_lock`， `arch_spin_unlock` 等等)只在 `CONFIG_QUEUED_SPINLOCKS` 内核配置选项禁用的时才定义：

```c
#ifdef CONFIG_QUEUED_SPINLOCKS
#include <asm/qspinlock.h>
#else
static __always_inline void arch_spin_lock(arch_spinlock_t *lock)
{
    ...
    ...
    ...
}
...
...
...
#endif
```
这意味着 [arch/x86/include/asm/qspinlock.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/qspinlock.h) 这个头文件提供提供这些函数自己的实现。实际上这些函数是宏定义并且在分布在其他头文件中。这个头文件是 [include/asm-generic/qspinlock.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlock.h#L126)。如果我们查看这个头文件，我们会发现这些宏的定义：

```c
#define arch_spin_is_locked(l)          queued_spin_is_locked(l)
#define arch_spin_is_contended(l)       queued_spin_is_contended(l)
#define arch_spin_value_unlocked(l)     queued_spin_value_unlocked(l)
#define arch_spin_lock(l)               queued_spin_lock(l)
#define arch_spin_trylock(l)            queued_spin_trylock(l)
#define arch_spin_unlock(l)             queued_spin_unlock(l)
#define arch_spin_lock_flags(l, f)      queued_spin_lock(l)
#define arch_spin_unlock_wait(l)        queued_spin_unlock_wait(l)
```

在我们考虑怎么排列自旋锁和实现他们的 [API](https://en.wikipedia.org/wiki/Application_programming_interface)，我们首先看看理论部分。

介绍队列自旋锁
-------------------------------------------------------------------------------

队列自旋锁是 Linux 内核的[锁机制](https://en.wikipedia.org/wiki/Lock_%28computer_science%29)，是标准`自旋锁`的代替物。至少对 [x86_64](https://en.wikipedia.org/wiki/X86-64) 架构是真的。如果我们查看了以下内核配置文件 - [kernel/Kconfig.locks](https://github.com/torvalds/linux/blob/master/kernel/Kconfig.locks)，我们将会发现以下配置入口：

```
config ARCH_USE_QUEUED_SPINLOCKS
	bool

config QUEUED_SPINLOCKS
	def_bool y if ARCH_USE_QUEUED_SPINLOCKS
	depends on SMP
```
这意味着如果 `ARCH_USE_QUEUED_SPINLOCKS` 启用，那么 `CONFIG_QUEUED_SPINLOCKS` 内核配置选项将默认启用。 我们能够看到 `ARCH_USE_QUEUED_SPINLOCKS` 在 `x86_64` 特定内核配置文件 - [arch/x86/Kconfig](https://github.com/torvalds/linux/blob/master/arch/x86/Kconfig) 默认开启：

```
config X86
    ...
    ...
    ...
    select ARCH_USE_QUEUED_SPINLOCKS
    ...
    ...
    ...
```

在开始考虑什么是队列自旋锁概念之前，让我们看看其他`自旋锁`的类型。一开始我们考虑`正常`自旋锁是如何实现的。通常，`正常`自旋锁的实现是基于 [test and set](https://en.wikipedia.org/wiki/Test-and-set) 指令。这个指令的工作原则真的很简单。该指令写入一个值到内存地址然后返回该地址原来的旧值。这些操作都是在院子的上下文中完成的。也就是说，这个指令是不可中断的。因此如果第一个线程开始执行这个指令，第二个线程将会等待，直到第一个线程完成。基本锁可以在这个机制之上建立。可能看起来如下所示：

```C
int lock(lock)
{
    while (test_and_set(lock) == 1)
        ;
    return 0;
}

int unlock(lock)
{
    lock=0;

    return lock;
}
```

第一个线程将执行 `test_and_set` 指令设置 `lock` 为 `1`。当第二个线程调用 `lock` 函数，它将在 `while` 循环中自旋，直到第一个线程调用 `unlock` 函数而且 `lock` 等于 `0`。这个实现对于执行不是很好，因为该实现至少有两个问题。第一个问题是该实现可能是非公平的而且一个处理器的线程可能有很长的等待时间，即使有其他线程也在等待释放锁，它还是调用了 `lock`。第二个问题是所有想要获取锁的线程，必须在共享内存的变量上执行很多类似`test_and_set` 这样的`原子`操作。这导致缓存失效，因为处理器缓存会存储 `lock=1`，但是在线程释放锁之后，内存中 `lock`可能只是`1`。

在上一[部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/linux-sync-1.html) 我们了解了自旋锁的第二种实现 -
`排队自旋锁(ticket spinlock)`。这一方法解决了第一个问题而且能够保证想要获取锁的线程的顺序，但是仍然存在第二个问题。

这一部分的主旨是 `队列自旋锁`。这个方法能够帮助解决上述的两个问题。`队列自旋锁`允许每个处理器对自旋过程使用他自己的内存地址。通过学习名为 [MCS](http://www.cs.rochester.edu/~scott/papers/1991_TOCS_synch.pdf) 锁的这种基于队列自旋锁的实现，能够最好理解基于队列自旋锁的基本原则。在了解`队列自旋锁`的实现之前，我们先尝试理解什么是 `MCS` 锁。

`MCS`锁的基本理念就在上一段已经写到了，一个线程在本地变量上自旋然后每个系统的处理器自己拥有这些变量的拷贝。换句话说这个概念建立在 Linux 内核中的 [per-cpu](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html) 变量概念之上。

当第一个线程想要获取锁，线程在`队列`中注册了自身，或者换句话说，因为线程现在是闲置的，线程要加入特殊`队列`并且获取锁。当第二个线程想要在第一个线程释放锁之前获取相同锁，这个线程就会把他自身的所变量的拷贝加入到这个特殊`队列`中。这个例子中第一个线程会包含一个 `next` 字段指向第二个线程。从这一时刻，第二个线程会等待直到第一个线程释放它的锁并且关于这个事件通知给 `next` 线程。第一个线程从`队列`中删除而第二个线程持有该锁。

我们可以这样代表示意一下：

空队列：

```
+---------+
|         |
|  Queue  |
|         |
+---------+
```

第一个线程尝试获取锁：

```
+---------+     +----------------------------+
|         |     |                            |
|  Queue  |---->| First thread acquired lock |
|         |     |                            |
+---------+     +----------------------------+
```

第二个队列尝试获取锁:

```
+---------+     +----------------------------------------+     +-------------------------+
|         |     |                                        |     |                         |
|  Queue  |---->|  Second thread waits for first thread  |<----| First thread holds lock |
|         |     |                                        |     |                         |
+---------+     +----------------------------------------+     +-------------------------+
```

或者伪代码描述为：

```C
void lock(...)
{
    lock.next = NULL;
    ancestor = put_lock_to_queue_and_return_ancestor(queue, lock);

    // if we have ancestor, the lock already acquired and we
    // need to wait until it will be released
    if (ancestor)
    {
        lock.locked = 1;
        ancestor.next = lock;

        while (lock.is_locked == true)
            ;
    }

    // in other way we are owner of the lock and may exit
}

void unlock(...)
{
    // do we need to notify somebody or we are alonw in the
    // queue?
    if (lock.next != NULL) {
        // the while loop from the lock() function will be
        // finished
        lock.next.is_locked = false;
        // delete ourself from the queue and exit
        ...
        ...
        ...
        return;
    }

    // So, we have no next threads in the queue to notify about
    // lock releasing event. Let's just put `0` to the lock, will
    // delete ourself from the queue and exit.
}
```

想法很简单，但是`队列自旋锁`的实现一定是比伪代码复杂。就如同我上面写到的，`队列自旋锁`机制计划在 Linux 内核中成为`排队自旋锁`的替代品。但你们可能还记得，常用`自旋锁`适用于`32位(32-bit)`的 [字(word)](https://en.wikipedia.org/wiki/Word_%28computer_architecture%29)。而基于`MCS`的锁不能使用这个大小，你们可能知道 `spinlock_t` 类型在 Linux 内核中的使用是[宽字符(widely)](http://lxr.free-electrons.com/ident?i=spinlock_t)的。这种情况下可能不得不重写 Linux 内核中重要的组成部分，但这是不可接受的。除了这一点，一些包含自旋锁用于保护的内核结构不能增长大小。但无论怎样，基于这一概念的 Linux 内核中的`队列自旋锁`实现有一些修改，可以适应`32`位的字。

这就是所有有关`队列自旋锁`的理论，现在让我们考虑以下在 Linux 内核中这个机制是如何实现的。`队列自旋锁`的实现看起来比`排队自旋锁`的实现更加复杂和混乱，但是细致的研究会引导成功。

队列自旋锁的API
-------------------------------------------------------------------------------

现在我们从原理角度了解了一些`队列自旋锁`，是时候了解 Linux 内核中这一机制的实现了。就像我们之前了解的那样 [include/asm-generic/qspinlock.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlock.h#L126) 头文件提供一套宏，代表 API 中的自旋锁的获取、释放等等。

```C
#define arch_spin_is_locked(l)          queued_spin_is_locked(l)
#define arch_spin_is_contended(l)       queued_spin_is_contended(l)
#define arch_spin_value_unlocked(l)     queued_spin_value_unlocked(l)
#define arch_spin_lock(l)               queued_spin_lock(l)
#define arch_spin_trylock(l)            queued_spin_trylock(l)
#define arch_spin_unlock(l)             queued_spin_unlock(l)
#define arch_spin_lock_flags(l, f)      queued_spin_lock(l)
#define arch_spin_unlock_wait(l)        queued_spin_unlock_wait(l)
```

所有这些宏扩展了同一头文件下的函数的调用。此外，我们发现 [include/asm-generic/qspinlock_types.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlock_types.h) 头文件的 `qspinlock` 结构代表了 Linux 内核队列自旋锁。

```C
typedef struct qspinlock {
	atomic_t	val;
} arch_spinlock_t;
```

如我们所了解的，`qspinlock` 结构只包含了一个字段 - `val`。这个字段代表给定`自旋锁`的状态。`4` 个字节字段包括如下 4 个部分：

* `0-7` - 上锁字节(locked byte);
* `8` - 未决位(pending bit);
* `16-17` - 这两位代表了 `MCS` 锁的 `per_cpu` 数组(马上就会了解)；
* `18-31` - 包括表明队列尾部的处理器数。

`9-15` 字节没有被使用。

就像我们已经知道的，系统中每个处理器有自己的锁拷贝。这个锁由以下结构所表示：

```C
struct mcs_spinlock {
       struct mcs_spinlock *next;
       int locked;
       int count;
};
```

来自 [kernel/locking/mcs_spinlock.h](https://github.com/torvalds/linux/blob/master/kernel/locking/mcs_spinlock.h) 头文件。第一个字段代表了指向`队列`中下一个线程的指针。第二个字段代表了`队列`中当前线程的状态，其中 `1` 是 `锁`已经获取而 `0` 相反。然后最后一个 `mcs_spinlock` 字段 结构代表嵌套锁 (nested locks)，了解什么是嵌套锁，就像想象一下当线程已经获取锁的情况，而被硬件[中断](https://en.wikipedia.org/wiki/Interrupt) 所中断，然后[中断处理程序](https://en.wikipedia.org/wiki/Interrupt_handler)又尝试获取锁。这个例子里，每个处理器不只是 `mcs_spinlock` 结构的拷贝，也是这些结构的数组：

```C
static DEFINE_PER_CPU_ALIGNED(struct mcs_spinlock, mcs_nodes[4]);
```

此数组允许以下情况的四个事件的锁获取的四个尝试(原文：This array allows to make four attempts of a lock acquisition for the four events in following contexts:
)：
* 普通任务上下文；
* 硬件中断上下文；
* 软件中断上下文；
* 屏蔽中断上下文。

现在让我们返回 `qspinlock` 结构和`队列自旋锁`的 `API` 中来。在我们考虑`队列自旋锁`的 `API` 之前，请注意 `qspinlock` 结构的 `val` 字段有类型 - `atomic_t`，此类型代表原子变量或者变量的一次操作(原文：one operation at a time variable)。一次，所有这个字段的操作都是[原子的](https://en.wikipedia.org/wiki/Linearizability)。比如说让我们看看 `val` API 的值：

```C
static __always_inline int queued_spin_is_locked(struct qspinlock *lock)
{
	return atomic_read(&lock->val);
}
```

Ok，现在我们知道 Linux 内核的代表队列自旋锁数据结构，那么是时候看看`队列自旋锁`[API](https://en.wikipedia.org/wiki/Application_programming_interface)中`主要（main）`函数的实现。

```C
#define arch_spin_lock(l)               queued_spin_lock(l)
```

没错，这个函数是 - `queued_spin_lock`。正如我们可能从函数名中所了解的一样，函数允许通过线程获取锁。这个函数在 [include/asm-generic/qspinlock_types.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlock_types.h) 头文件中定义，它的实现看起来是这样：

```C
static __always_inline void queued_spin_lock(struct qspinlock *lock)
{
        u32 val;

        val = atomic_cmpxchg_acquire(&lock->val, 0, _Q_LOCKED_VAL);
        if (likely(val == 0))
                 return;
        queued_spin_lock_slowpath(lock, val);
}
```

看起来很简单，除了 `queued_spin_lock_slowpath` 函数，我们可能发现它只有一个参数。在我们的例子中这个参数代表 `队列自旋锁` 被上锁。让我们考虑`队列`锁为空，现在第一个线程想要获取锁的情况。正如我们可能了解的 `queued_spin_lock` 函数从调用 `atomic_cmpxchg_acquire` 宏开始。就像你们可能从宏的名字猜到的那样，它执行原子的 [CMPXCHG](http://x86.renejeschke.de/html/file_module_x86_id_41.html) 指令，使用第一个参数（当前给定自旋锁的状态）比较第二个参数（在我们的例子为零）的值，如果他们相等，那么第二个参数在存储位置保存 `_Q_LOCKED_VAL` 的值，该存储位置通过 `&lock->val` 指向并且返回这个存储位置的初始值。

`atomic_cmpxchg_acquire` 宏定义在 [include/linux/atomic.h](https://github.com/torvalds/linux/blob/master/include/linux/atomic.h) 头文件中并且扩展了 `atomic_cmpxchg` 函数的调用：

```C
#define  atomic_cmpxchg_acquire         atomic_cmpxchg
```
这实现是架构所指定的。我们考虑 [x86_64](https://en.wikipedia.org/wiki/X86-64) 架构，因此在我们的例子中这个头文件在 [arch/x86/include/asm/atomic.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/atomic.h) 并且`atomic_cmpxchg` 函数的实现只是返回 `cmpxchg` 宏的结果：

```C
static __always_inline int atomic_cmpxchg(atomic_t *v, int old, int new)
{
        return cmpxchg(&v->counter, old, new);
}
```

这个宏在[arch/x86/include/asm/cmpxchg.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/cmpxchg.h)头文件中定义，看上去是这样：

```C
#define cmpxchg(ptr, old, new) \
    __cmpxchg(ptr, old, new, sizeof(*(ptr)))

#define __cmpxchg(ptr, old, new, size) \
    __raw_cmpxchg((ptr), (old), (new), (size), LOCK_PREFIX)
```

就像我们可能了解的那样，`cmpxchg` 宏使用几乎相同的参数集合扩展了 `__cpmxchg` 宏。新添加的参数是原子值的大小。`__cpmxchg` 宏添加了 `LOCK_PREFIX`，还扩展了 `__raw_cmpxchg` 宏中 `LOCK_PREFIX`的 [LOCK](http://x86.renejeschke.de/html/file_module_x86_id_159.html)指令。毕竟 `__raw_cmpxchg` 对我们来说做了所有的的工作：

```C
#define __raw_cmpxchg(ptr, old, new, size, lock) \
({
    ...
    ...
    ...
    volatile u32 *__ptr = (volatile u32 *)(ptr);            \
    asm volatile(lock "cmpxchgl %2,%1"                      \
                 : "=a" (__ret), "+m" (*__ptr)              \
                 : "r" (__new), "" (__old)                  \
                 : "memory");                               \
    ...
    ...
    ...
})
```

在 `atomic_cmpxchg_acquire` 宏被执行后，该宏返回内存地址之前的值。现在只有一个线程尝试获取锁，因此 `val` 将会置为零然后我们从 `queued_spin_lock` 函数返回：

```C
val = atomic_cmpxchg_acquire(&lock->val, 0, _Q_LOCKED_VAL);
if (likely(val == 0))
    return;
```

此时此刻，我们的第一个线程持有锁。注意这个行为与在 `MCS` 算法的描述有所区别。线程获取锁，但是我们不添加此线程入`队列`。就像我之前已经写到的，`队列自旋锁` 概念的实现在 Linux 内核中基于 `MCS` 算法，但是于此同时它对优化目的有一些差异。

所以第一个线程已经获取了锁然后现在让我们考虑第二个线程尝试获取相同的锁的情况。第二个线程将从同样的 `queued_spin_lock` 函数开始，但是 `lock->val` 会包含 `1` 或者 `_Q_LOCKED_VAL`，因为第一个线程已经持有了锁。因此，在本例中 `queued_spin_lock_slowpath` 函数将会被调用。`queued_spin_lock_slowpath`函数定义在 [kernel/locking/qspinlock.c](https://github.com/torvalds/linux/blob/master/kernel/locking/qspinlock.c) 源码文件中并且从以下的检查开始：

```C
void queued_spin_lock_slowpath(struct qspinlock *lock, u32 val)
{
	if (pv_enabled())
	    goto queue;

    if (virt_spin_lock(lock))
		return;

    ...
    ...
    ...
}
```

这些检查操作检查了 `pvqspinlock` 的状态。`pvqspinlock` 是在[准虚拟化（paravirtualized）](https://en.wikipedia.org/wiki/Paravirtualization)环境中的`队列自旋锁`。就像这一章节只相关 Linux 内核同步原语一样，我们跳过这些和其他不直接相关本章节主题的部分。这些检查之后我们比较使用 `_Q_PENDING_VAL` 宏的值所代表的锁，然后什么都不做直到该比较为真（原文：After these checks we compare our value which represents lock with the value of the `_Q_PENDING_VAL` macro and do nothing while this is true）：

```C
if (val == _Q_PENDING_VAL) {
	while ((val = atomic_read(&lock->val)) == _Q_PENDING_VAL)
		cpu_relax();
}
```
这里 `cpu_relax` 只是 [NOP](https://en.wikipedia.org/wiki/NOP) 指令。综上，我们了解了锁饱含着 - `pending` 位。这个位代表了想要获取锁的线程，但是这个锁已经被其他线程获取了，并且与此同时`队列`为空。在本例中，`pending` 位将被设置并且`队列`不会被创建(touched)。这是优化所完成的，因为不需要考虑在引发缓存无效的自身 `mcs_spinlock` 数组的创建产生的非必需隐患（原文：This is done for optimization, because there are no need in unnecessary latency which will be caused by the cache invalidation in a touching of own `mcs_spinlock` array.）。

下一步我们进入下面的循环：

```C
for (;;) {
	if (val & ~_Q_LOCKED_MASK)
		goto queue;

	new = _Q_LOCKED_VAL;
	if (val == new)
		new |= _Q_PENDING_VAL;

	old = atomic_cmpxchg_acquire(&lock->val, val, new);
	if (old == val)
		break;

	val = old;
}
```

这里第一个 `if` 子句检查锁 (`val`) 的状态是上锁还是待定的(pending)。这意味着第一个线程已经获取了锁，第二个线程也试图获取锁，但现在第二个线程是待定状态。本例中我们需要开始建立队列。我们将稍后考虑这个情况。在我们的例子中，第一个线程持有锁而第二个线程也尝试获取锁。这个检查之后我们在上锁状态并且使用之前锁状态比较后创建新锁。就像你记得的那样，`val` 包含了 `&lock->val` 状态，在第二个线程调用 `atomic_cmpxchg_acquire` 宏后状态将会等于 `1`。由于 `new` 和 `val` 的值相等，所以我们在第二个线程的锁上设置待定位。在此之后，我们需要再次检查 `&lock->val` 的值，因为第一个线程可能在这个时候释放锁。如果第一个线程还又没释放锁，`旧`的值将等于 `val` （因为 `atomic_cmpxchg_acquire` 将会返回存储地址指向 `lock->val` 的值并且当前为 `1`）然后我们将退出循环。因为我们退出了循环，我们会等待第一个线程直到它释放锁，清除待定位，获取锁并且返回：

```C
smp_cond_acquire(!(atomic_read(&lock->val) & _Q_LOCKED_MASK));
clear_pending_set_locked(lock);
return;
```

注意我们还没创建`队列`。这里我们不需要，因为对于两个线程来说，队列只是导致对内存访问的非必需潜在因素。在其他的例子中，第一个线程可能在这个时候释放其锁。在本例中 `lock->val` 将包含 `_Q_LOCKED_VAL | _Q_PENDING_VAL` 并且我们会开始建立`队列`。通过获得处理器执行线程的本地 `mcs_nodes` 数组的拷贝我们开始建立`队列`：

```C
node = this_cpu_ptr(&mcs_nodes[0]);
idx = node->count++;
tail = encode_tail(smp_processor_id(), idx);
```

除此之外我们计算 表示`队列`尾部和代表 `mcs_nodes` 数组实体的`索引`的`tail` 。在此之后我们设置 `node` 指向正确的 `mcs_nodes` 数组，设置 `locked` 为零应为这个线程还没有获取锁，还有 `next` 为 `NULL` 因为我们不知道任何有关其他`队列`实体的信息：

```C
node += idx;
node->locked = 0;
node->next = NULL;
```

我们已经创建了对于执行当前线程想获取锁的处理器的队列的`每个 cpu（per-cpu）` 的拷贝，这意味着锁的拥有者可能在这个时刻释放了锁。因此我们可能通过 `queued_spin_trylock` 函数的调用尝试去再次获取锁。

```C
if (queued_spin_trylock(lock))
		goto release;
```

`queued_spin_trylock` 函数在 [include/asm-generic/qspinlock.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/qspinlock.h) 头文件中被定义而且就像 `queued_spin_lock` 函数一样：

```C
static __always_inline int queued_spin_trylock(struct qspinlock *lock)
{
	if (!atomic_read(&lock->val) &&
	   (atomic_cmpxchg_acquire(&lock->val, 0, _Q_LOCKED_VAL) == 0))
		return 1;
	return 0;
}
```
如果锁成功被获取那么我们跳过`释放`标签而释放`队列`中的一个节点：

```C
release:
	this_cpu_dec(mcs_nodes[0].count);
```

现在我们不再需要它了，因为锁已经获得了。如果 `queued_spin_trylock` 不成功，我们更新队列的尾部：

```C
old = xchg_tail(lock, tail);
```

然后检索原先的尾部。下一步是检查`队列`是否为空。这个例子中我们需要用新的实体链接之前的实体：

```C
if (old & _Q_TAIL_MASK) {
	prev = decode_tail(old);
	WRITE_ONCE(prev->next, node);

    arch_mcs_spin_lock_contended(&node->locked);
}
```

队列实体链接之后，我们开始等待直到队列的头部到来。由于我们等待头部，我们需要对可能在这个等待实践加入的新的节点做一些检查：

```C
next = READ_ONCE(node->next);
if (next)
	prefetchw(next);
```

如果新节点被添加，我们从通过使用 [PREFETCHW](http://www.felixcloutier.com/x86/PREFETCHW.html) 指令指出下一个队列实体的内存中预先去除缓存线（cache line）。以优化为目的我们现在预先载入这个指针。我们只是改变了队列的头而这意味着有将要到来的 `MCS` 进行解锁操作并且下一个实体会被创建。

是的，从这个时刻我们在`队列`的头部。但是在我们有能力获取锁之前，我们需要至少等待两个事件：当前锁的拥有者释放锁和第二个线程处于`待定`位也获取锁：

```C
smp_cond_acquire(!((val = atomic_read(&lock->val)) & _Q_LOCKED_PENDING_MASK));
```

两个线程都释放锁后，`队列`的头部会持有锁。最后我们只是需要更新`队列`尾部然后移除从队列中移除头部。

以上。

总结
--------------------------------------------------------------------------------

这是 Linux 内核[同步原语](https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29)章节第二部分的结尾。在上一个[部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/linux-sync-1.html)我们已经见到了第一个同步原语`自旋锁`通过 Linux 内核 实现的`排队自旋锁（ticket spinlock）`。在这个部分我们了解了另一个`自旋锁`机制的实现 - `队列自旋锁`。下一个部分我们继续深入 Linux 内核同步原语。

如果您有疑问或者建议，请在twitter [0xAX](https://twitter.com/0xAX) 上联系我，通过 [email](anotherworldofworld@gmail.com) 联系我，或者创建一个 [issue](https://github.com/0xAX/linux-insides/issues/new).

**友情提示：英语不是我的母语，对于译文给您带来了的不便我感到非常抱歉。如果您发现任何错误请给我发送PR到 [linux-insides](https://github.com/0xAX/linux-insides)。**

链接
--------------------------------------------------------------------------------

* [spinlock](https://en.wikipedia.org/wiki/Spinlock)
* [interrupt](https://en.wikipedia.org/wiki/Interrupt)
* [interrupt handler](https://en.wikipedia.org/wiki/Interrupt_handler)
* [API](https://en.wikipedia.org/wiki/Application_programming_interface)
* [Test and Set](https://en.wikipedia.org/wiki/Test-and-set)
* [MCS](http://www.cs.rochester.edu/~scott/papers/1991_TOCS_synch.pdf)
* [per-cpu variables](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html)
* [atomic instruction](https://en.wikipedia.org/wiki/Linearizability)
* [CMPXCHG instruction](http://x86.renejeschke.de/html/file_module_x86_id_41.html)
* [LOCK instruction](http://x86.renejeschke.de/html/file_module_x86_id_159.html)
* [NOP instruction](https://en.wikipedia.org/wiki/NOP)
* [PREFETCHW instruction](http://www.felixcloutier.com/x86/PREFETCHW.html)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [Previous part](https://xinqiu.gitbooks.io/linux-insides-cn/content/SyncPrim/linux-sync-1.html)
