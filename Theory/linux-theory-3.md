内联汇编
================================================================================

介绍
--------------------------------------------------------------------------------

当阅读 [Linux kernel](https://github.com/torvalds/linux) 源代码时，经常看到这样的代码：

```C
__asm__("andq %%rsp,%0; ":"=r" (ti) : "0" (CURRENT_MASK));
```

这就是 [内联汇编](https://en.wikipedia.org/wiki/Inline_assembler) 或者换句话说，汇编程序代码已集成到高级编程语言中。在此情况下，高级编程语言是 [C](https://en.wikipedia.org/wiki/C_%28programming_language%29)。`C` 编程语言不算是很高级，但仍然是。

如果您熟悉 [汇编](https://en.wikipedia.org/wiki/Assembly_language) 编程语言，你可能会注意到，`内联汇编` 与普通汇编程序并无太大区别。此外，被称为 `basic form` 的内联汇编的特殊形式和普通汇编程序完全相同。例如：

```C
__asm__("movq %rax, %rsp");
```

或：

```C
__asm__("hlt");
```

你可能会在纯汇编代码中看到相同的代码（当然没有 `__asm__` 前缀）。这很相似，但并不像乍看起来那么简单。实际上，[GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection) 支持两种形式的内联汇编语句：

* `basic`;
* `extended`.

基本形式只包括两部分：`__asm__` 关键字和包含有效汇编指令的字符串。例如，它可能是这样的：

```C
__asm__("movq    $3, %rax\t\n"
        "movq    %rsi, %rdi");
```

关键字 `asm` 可以用来代替 `__asm__`，然而 `__asm__` 是可移植的，而关键字 `asm` 是 `GNU` [扩展](https://gcc.gnu.org/onlinedocs/gcc/C-Extensions.html)。在后面的示例中，我将只使用 `__asm__` 变体。

如果你了解汇编编程语言，就会觉得这很熟悉。主要问题在于第二种形式的内联汇编语句 - `extended`。在这种形式中，我们可以向汇编语句传递参数，执行 [跳转](https://en.wikipedia.org/wiki/Branch_%28computer_science%29) 等。听起来不难，但除了汇编语言知识外，还需要了解特殊规则。每当我在 Linux 内核中看到一段内联汇编代码时，我都需要参考 `GCC` 的官方 [文档](https://gcc.gnu.org/onlinedocs/) 以记住特定 `限定符` 的行为方式或者 `=&r` 的含义。

我决定写这一部分来巩固与内联汇编相关的知识，因为内联汇编语句在 Linux 内核中相当常见，我们会在 [linux-insides-zh](https://github.com/hust-open-atom-club/linux-insides-zh/blob/master/SUMMARY.md) 中多次看到它。我想，如果我们能有一个专门的部分，包含有关内联汇编重要方面的信息，那将会非常有用。当然，你可以在官方 [文档](https://gcc.gnu.org/onlinedocs/gcc/Using-Assembly-Language-with-C.html#Using-Assembly-Language-with-C) 中找到有关内联汇编的全面信息，但我喜欢将所有信息集中在一处。

** 注：本部件不提供汇编编程指南。它的目的不是教你用汇编程序编写程序，也不是让你知道汇编程序指令的含义。只是给扩展的 ASM 做个小备忘。**

扩展内联汇编简介
--------------------------------------------------------------------------------

那么，让我们开始吧。如上所述，`basic` 汇编语句由 `asm` 或 `__asm__` 关键字和一组汇编指令组成。这种形式与“正常”汇编没有任何区别。最有趣的部分是带操作数的内联汇编程序，或称 `extended` 汇编程序。扩展汇编语句看起来更复杂，由两个及以上的部分组成：

```assembly
__asm__ [volatile] [goto] (AssemblerTemplate
                           [ : OutputOperands ]
                           [ : InputOperands  ]
                           [ : Clobbers       ]
                           [ : GotoLabels     ]);
```

所有标有方括号的参数均为可选参数。你可能会注意到，如果我们跳过可选参数以及修饰符 `volatile` 和 `goto`，就会得到 `basic` 形式。

让我们按顺序开始考虑这个问题。第一个可选的 `限定符 ` 是 `volatile`。该规范告诉编译器，汇编语句可能会产生 `副作用`。在这种情况下，我们需要防止编译器对给定的汇编语句进行优化。简单地说，`volatile` 指示编译器不要修改语句，并将其放在原始代码中的位置。举例来说，让我们看看 [Linux 内核](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/desc.h) 中的以下函数：

```C
static inline void native_load_gdt(const struct desc_ptr *dtr)
{
	asm volatile("lgdt %0"::"m" (*dtr));
}
```

在这里，我们看到 `native_load_gdt` 函数，它通过 `lgdt` 指令将基址从 [全局描述符表](https://en.wikipedia.org/wiki/Global_Descriptor_Table) 加载到 `GDTR` 寄存器。该汇编语句标有 `volatile` 限定符。这是非常重要的，编译器不会改变该汇编语句在生成代码中的原始位置。否则，`GDTR` 寄存器可能包含错误的 `全局描述符表` 地址，或者地址是正确的但结构尚未填充。这可能导致产生异常，使内核无法正常启动。

第二个可选的 `限定符` 是 `goto`。该限定符告诉编译器，给定的汇编语句可以跳转到 `GotoLabels` 中列出的标签之一。例如：

```C
__asm__ goto("jmp %l[label]" : : : : label);
```

说完这两个限定符，让我们来看看汇编语句正文的主要部分。如上所述，汇编语句的主要部分由以下四部分组成：

* 汇编指令模板;
* 输出操作数列表;
* 输入操作数列表;
* 破坏列表.

第一个字符串包含一组有效的汇编指令，这些指令可以用 `\t\n` 序列分隔。在 `extended` 形式中，处理器 [寄存器](https://en.wikipedia.org/wiki/Processor_register) 的名称必须有前缀 `%%`，其它符号（如立即数）必须以 `$` 符号开头。`输出操作数列表` 和 `输入操作数列表` 是以逗号分隔的 [C](https://en.wikipedia.org/wiki/C_%28programming_language%29) 变量，可以有 `约束` 条件。`破坏列表` 是一个寄存器或其他值的列表，这些寄存器或值会被 `汇编指令模板` 中的汇编指令修改，但不在 `输出参数列表` 中。在深入了解示例之前，我们必须先了解一下 `约束`。约束是指定操作数位置的字符串。例如，操作数的值可以写入处理器寄存器或从内存中读取等。

请看下面这个简单的例子：

```C
#include <stdio.h>

int main(void)
{
        unsigned long a = 5;
        unsigned long b = 10;
        unsigned long sum = 0;

        __asm__("addq %1,%2" : "=r" (sum) : "r" (a), "0" (b));
        printf("a + b = %lu\n", sum);
        return 0;
}
```

让我们编译并运行它，以确保它能按预期运行：

```
$ gcc test.c -o test
./test
a + b = 15
```

成功了。现在让我们详细看看这个例子。这里我们看到一个简单的 `C` 程序，它计算两个变量的和，并将计算结果放入 `sum` 变量，最后打印结果。本例由三部分组成。首先是汇编指令 [add](http://x86.renejeschke.de/html/file_module_x86_id_5.html)，它将源操作数的值与目的操作数的值相加，并将结果存储在目的操作数中。在我们的例子中：

```assembly
addq %1, %2
```

将扩展成：

```assembly
addq a, b
```

在 `输出操作数列表` 和 `输出操作数列表` 中的变量和表达式与 `汇编指令模板` 相匹配。当输入/输出操作数指定为 `%N`时，`N` 就是操作数的位置，从左至右从 `0` 开始。汇编语句的第二部分位于第一个 `:` 符号之后，包含输出值的定义：

```assembly
"=r" (sum)
```

请注意，`sum` 带有两个特殊符号：`=r`。这是我们遇到的第一个约束。这里的实际约束条件只是 `r` 本身。修饰符 `=` 表示输出值，这就是告诉编译器，之值将被丢弃，由新数据取代。除 `=` 修饰符外，`GCC` 还支持以下三种修饰符：

* `+` - 指令读取和写入参数;
* `&` - 输出寄存器不应与输入寄存器重叠，只能用于输出;
* `%` - 告诉编译器操作数可以是 [交换](https://en.wikipedia.org/wiki/Commutative_property)。

现在让我们回到 `r` 限定符。如上所述，限定符表示操作数的位置。`r` 限定符表示值将被存储在一个 [通用寄存器](https://en.wikipedia.org/wiki/Processor_register)中。最后一部分汇编：

```assembly
"r" (a), "0" (b)
```

这些是输入操作数 - 变量 `a` 和 `b`。我们已经知道 `r` 限定符的作用。现在我们来看看变量 `b` 的约束条件。`0` 或任何其它数据 `1` 到 `9` 叫 “匹配约束”。这样，一个操作数就可以用于多个规则。约束的值是源操作数索引。在我们的例子中 `0` 将匹配 `sum`。如果我们看一下程序的汇编输出：

```C
0000000000400400 <main>:
  ...
  ...
  ...
  4004fe:       48 c7 45 f8 05 00 00    movq   $0x5,-0x8(%rbp)
  400506:       48 c7 45 f0 0a 00 00    movq   $0xa,-0x10(%rbp)

  400516:       48 8b 55 f8             mov    -0x8(%rbp),%rdx
  40051a:       48 8b 45 f0             mov    -0x10(%rbp),%rax
  40051e:       48 01 d0                add    %rdx,%rax
```

首先，值 `5` 和 `10` 将被压入栈中，然后这些值将被移动到两个通用寄存器中：`%rdx` 和 `%rax`。

这样，`%rax` 寄存器既可以用来存储 `b` 的值，也可以用来存储计算结果。**注意** 我使用的是 `gcc 6.3.1` 版本，因此您的编译器生成的代码可能有所不同。

我们已经了解了内联汇编语句的输入和输出参数。在我们继续讨论 `gcc` 支持的其他约束之前，我们还没有讨论内联汇编语句的一个剩余部分 - `破坏列表`.

破坏列表
--------------------------------------------------------------------------------

如上所述，"破坏" 部分包含一个以逗号分隔的寄存器列表，其内容将被汇编代码修改。如果我们的汇编表达式需要额外的寄存器进行计算，这一点就很有用。如果我们在内联汇编语句中添加被破坏的寄存器，编译器就会考虑到这一点，有关寄存器就不会同时被编译器使用。

请看前面的例子，但我们将增加一条简单的汇编指令：

```C
__asm__("movq $100, %%rdx\t\n"
        "addq %1,%2" : "=r" (sum) : "r" (a), "0" (b));
```

如果我们看一下汇编输出：

```C
0000000000400400 <main>:
  ...
  ...
  ...
  4004fe:       48 c7 45 f8 05 00 00    movq   $0x5,-0x8(%rbp)
  400506:       48 c7 45 f0 0a 00 00    movq   $0xa,-0x10(%rbp)

  400516:       48 8b 55 f8             mov    -0x8(%rbp),%rdx
  40051a:       48 8b 45 f0             mov    -0x10(%rbp),%rax

  40051e:       48 c7 c2 64 00 00 00    mov    $0x64,%rdx
  400525:       48 01 d0                add    %rdx,%rax
```

我们将看到 `%rdx` 寄存器被覆盖为 `0x64` 即 `100`，结果将是 `110` 而不是 `10`。现在，如果我们将 `%rdx` 寄存器添加到 `破坏` 寄存器列表中：

```C
__asm__("movq $100, %%rdx\t\n"
        "addq %1,%2" : "=r" (sum) : "r" (a), "0" (b) : "%rdx");
```

并再次查看汇编的输出：

```C
0000000000400400 <main>:
  4004fe:       48 c7 45 f8 05 00 00    movq   $0x5,-0x8(%rbp)
  400506:       48 c7 45 f0 0a 00 00    movq   $0xa,-0x10(%rbp)

  400516:       48 8b 4d f8             mov    -0x8(%rbp),%rcx
  40051a:       48 8b 45 f0             mov    -0x10(%rbp),%rax

  40051e:       48 c7 c2 64 00 00 00    mov    $0x64,%rdx
  400525:       48 01 c8                add    %rcx,%rax
```

`%rcx` 寄存器将用于 `sum` 计算，从而保留了程序的预期语义。除了通用寄存器外，我们还可以传递两个特殊的限定符。它们是：

* `cc`;
* `memory`.

第一个 `cc` 表示汇编程序代码修改了 [flags](https://en.wikipedia.org/wiki/FLAGS_register) 寄存器。如果汇编程序中包含算术或逻辑指令，通常会使用这种方法：

```C
__asm__("incq %0" ::""(variable): "cc");
```

第二个 `memory` 指定符告诉编译器，给定的内联汇编语句将对输出列表中操作数未指定的内存执行读/写操作。这可以防止编译器在寄存器中使用已加载和缓存的内存值。让我们看看下面的例子：

```C
#include <stdio.h>

int main(void)
{
        unsigned long a[3] = {10000000000, 0, 1};
        unsigned long b = 5;

        __asm__ volatile("incq %0" :: "m" (a[0]));

        printf("a[0] - b = %lu\n", a[0] - b);
        return 0;
}
```

这个例子可能是人为的，但它说明了主要观点。这里我们有一个整数数组和一个整数变量。示例非常简单，我们取`a`中的第一个元素并递增其值。然后，我们计算 `a` 的第一个元素中减去 `b` 的值。最后，我们打印出结果。如果我们编译并运行这个简单的示例，结果可能会让你大吃一惊：

```
~$ gcc -O3  test.c -o test
~$ ./test
a[0] - b = 9999999995
```

这里的结果是 `a[0] - b = 9999999995`，但为什么呢？我们递增 `a[0]` 并减去 `b`，所以这里的结果应该是 `a[0] - b = 9999999996` 。

让我们看看这个例子的汇编输出：

```assembly
00000000004004f6 <main>:
  4004b4:       48 b8 00 e4 0b 54 02    movabs $0x2540be400,%rax
  4004be:       48 89 04 24             mov    %rax,(%rsp)
  ...
  ...
  ...
  40050e:       ff 44 24 f0             incq   (%rsp)

  4004d8:       48 be fb e3 0b 54 02    movabs $0x2540be3fb,%rsi
```

我们会看到 `a` 的第一个元素包含值 `0x2540be400`（`10000000000`）。最后两行代码是实际计算。

我们看到了带有 `incq` 的递增指令，但随后只是将 `0x2540be3fb` (`9999999995`)移动到了 `%rsi` 寄存器。这看起来很奇怪。

问题在于我们将 `-O3` 标志传递给了 `gcc`，因此，编译器在编译时进行了一些常量折叠和传播，以确定 `a[0] - 5` 的结果，并在运行 `movabs` 时将其还原为常量 `0x2540be3fb` 即 `9999999995`。

现在，让我们把 `memory` 添加到破坏列表中：

```C
__asm__ volatile("incq %0" :: "m" (a[0]) : "memory");
```

运行后的新结果是

```
~$ gcc -O3  test.c -o test
~$ ./test
a[0] - b = 9999999996
```

现在结果是正确的。如果我们再看一下汇编输出结果

```assembly
00000000004004f6 <main>:
  400404:       48 b8 00 e4 0b 54 02    movabs $0x2540be400,%rax
  40040b:       00 00 00
  40040e:       48 89 04 24             mov    %rax,(%rsp)
  400412:       48 c7 44 24 08 00 00    movq   $0x0,0x8(%rsp)
  400419:       00 00
  40041b:       48 c7 44 24 10 01 00    movq   $0x1,0x10(%rsp)
  400422:       00 00
  400424:       48 ff 04 24             incq   (%rsp)
  400428:       48 8b 04 24             mov    (%rsp),%rax
  400431:       48 8d 70 fb             lea    -0x5(%rax),%rsi
```

我们将看到最后两行的不同之处：

```assembly
  400428:       48 8b 04 24             mov    (%rsp),%rax
  400431:       48 8d 70 fb             lea    -0x5(%rax),%rsi
```

现在，`GCC` 不使用常量折叠，而是在汇编中保留计算结果，并在之后将 `a[0]` 的值放入 `%rax` 寄存器。最后，它只是从 `%rax` 寄存器中减去 `b` 的常量值，然后将结果放入 `%rsi`。

除了 `memory` 指定符，我们还在这里看到了一个新的约束条件 - `m`。该约束告诉编译器使用 `a[0]` 的地址，而不是其值。现在我们已经完成了 `破坏` 的学习，我们可以继续看看 `GCC` 支持的除 `r` 和 `m` 以外的其他约束。

约束
---------------------------------------------------------------------------------

现在，我们已经完成了内联汇编语句的所有三个部分，让我们回到约束上。在前面的章节中，我们已经看到了一些约束条件，如 `r` 表示 `寄存器` 操作数、`m` 表示内存操作数、`0-9` 表示重复使用的索引操作数。除此以外，`GCC` 还支持其他约束。例如，`i` 约束表示一个具有已知值的 `立即` 整数操作数：

```C
#include <stdio.h>

int main(void)
{
        int a = 0;

        __asm__("movl %1, %0" : "=r"(a) : "i"(100));
        printf("a = %d\n", a);
        return 0;
}
```

结果是：

```
~$ gcc test.c -o test
~$ ./test
a = 100
```

`I` 表示一个 32 位立即数。`i` 和 `I` 的不同之处在于 `i` 是通用的, 而 `I` 严格指定为 32 位整数数据。例如，如果您尝试编译以下代码：

```C
unsigned long test_asm(int nr)
{
        unsigned long a = 0;

        __asm__("movq %1, %0" : "=r"(a) : "I"(0xffffffffffff));
        return a;
}
```

就会出现错误：

```
$ gcc -O3 test.c -o test
test.c: In function ‘test_asm’:
test.c:7:9: warning: asm operand 1 probably doesn’t match constraints
         __asm__("movq %1, %0" : "=r"(a) : "I"(0xffffffffffff));
         ^
test.c:7:9: error: impossible constraint in ‘asm’
```

同时：

```C
unsigned long test_asm(int nr)
{
        unsigned long a = 0;

        __asm__("movq %1, %0" : "=r"(a) : "i"(0xffffffffffff));
        return a;
}
```

运行正常：

```
~$ gcc -O3 test.c -o test
~$ echo $?
0
```

`GCC` 也支持 `J`, `K`, `N` 约束，分别是 0-63 位整数, 8 位有符号整数和 8 位无符号整数。`o` 约束表示具有 `可偏移` 内存地址的内存操作数。例如：

```C
#include <stdio.h>

int main(void)
{
        static unsigned long arr[3] = {0, 1, 2};
        static unsigned long element;

        __asm__ volatile("movq 16+%1, %0" : "=r"(element) : "o"(arr));
        printf("%lu\n", element);
        return 0;
}
```

结果正如所料：

```
~$ gcc -O3 test.c -o test
~$ ./test
2
```

所有这些约束条件都可以组合使用（只要不冲突）。在这种情况下，编译器会根据具体情况选择最佳的约束。例如：

```C
unsigned long a = 10;
unsigned long b = 20;

void main(void)
{
    __asm__ ("movq %1,%0" : "=mr"(b) : "rm"(a));
}
```

这将会使用内存操作数：

```assembly
main:
        movq a(%rip),b(%rip)
        ret
b:
        .quad   20
a:
        .quad   10
```

而不是直接使用通用寄存器。

这就是内联汇编语句中所有常用的约束条件。更多信息请参阅官方 [文档](https://gcc.gnu.org/onlinedocs/gcc/Simple-Constraints.html#Simple-Constraints)。

特定架构约束
--------------------------------------------------------------------------------

在结束之前，让我们来看看一组特殊的约束条件。这些限制针对特定的体系结构，本书则针对 [x86_64](https://en.wikipedia.org/wiki/X86-64) 体系结构，我们将探讨与之相关的约束因素。首先 `a` 到 `d`、`S` 和 `D` 约束代表 [通用用途](https://en.wikipedia.org/wiki/Processor_register) 寄存器。在这个示例中 `a` 约束对应 `%al`，`%ax`，`%eax` 或 `%rax` 寄存器，具体依赖于指令宽度。`S` 和 `D` 约束分别表示 `%si` 和 `%di` 寄存器。让我们以前面的例子为例。我们可以从汇编输出中看到，`a` 变量的值存储在 `%eax` 寄存器中。现在让我们看看在有其他约束时汇编输出：

```C
#include <stdio.h>

int a = 1;

int main(void)
{
        int b;
        __asm__ ("movq %1,%0" : "=r"(b) : "d"(a));
        return b;
}
```

现在我们可以看到，`a` 变量的值将存储在 `%rax` 寄存器中：

```assembly
0000000000400400 <main>:
  4004aa:       48 8b 05 6f 0b 20 00    mov    0x200b6f(%rip),%rax        # 601020 <a>
```

`f` 和 `t` 约束表示任何浮点堆栈寄存器 - `%st` 和浮点堆栈的顶端。`u` 约束表示浮点堆栈顶端的第二个值。

仅此而已。有关 [x86_64](https://en.wikipedia.org/wiki/X86-64) 和更多详情，请参阅官方 [文档](https://gcc.gnu.org/onlinedocs/gcc/Machine-Constraints.html#Machine-Constraints)。

Links
--------------------------------------------------------------------------------

* [Linux kernel source code](https://github.com/torvalds/linux)
* [assembly programming language](https://en.wikipedia.org/wiki/Assembly_language)
* [GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [GNU extension](https://gcc.gnu.org/onlinedocs/gcc/C-Extensions.html)
* [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)
* [Processor registers](https://en.wikipedia.org/wiki/Processor_register)
* [add instruction](http://x86.renejeschke.de/html/file_module_x86_id_5.html)
* [flags register](https://en.wikipedia.org/wiki/FLAGS_register)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [constraints](https://gcc.gnu.org/onlinedocs/gcc/Machine-Constraints.html#Machine-Constraints)
