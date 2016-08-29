Linux 内核里的数据结构——位数组
================================================================================

Linux 内核中的位数组和位操作
--------------------------------------------------------------------------------

除了不同的基于[链式](https://en.wikipedia.org/wiki/Linked_data_structure)和[树](https://en.wikipedia.org/wiki/Tree_%28data_structure%29)的数据结构以外，Linux 内核也为[位数组](https://en.wikipedia.org/wiki/Bit_array)（或称为位图（bitmap））提供了 [API](https://en.wikipedia.org/wiki/Application_programming_interface)。位数组在 Linux 内核里被广泛使用，并且在以下的源代码文件中包含了与这样的结构搭配使用的通用 `API`：

* [lib/bitmap.c](https://github.com/torvalds/linux/blob/master/lib/bitmap.c)
* [include/linux/bitmap.h](https://github.com/torvalds/linux/blob/master/include/linux/bitmap.h)

除了这两个文件之外，还有体系结构特定的头文件，它们为特定的体系结构提供优化的位操作。我们将探讨 [x86_64](https://en.wikipedia.org/wiki/X86-64) 体系结构，因此在我们的例子里，它会是

* [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/bitops.h)

头文件。正如我上面所写的，`位图`在 Linux 内核中被广泛地使用。例如，`位数组`常常用于保存一组在线/离线处理器，以便系统支持[热插拔](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)的 CPU（你可以在 [cpumasks](https://0xax.gitbooks.io/linux-insides/content/Concepts/cpumask.html) 部分阅读更多相关知识 ），一个位数组（bit array）可以在 Linux 内核初始化等期间保存一组已分配的[中断处理](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)。

因此，本部分的主要目的是了解位数组（bit array）是如何在 Linux 内核中实现的。让我们现在开始吧。

位数组声明
================================================================================

在我们开始查看`位图`操作的 `API` 之前，我们必须知道如何在 Linux 内核中声明它。有两种声明位数组的通用方法。第一种简单的声明一个位数组的方法是，定义一个 `unsigned long` 的数组，例如：

```C
unsigned long my_bitmap[8]
```

第二种方法，是使用 `DECLARE_BITMAP` 宏，它定义于 [include/linux/types.h](https://github.com/torvalds/linux/blob/master/include/linux/types.h) 头文件：

```C
#define DECLARE_BITMAP(name,bits) \
    unsigned long name[BITS_TO_LONGS(bits)]
```

我们可以看到 `DECLARE_BITMAP` 宏使用两个参数：

* `name` - 位图名称;
* `bits` - 位图中位数;

并且只是使用 `BITS_TO_LONGS(bits)` 元素展开 `unsigned long` 数组的定义。 `BITS_TO_LONGS` 宏将一个给定的位数转换为 `long` 的个数，换言之，就是计算 `bits` 中有多少个 `8` 字节元素：

```C
#define BITS_PER_BYTE           8
#define DIV_ROUND_UP(n,d) (((n) + (d) - 1) / (d))
#define BITS_TO_LONGS(nr)       DIV_ROUND_UP(nr, BITS_PER_BYTE * sizeof(long))
```

因此，例如 `DECLARE_BITMAP(my_bitmap, 64)` 将产生：

```python
>>> (((64) + (64) - 1) / (64))
1
```

与：

```C
unsigned long my_bitmap[1];
```

在能够声明一个位数组之后，我们便可以使用它了。

体系结构特定的位操作
================================================================================

我们已经看了上面提及的一对源文件和头文件，它们提供了位数组操作的 [API](https://en.wikipedia.org/wiki/Application_programming_interface)。其中重要且广泛使用的位数组 API 是体系结构特定的且位于已提及的头文件中 [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/bitops.h)。

首先让我们查看两个最重要的函数：

* `set_bit`;
* `clear_bit`.

我认为没有必要解释这些函数的作用。从它们的名字来看，这已经很清楚了。让我们直接查看它们的实现。如果你浏览 [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/bitops.h) 头文件，你将会注意到这些函数中的每一个都有[原子性](https://en.wikipedia.org/wiki/Linearizability)和非原子性两种变体。在我们开始深入这些函数的实现之前，首先，我们必须了解一些有关原子（atomic）操作的知识。

简而言之，原子操作保证两个或以上的操作不会并发地执行同一数据。`x86` 体系结构提供了一系列原子指令，例如， [xchg](http://x86.renejeschke.de/html/file_module_x86_id_328.html)、[cmpxchg](http://x86.renejeschke.de/html/file_module_x86_id_41.html) 等指令。除了原子指令，一些非原子指令可以在 [lock](http://x86.renejeschke.de/html/file_module_x86_id_159.html) 指令的帮助下具有原子性。现在你已经对原子操作有了足够的了解，我们可以接着探讨 `set_bit` 和 `clear_bit` 函数的实现。

我们先考虑函数的非原子性（non-atomic）变体。非原子性的 `set_bit` 和 `clear_bit` 的名字以双下划线开始。正如我们所知道的，所有这些函数都定义于 [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/bitops.h) 头文件，并且第一个函数就是 `__set_bit`:

```C
static inline void __set_bit(long nr, volatile unsigned long *addr)
{
	asm volatile("bts %1,%0" : ADDR : "Ir" (nr) : "memory");
}
```

正如我们所看到的，它使用了两个参数：

* `nr` - 位数组中的位号（LCTT 译注：从 0开始）
* `addr` - 我们需要置位的位数组地址

注意，`addr` 参数使用 `volatile` 关键字定义，以告诉编译器给定地址指向的变量可能会被修改。 `__set_bit` 的实现相当简单。正如我们所看到的，它仅包含一行[内联汇编代码](https://en.wikipedia.org/wiki/Inline_assembler)。在我们的例子中，我们使用 [bts](http://x86.renejeschke.de/html/file_module_x86_id_25.html) 指令，从位数组中选出一个第一操作数（我们的例子中的 `nr`）所指定的位，存储选出的位的值到 [CF](https://en.wikipedia.org/wiki/FLAGS_register) 标志寄存器并设置该位（LCTT 译注：即 `nr` 指定的位置为 1）。

注意，我们了解了 `nr` 的用法，但这里还有一个参数 `addr` 呢！你或许已经猜到秘密就在 `ADDR`。 `ADDR` 是一个定义在同一个头文件中的宏，它展开为一个包含给定地址和 `+m` 约束的字符串：

```C
#define ADDR				BITOP_ADDR(addr)
#define BITOP_ADDR(x) "+m" (*(volatile long *) (x))
```

除了 `+m` 之外，在 `__set_bit` 函数中我们可以看到其他约束。让我们查看并试着理解它们所表示的意义：

* `+m` - 表示内存操作数，这里的 `+` 表明给定的操作数为输入输出操作数；
* `I` - 表示整型常量；
* `r` - 表示寄存器操作数

除了这些约束之外，我们也能看到 `memory` 关键字，其告诉编译器这段代码会修改内存中的变量。到此为止，现在我们看看相同的原子性（atomic）变体函数。它看起来比非原子性（non-atomic）变体更加复杂：

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

（LCTT 译注：BITOP_ADDR 的定义为：`#define BITOP_ADDR(x) "=m" (*(volatile long *) (x))`，ORB 为字节按位或。）

首先注意，这个函数使用了与 `__set_bit` 相同的参数集合，但额外地使用了 `__always_inline` 属性标记。 `__always_inline` 是一个定义于 [include/linux/compiler-gcc.h](https://github.com/torvalds/linux/blob/master/include/linux/compiler-gcc.h) 的宏，并且只是展开为 `always_inline` 属性：

```C
#define __always_inline inline __attribute__((always_inline))
```

其意味着这个函数总是内联的，以减少 Linux 内核映像的大小。现在让我们试着了解下 `set_bit` 函数的实现。首先我们在 `set_bit` 函数的开头检查给定的位的数量。`IS_IMMEDIATE` 宏定义于相同的[头文件](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/bitops.h)，并展开为 [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection) 内置函数的调用：

```C
#define IS_IMMEDIATE(nr)		(__builtin_constant_p(nr))
```

如果给定的参数是编译期已知的常量，`__builtin_constant_p` 内置函数则返回 `1`，其他情况返回 `0`。假若给定的位数是编译期已知的常量，我们便无须使用效率低下的 `bts` 指令去设置位。我们可以只需在给定地址指向的字节上执行 [按位或](https://en.wikipedia.org/wiki/Bitwise_operation#OR) 操作，其字节包含给定的位，掩码位数表示高位为 `1`，其他位为 0 的掩码。在其他情况下，如果给定的位号不是编译期已知常量，我们便做和 `__set_bit` 函数一样的事。`CONST_MASK_ADDR` 宏：

```C
#define CONST_MASK_ADDR(nr, addr)	BITOP_ADDR((void *)(addr) + ((nr)>>3))
```

展开为带有到包含给定位的字节偏移的给定地址，例如，我们拥有地址 `0x1000` 和位号 `0x9`。因为 `0x9` 代表 `一个字节 + 一位`，所以我们的地址是 `addr + 1`:

```python
>>> hex(0x1000 + (0x9 >> 3))
'0x1001'
```

`CONST_MASK` 宏将我们给定的位号表示为字节，位号对应位为高位 `1`，其他位为 `0`：

```C
#define CONST_MASK(nr)			(1 << ((nr) & 7))
```

```python
>>> bin(1 << (0x9 & 7))
'0b10'
```

最后，我们应用 `按位或` 运算到这些变量上面，因此，假如我们的地址是 `0x4097` ，并且我们需要置位号为 `9` 的位为 1：

```python
>>> bin(0x4097)
'0b100000010010111'
>>> bin((0x4097 >> 0x9) | (1 << (0x9 & 7)))
'0b100010'
```

`第 9 位` 将会被置位。（LCTT 译注：这里的 9 是从 0 开始计数的，比如0010，按照作者的意思，其中的 1 是第 1 位）

注意，所有这些操作使用 `LOCK_PREFIX` 标记，其展开为 [lock](http://x86.renejeschke.de/html/file_module_x86_id_159.html) 指令，保证该操作的原子性。

正如我们所知，除了 `set_bit` 和 `__set_bit` 操作之外，Linux 内核还提供了两个功能相反的函数，在原子性和非原子性的上下文中清位。它们是 `clear_bit` 和 `__clear_bit`。这两个函数都定义于同一个[头文件](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/bitops.h) 并且使用相同的参数集合。不仅参数相似，一般而言，这些函数与  `set_bit` 和 `__set_bit` 也非常相似。让我们查看非原子性 `__clear_bit` 的实现吧： 

```C
static inline void __clear_bit(long nr, volatile unsigned long *addr)
{
	asm volatile("btr %1,%0" : ADDR : "Ir" (nr));
}
```

没错，正如我们所见，`__clear_bit` 使用相同的参数集合，并包含极其相似的内联汇编代码块。它只是使用 [btr](http://x86.renejeschke.de/html/file_module_x86_id_24.html) 指令替换了 `bts`。正如我们从函数名所理解的一样，通过给定地址，它清除了给定的位。`btr` 指令表现得像 `bts`（LCTT 译注：原文这里为 btr，可能为笔误，修正为 bts）。该指令选出第一操作数所指定的位，存储它的值到 `CF` 标志寄存器，并且清除第二操作数指定的位数组中的对应位。

`__clear_bit` 的原子性变体为 `clear_bit`：

```C
static __always_inline void
clear_bit(long nr, volatile unsigned long *addr)
{
	if (IS_IMMEDIATE(nr)) {
		asm volatile(LOCK_PREFIX "andb %1,%0"
			: CONST_MASK_ADDR(nr, addr)
			: "iq" ((u8)~CONST_MASK(nr)));
	} else {
		asm volatile(LOCK_PREFIX "btr %1,%0"
			: BITOP_ADDR(addr)
			: "Ir" (nr));
	}
}
```

并且正如我们所看到的，它与 `set_bit` 非常相似，只有两处不同。第一处差异为 `clear_bit` 使用 `btr` 指令来清位，而 `set_bit` 使用 `bts` 指令来置位。第二处差异为 `clear_bit` 使用否定的位掩码和 `按位与` 在给定的字节上置位，而 `set_bit` 使用 `按位或` 指令。

到此为止，我们可以在任意位数组置位和清位了，我们将看看位掩码上的其他操作。

在 Linux 内核中对位数组最广泛使用的操作是设置和清除位，但是除了这两个操作外，位数组上其他操作也是非常有用的。Linux 内核里另一种广泛使用的操作是知晓位数组中一个给定的位是否被置位。我们能够通过 `test_bit` 宏的帮助实现这一功能。这个宏定义于 [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/bitops.h) 头文件，并根据位号分别展开为 `constant_test_bit` 或 `variable_test_bit` 调用。

```C
#define test_bit(nr, addr)			\
	(__builtin_constant_p((nr))                 \
	 ? constant_test_bit((nr), (addr))	        \
	 : variable_test_bit((nr), (addr)))
```

因此，如果 `nr` 是编译期已知常量，`test_bit` 将展开为 `constant_test_bit` 函数的调用，而其他情况则为 `variable_test_bit`。现在让我们看看这些函数的实现，让我们从 `variable_test_bit` 开始看起：  

```C
static inline int variable_test_bit(long nr, volatile const unsigned long *addr)
{
	int oldbit;

	asm volatile("bt %2,%1\n\t"
		     "sbb %0,%0"
		     : "=r" (oldbit)
		     : "m" (*(unsigned long *)addr), "Ir" (nr));

	return oldbit;
}
```

`variable_test_bit` 函数使用了与 `set_bit` 及其他函数使用的相似的参数集合。我们也可以看到执行 [bt](http://x86.renejeschke.de/html/file_module_x86_id_22.html) 和 [sbb](http://x86.renejeschke.de/html/file_module_x86_id_286.html) 指令的内联汇编代码。`bt` （或称 `bit test`）指令从第二操作数指定的位数组选出第一操作数指定的一个指定位，并且将该位的值存进标志寄存器的 [CF](https://en.wikipedia.org/wiki/FLAGS_register) 位。第二个指令 `sbb` 从第二操作数中减去第一操作数，再减去 `CF` 的值。因此，这里将一个从给定位数组中的给定位号的值写进标志寄存器的 `CF` 位，并且执行 `sbb` 指令计算： `00000000 - CF`，并将结果写进 `oldbit` 变量。

`constant_test_bit` 函数做了和我们在 `set_bit` 所看到的一样的事：

```C
static __always_inline int constant_test_bit(long nr, const volatile unsigned long *addr)
{
	return ((1UL << (nr & (BITS_PER_LONG-1))) &
		(addr[nr >> _BITOPS_LONG_SHIFT])) != 0;
}
```

它生成了一个位号对应位为高位 `1`，而其他位为 `0` 的字节（正如我们在 `CONST_MASK` 所看到的），并将 [按位与](https://en.wikipedia.org/wiki/Bitwise_operation#AND) 应用于包含给定位号的字节。

下一个被广泛使用的位数组相关操作是改变一个位数组中的位。为此，Linux 内核提供了两个辅助函数：

* `__change_bit`;
* `change_bit`.

你可能已经猜测到，就拿 `set_bit` 和 `__set_bit` 例子说，这两个变体分别是原子和非原子版本。首先，让我们看看 `__change_bit` 函数的实现：

```C
static inline void __change_bit(long nr, volatile unsigned long *addr)
{
    asm volatile("btc %1,%0" : ADDR : "Ir" (nr));
}
```

相当简单，不是吗？ `__change_bit` 的实现和 `__set_bit` 一样，只是我们使用 [btc](http://x86.renejeschke.de/html/file_module_x86_id_23.html) 替换 `bts` 指令而已。 该指令从一个给定位数组中选出一个给定位，将该为位的值存进 `CF` 并使用求反操作改变它的值，因此值为 `1` 的位将变为 `0`，反之亦然：

```python
>>> int(not 1)
0
>>> int(not 0)
1
```

`__change_bit` 的原子版本为 `change_bit` 函数：

```C
static inline void change_bit(long nr, volatile unsigned long *addr)
{
	if (IS_IMMEDIATE(nr)) {
		asm volatile(LOCK_PREFIX "xorb %1,%0"
			: CONST_MASK_ADDR(nr, addr)
			: "iq" ((u8)CONST_MASK(nr)));
	} else {
		asm volatile(LOCK_PREFIX "btc %1,%0"
			: BITOP_ADDR(addr)
			: "Ir" (nr));
	}
}
```

它和 `set_bit` 函数很相似，但也存在两点不同。第一处差异为 `xor` 操作而不是 `or`。第二处差异为 `btc`（ LCTT 译注：原文为 `bts`，为作者笔误） 而不是 `bts`。

目前，我们了解了最重要的体系特定的位数组操作，是时候看看一般的位图 API 了。

通用位操作
================================================================================

除了 [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/bitops.h) 中体系特定的 API 外，Linux 内核提供了操作位数组的通用 API。正如我们本部分开头所了解的一样，我们可以在 [include/linux/bitmap.h](https://github.com/torvalds/linux/blob/master/include/linux/bitmap.h) 头文件和 [lib/bitmap.c](https://github.com/torvalds/linux/blob/master/lib/bitmap.c)  源文件中找到它。但在查看这些源文件之前，我们先看看 [include/linux/bitops.h](https://github.com/torvalds/linux/blob/master/include/linux/bitops.h) 头文件，其提供了一系列有用的宏，让我们看看它们当中一部分。

首先我们看看以下 4 个 宏：

* `for_each_set_bit`
* `for_each_set_bit_from`
* `for_each_clear_bit`
* `for_each_clear_bit_from`

所有这些宏都提供了遍历位数组中某些位集合的迭代器。第一个宏迭代那些被置位的位。第二个宏也是一样，但它是从某一个确定的位开始。最后两个宏做的一样，但是迭代那些被清位的位。让我们看看 `for_each_set_bit` 宏：

```C
#define for_each_set_bit(bit, addr, size) \
	for ((bit) = find_first_bit((addr), (size));		\
	     (bit) < (size);					\
	     (bit) = find_next_bit((addr), (size), (bit) + 1))
```

正如我们所看到的，它使用了三个参数，并展开为一个循环，该循环从作为 `find_first_bit` 函数返回结果的第一个置位开始，到小于给定大小的最后一个置位为止。

除了这四个宏， [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/bitops.h) 也提供了 `64-bit` 或 `32-bit` 变量循环的 API 等等。

下一个 [头文件](https://github.com/torvalds/linux/blob/master/include/linux/bitmap.h) 提供了操作位数组的 API。例如，它提供了以下两个函数： 

* `bitmap_zero`;
* `bitmap_fill`.

它们分别可以清除一个位数组和用 `1` 填充位数组。让我们看看 `bitmap_zero` 函数的实现：

```C
static inline void bitmap_zero(unsigned long *dst, unsigned int nbits)
{
	if (small_const_nbits(nbits))
		*dst = 0UL;
	else {
		unsigned int len = BITS_TO_LONGS(nbits) * sizeof(unsigned long);
		memset(dst, 0, len);
	}
}
```

首先我们可以看到对 `nbits` 的检查。 `small_const_nbits` 是一个定义在同一个[头文件](https://github.com/torvalds/linux/blob/master/include/linux/bitmap.h) 的宏：

```C
#define small_const_nbits(nbits) \
	(__builtin_constant_p(nbits) && (nbits) <= BITS_PER_LONG)
```

正如我们可以看到的，它检查 `nbits` 是否为编译期已知常量，并且其值不超过 `BITS_PER_LONG` 或 `64`。如果位数目没有超过一个 `long` 变量的位数，我们可以仅仅设置为 0。在其他情况，我们需要计算有多少个需要填充位数组的 `long` 变量并且使用 [memset](http://man7.org/linux/man-pages/man3/memset.3.html) 进行填充。

`bitmap_fill` 函数的实现和 `biramp_zero` 函数很相似，除了我们需要在给定的位数组中填写 `0xff` 或 `0b11111111`：

```C
static inline void bitmap_fill(unsigned long *dst, unsigned int nbits)
{
	unsigned int nlongs = BITS_TO_LONGS(nbits);
	if (!small_const_nbits(nbits)) {
		unsigned int len = (nlongs - 1) * sizeof(unsigned long);
		memset(dst, 0xff,  len);
	}
	dst[nlongs - 1] = BITMAP_LAST_WORD_MASK(nbits);
}
```

除了 `bitmap_fill` 和 `bitmap_zero`，[include/linux/bitmap.h](https://github.com/torvalds/linux/blob/master/include/linux/bitmap.h) 头文件也提供了和 `bitmap_zero` 很相似的 `bitmap_copy`，只是仅仅使用 [memcpy](http://man7.org/linux/man-pages/man3/memcpy.3.html) 而不是  [memset](http://man7.org/linux/man-pages/man3/memset.3.html) 这点差异而已。它也提供了位数组的按位操作，像 `bitmap_and`, `bitmap_or`, `bitamp_xor`等等。我们不会探讨这些函数的实现了，因为如果你理解了本部分的所有内容，这些函数的实现是很容易理解的。无论如何，如果你对这些函数是如何实现的感兴趣，你可以打开并研究 [include/linux/bitmap.h](https://github.com/torvalds/linux/blob/master/include/linux/bitmap.h) 头文件。

本部分到此为止。

注： 本文由 [LCTT](https://github.com/LCTT/TranslateProject) 原创翻译，[Linux中国](https://linux.cn/) 荣誉推出

链接
================================================================================

* [bitmap](https://en.wikipedia.org/wiki/Bit_array)
* [linked data structures](https://en.wikipedia.org/wiki/Linked_data_structure)
* [tree data structures](https://en.wikipedia.org/wiki/Tree_%28data_structure%29) 
* [hot-plug](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)
* [cpumasks](https://0xax.gitbooks.io/linux-insides/content/Concepts/cpumask.html)
* [IRQs](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [API](https://en.wikipedia.org/wiki/Application_programming_interface)
* [atomic operations](https://en.wikipedia.org/wiki/Linearizability)
* [xchg instruction](http://x86.renejeschke.de/html/file_module_x86_id_328.html)
* [cmpxchg instruction](http://x86.renejeschke.de/html/file_module_x86_id_41.html)
* [lock instruction](http://x86.renejeschke.de/html/file_module_x86_id_159.html)
* [bts instruction](http://x86.renejeschke.de/html/file_module_x86_id_25.html)
* [btr instruction](http://x86.renejeschke.de/html/file_module_x86_id_24.html)
* [bt instruction](http://x86.renejeschke.de/html/file_module_x86_id_22.html)
* [sbb instruction](http://x86.renejeschke.de/html/file_module_x86_id_286.html)
* [btc instruction](http://x86.renejeschke.de/html/file_module_x86_id_23.html)
* [man memcpy](http://man7.org/linux/man-pages/man3/memcpy.3.html) 
* [man memset](http://man7.org/linux/man-pages/man3/memset.3.html)
* [CF](https://en.wikipedia.org/wiki/FLAGS_register)
* [inline assembler](https://en.wikipedia.org/wiki/Inline_assembler)
* [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)