CPU masks
================================================================================

介绍
--------------------------------------------------------------------------------

`Cpumasks` 是Linux内核提供的保存系统CPU信息的特殊方法。包含 `Cpumasks` 操作 API 相关的源码和头文件：

* [include/linux/cpumask.h](https://github.com/torvalds/linux/blob/master/include/linux/cpumask.h)
* [lib/cpumask.c](https://github.com/torvalds/linux/blob/master/lib/cpumask.c)
* [kernel/cpu.c](https://github.com/torvalds/linux/blob/master/kernel/cpu.c)

正如 [include/linux/cpumask.h](https://github.com/torvalds/linux/blob/master/include/linux/cpumask.h) 注释：Cpumasks 提供了代表系统中 CPU 集合的位图，一位放置一个 CPU 序号。我们已经在 [Kernel entry point](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html) 部分，函数 `boot_cpu_init` 中看到了一点cpumask。这个函数将第一个启动的 cpu 上线、激活等等……

```C
set_cpu_online(cpu, true);
set_cpu_active(cpu, true);
set_cpu_present(cpu, true);
set_cpu_possible(cpu, true);
```

`set_cpu_possible` 是一个在系统启动时任意时刻都可插入的 cpu ID 集合。`cpu_present` 代表了当前插入的 CPUs。`cpu_online` 是 `cpu_present` 的子集，表示可调度的 CPUs。这些掩码依赖于 `CONFIG_HOTPLUG_CPU` 配置选项，以及 `possible == present` 和 `active == online`选项是否被禁用。这些函数的实现很相似，检测第二个参数，如果为 `true`，就调用 `cpumask_set_cpu` ，否则调用 `cpumask_clear_cpu`。

有两种方法创建 `cpumask`。第一种是用 `cpumask_t`。定义如下：

```C
typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t;
```

它封装了 `cpumask` 结构，其包含了一个位掩码 `bits` 字段。`DECLARE_BITMAP` 宏有两个参数：

* bitmap name;
* number of bits.

并以给定名称创建了一个 `unsigned long` 数组。它的实现非常简单：

```C
#define DECLARE_BITMAP(name,bits) \
        unsigned long name[BITS_TO_LONGS(bits)]
```

其中 `BITS_TO_LONGS`：

```C
#define BITS_TO_LONGS(nr)       DIV_ROUND_UP(nr, BITS_PER_BYTE * sizeof(long))
#define DIV_ROUND_UP(n,d) (((n) + (d) - 1) / (d))
```

因为我们专注于 `x86_64` 架构，`unsigned long` 是8字节大小，因此我们的数组仅包含一个元素：

```
(((8) + (8) - 1) / (8)) = 1
```

`NR_CPUS` 宏表示系统中的 CPU，依赖于在 [include/linux/threads.h](https://github.com/torvalds/linux/blob/master/include/linux/threads.h) 中定义的 `CONFIG_NR_CPUS` 宏，看起来像这样：

```C
#ifndef CONFIG_NR_CPUS
        #define CONFIG_NR_CPUS  1
#endif

#define NR_CPUS         CONFIG_NR_CPUS
```

第二种定义 cpumask 的方法是直接使用宏 `DECLARE_BITMAP` 和 `to_cpumask` 宏，后者将给定的位图转化为 `struct cpumask *`：

```C
#define to_cpumask(bitmap)                                              \
        ((struct cpumask *)(1 ? (bitmap)                                \
                            : (void *)sizeof(__check_is_bitmap(bitmap))))
```

可以看到这里的三目运算符每次总是 `true`。`__check_is_bitmap` 内联函数定义为：

```C
static inline int __check_is_bitmap(const unsigned long *bitmap)
{
        return 1;
}
```

每次都是返回 `1`。我们需要它仅一个目的：编译时检测一个给定的 `bitmap` 是一个位图，换句话说，它检测一个 `bitmap` 有 `unsigned long *` 类型。因此我们传递 `cpu_possible_bits` 给宏 `to_cpumask` ，将 `unsigned long` 数组转换为 `struct cpumask *`。

cpumask API
--------------------------------------------------------------------------------

因为我们可以用上述方法之一来定义 cpumask，Linux 内核提供了 API 来操作 cpumask。我们看下上述函数之一，例如 `set_cpu_online`，这个函数有两个参数：

* CPU 数目;
* CPU 状态;

这个函数的实现是这样：

```C
void set_cpu_online(unsigned int cpu, bool online)
{
	if (online) {
		cpumask_set_cpu(cpu, to_cpumask(cpu_online_bits));
		cpumask_set_cpu(cpu, to_cpumask(cpu_active_bits));
	} else {
		cpumask_clear_cpu(cpu, to_cpumask(cpu_online_bits));
	}
}
```

首先检测第二个 `state` 参数并调用依赖它的 `cpumask_set_cpu` 或 `cpumask_clear_cpu`。这里我们可以看到在中 `cpumask_set_cpu` 的第二个参数转换为 `struct cpumask *`。在我们的例子中是位图 `cpu_online_bits`，定义如下：

```C
static DECLARE_BITMAP(cpu_online_bits, CONFIG_NR_CPUS) __read_mostly;
```

函数 `cpumask_set_cpu` 调用了一次 `set_bit` 函数：

```C
static inline void cpumask_set_cpu(unsigned int cpu, struct cpumask *dstp)
{
        set_bit(cpumask_check(cpu), cpumask_bits(dstp));
}
```

The `set_bit` function takes two parameters too, and sets a given bit (first parameter) in the memory (second parameter or `cpu_online_bits` bitmap). We can see here that before `set_bit` will be called, its two parameters will be passed to the

* cpumask_check;
* cpumask_bits.

Let's consider these two macros. First if `cpumask_check` does nothing in our case and just returns given parameter. The second `cpumask_bits` just returns the `bits` field from the given `struct cpumask *` structure:

```C
#define cpumask_bits(maskp) ((maskp)->bits)
```

Now let's look on the `set_bit` implementation:

```C
 static __always_inline void
 set_bit(long nr, volatile unsigned long *addr)
 {
         if (IS_IMMEDIATE(nr)) {
                asm volatile(LOCK_PREFIX "orb %1,%0"
                        : CONST_MASK_ADDR(nr, addr)
                        : "iq" ((u8)CONST_MASK(nr))
                        : "memory");
        } else {
                asm volatile(LOCK_PREFIX "bts %1,%0"
                        : BITOP_ADDR(addr) : "Ir" (nr) : "memory");
        }
 }
```

This function looks scary, but it is not so hard as it seems. First of all it passes `nr` or number of the bit to the `IS_IMMEDIATE` macro which just calls the GCC internal `__builtin_constant_p` function:

```C
#define IS_IMMEDIATE(nr)    (__builtin_constant_p(nr))
```

`__builtin_constant_p` checks that given parameter is known constant at compile-time. As our `cpu` is not compile-time constant, the `else` clause will be executed:

```C
asm volatile(LOCK_PREFIX "bts %1,%0" : BITOP_ADDR(addr) : "Ir" (nr) : "memory");
```

Let's try to understand how it works step by step:

`LOCK_PREFIX` is a x86 `lock` instruction. This instruction tells the cpu to occupy the system bus while the instruction(s) will be executed. This allows the CPU to synchronize memory access, preventing simultaneous access of multiple processors (or devices - the DMA controller for example) to one memory cell.

`BITOP_ADDR` casts the given parameter to the `(*(volatile long *)` and adds `+m` constraints. `+` means that this operand is both read and written by the instruction. `m` shows that this is a memory operand. `BITOP_ADDR` is defined as:

```C
#define BITOP_ADDR(x) "+m" (*(volatile long *) (x))
```

Next is the `memory` clobber. It tells the compiler that the assembly code performs memory reads or writes to items other than those listed in the input and output operands (for example, accessing the memory pointed to by one of the input parameters).

`Ir` - immediate register operand.


The `bts` instruction sets a given bit in a bit string and stores the value of a given bit in the `CF` flag. So we passed the cpu number which is zero in our case and after `set_bit` is executed, it sets the zero bit in the `cpu_online_bits` cpumask. It means that the first cpu is online at this moment.

Besides the `set_cpu_*` API, cpumask of course provides another API for cpumasks manipulation. Let's consider it in short.

Additional cpumask API
--------------------------------------------------------------------------------

cpumask provides a set of macros for getting the numbers of CPUs in various states. For example:

```C
#define num_online_cpus()	cpumask_weight(cpu_online_mask)
```

This macro returns the amount of `online` CPUs. It calls the `cpumask_weight` function with the `cpu_online_mask` bitmap (read about it). The`cpumask_weight` function makes one call of the `bitmap_weight` function with two parameters:

* cpumask bitmap;
* `nr_cpumask_bits` - which is `NR_CPUS` in our case.

```C
static inline unsigned int cpumask_weight(const struct cpumask *srcp)
{
	return bitmap_weight(cpumask_bits(srcp), nr_cpumask_bits);
}
```

and calculates the number of bits in the given bitmap. Besides the `num_online_cpus`, cpumask provides macros for the all CPU states:

* num_possible_cpus;
* num_active_cpus;
* cpu_online;
* cpu_possible.

and many more.

Besides that the Linux kernel provides the following API for the manipulation of `cpumask`:

* `for_each_cpu` - iterates over every cpu in a mask;
* `for_each_cpu_not` - iterates over every cpu in a complemented mask;
* `cpumask_clear_cpu` - clears a cpu in a cpumask;
* `cpumask_test_cpu` - tests a cpu in a mask;
* `cpumask_setall` - set all cpus in a mask;
* `cpumask_size` - returns size to allocate for a 'struct cpumask' in bytes;

and many many more...

Links
--------------------------------------------------------------------------------

* [cpumask documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)
