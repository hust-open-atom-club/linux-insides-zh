CPU masks
================================================================================

介绍
--------------------------------------------------------------------------------

`Cpumasks` 是Linux内核提供的保存系统CPU信息的特殊方法。包含 `Cpumasks` 操作 API 相关的源码和头文件：

* [include/linux/cpumask.h](https://github.com/torvalds/linux/blob/master/include/linux/cpumask.h)
* [lib/cpumask.c](https://github.com/torvalds/linux/blob/master/lib/cpumask.c)
* [kernel/cpu.c](https://github.com/torvalds/linux/blob/master/kernel/cpu.c)

正如 [include/linux/cpumask.h](https://github.com/torvalds/linux/blob/master/include/linux/cpumask.h) 注释：Cpumasks 提供了代表系统中 CPU 集合的位图，一位放置一个 CPU 序号。我们已经在 [Kernel entry point](http://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html) 部分，函数 `boot_cpu_init` 中看到了一点 cpumask。这个函数将第一个启动的 cpu 上线、激活等等……

```C
set_cpu_online(cpu, true);
set_cpu_active(cpu, true);
set_cpu_present(cpu, true);
set_cpu_possible(cpu, true);
```

`set_cpu_possible` 是一个在系统启动时任意时刻都可插入的 cpu ID 集合。`cpu_present` 代表了当前插入的 CPUs。`cpu_online` 是 `cpu_present` 的子集，表示可调度的 CPUs。这些掩码依赖于 `CONFIG_HOTPLUG_CPU` 配置选项，以及 `possible == present` 和 `active == online` 选项是否被禁用。这些函数的实现很相似，检测第二个参数，如果为 `true`，就调用 `cpumask_set_cpu` ，否则调用 `cpumask_clear_cpu`。

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

`NR_CPUS` 宏表示的是系统中 CPU 的数目，且依赖于在 [include/linux/threads.h](https://github.com/torvalds/linux/blob/master/include/linux/threads.h) 中定义的 `CONFIG_NR_CPUS` 宏，看起来像这样：

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

每次都是返回 `1`。我们需要它只是因为：编译时检测一个给定的 `bitmap` 是一个位图，换句话说，它检测一个 `bitmap` 是否有 `unsigned long *` 类型。因此我们传递 `cpu_possible_bits` 给宏 `to_cpumask` ，将 `unsigned long` 数组转换为 `struct cpumask *`。

cpumask API
--------------------------------------------------------------------------------

因为我们可以用其中一个方法来定义 cpumask，Linux 内核提供了 API 来处理 cpumask。我们来研究下其中一个函数，例如 `set_cpu_online`，这个函数有两个参数：

* CPU 数目;
* CPU 状态;

这个函数的实现如下所示：

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

该函数首先检测第二个 `state` 参数并调用依赖它的 `cpumask_set_cpu` 或 `cpumask_clear_cpu`。这里我们可以看到在中 `cpumask_set_cpu` 的第二个参数转换为 `struct cpumask *`。在我们的例子中是位图 `cpu_online_bits`，定义如下：

```C
static DECLARE_BITMAP(cpu_online_bits, CONFIG_NR_CPUS) __read_mostly;
```

函数 `cpumask_set_cpu` 仅调用了一次 `set_bit` 函数：

```C
static inline void cpumask_set_cpu(unsigned int cpu, struct cpumask *dstp)
{
        set_bit(cpumask_check(cpu), cpumask_bits(dstp));
}
```

`set_bit` 函数也有两个参数，设置了一个给定位（第一个参数）的内存（第二个参数或 `cpu_online_bits` 位图）。这儿我们可以看到在调用 `set_bit` 之前，它的两个参数会传递给

* cpumask_check;
* cpumask_bits.

让我们细看下这两个宏。第一个 `cpumask_check` 在我们的例子里没做任何事，只是返回了给的参数。第二个 `cpumask_bits` 只是返回了传入 `struct cpumask *` 结构的 `bits` 域。

```C
#define cpumask_bits(maskp) ((maskp)->bits)
```

现在让我们看下 `set_bit` 的实现：

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

这个函数看着吓人，但它没有看起来那么难。首先传参 `nr` 或者说位数给 `IS_IMMEDIATE` 宏，该宏调用了 GCC 内联函数 `__builtin_constant_p`：

```C
#define IS_IMMEDIATE(nr)    (__builtin_constant_p(nr))
```

`__builtin_constant_p` 检查给定参数是否编译时恒定变量。因为我们的 `cpu` 不是编译时恒定变量，将会执行 `else` 分支：

```C
asm volatile(LOCK_PREFIX "bts %1,%0" : BITOP_ADDR(addr) : "Ir" (nr) : "memory");
```

让我们试着一步一步来理解它如何工作的：

`LOCK_PREFIX` 是个 x86 `lock` 指令。这个指令告诉 CPU 当指令执行时占据系统总线。这允许 CPU 同步内存访问，防止多核（或多设备 - 比如 DMA 控制器）并发访问同一个内存cell。

`BITOP_ADDR` 转换给定参数至 `(*(volatile long *)` 并且加了 `+m` 约束。`+` 意味着这个操作数对于指令是可读写的。`m` 显示这是一个内存操作数。`BITOP_ADDR` 定义如下：

```C
#define BITOP_ADDR(x) "+m" (*(volatile long *) (x))
```

接下来是 `memory`。它告诉编译器汇编代码执行内存读或写到某些项，而不是那些输入或输出操作数（例如，访问指向输出参数的内存）。

`Ir` - 寄存器操作数。

`bts` 指令设置一个位字符串的给定位，存储给定位的值到 `CF` 标志位。所以我们传递 cpu 号，我们的例子中为 0，给 `set_bit` 并且执行后，其设置了在 `cpu_online_bits` cpumask 中的 0 位。这意味着第一个 cpu 此时上线了。

当然，除了 `set_cpu_*` API 外，cpumask 提供了其它 cpumasks 操作的 API。让我们简短看下。

附加的 cpumask API
--------------------------------------------------------------------------------

cpumaks 提供了一系列宏来得到不同状态 CPUs 序号。例如：

```C
#define num_online_cpus()	cpumask_weight(cpu_online_mask)
```

这个宏返回了 `online` CPUs 数量。它读取 `cpu_online_mask` 位图并调用了 `cpumask_weight` 函数。`cpumask_weight` 函数使用两个参数调用了一次 `bitmap_weight` 函数：

* cpumask bitmap;
* `nr_cpumask_bits` - 在我们的例子中就是 `NR_CPUS`。

```C
static inline unsigned int cpumask_weight(const struct cpumask *srcp)
{
	return bitmap_weight(cpumask_bits(srcp), nr_cpumask_bits);
}
```

并计算给定位图的位数。除了 `num_online_cpus`，cpumask还提供了所有 CPU 状态的宏：

* num_possible_cpus;
* num_active_cpus;
* cpu_online;
* cpu_possible.

等等。

除了 Linux 内核提供的下述操作 `cpumask` 的 API：

* `for_each_cpu` - 遍历一个mask的所有 cpu;
* `for_each_cpu_not` - 遍历所有补集的 cpu;
* `cpumask_clear_cpu` - 清除一个 cpumask 的 cpu;
* `cpumask_test_cpu` - 测试一个 mask 中的 cpu;
* `cpumask_setall` - 设置 mask 的所有 cpu;
* `cpumask_size` - 返回分配 'struct cpumask' 字节数大小;

还有很多。

链接
--------------------------------------------------------------------------------

* [cpumask documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)
