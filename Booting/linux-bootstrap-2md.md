# 在内核安装代码的第一步
#https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html

内核启动的第一步  
--------------------------------------------------------------------------------

在[上一节中](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-1.html)我们开始接触到内核启动代码，并且分析了初始化部分，最后我们停在了对`main`函数（`main`函数是第一个用C写的函数）的调用（`main`函数位于[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)。

在这一节中我们将继续对内核启动过程的研究，我们将  
* 认识`保护模式` 
* 如何从实模式进入保护模式 
* 堆和字符界面初始化 
* 内存检测，cpu验证，键盘初始化 
* 还有更多 

现在让我们开始我们的旅程

保护模式 
--------------------------------------------------------------------------------
在操作系统可以使用Intel 64位CPU的[长模式](http://en.wikipedia.org/wiki/Long_mode)之前，内核必须首先将CPU切换到保护模式运行。

什么是[保护模式](https://en.wikipedia.org/wiki/Protected_mode)？保护模式于1982年被引入到Intel CPU家族，并且从那之后，知道Intel 64出现，保护模式都是Intel CPU的主要运行模式。

淘汰[实模式](http://wiki.osdev.org/Real_Mode)的主要原因是因为在实模式下，系统能够访问的内存非常有限。如果你还记得我们在上一节说的，在实模式下，系统最多只能访问1M内存，而且在很多时候，实际能够访问的内存只有640K。

保护模式带来了很多的改变，不过只要的改变都集中在内存管理方法。在保护模式中，实模式的20位地址线被替换成32位地址线，因此系统可以访问多大4GB的地址空间。另外，在保护模式中引入了[内存分页](http://en.wikipedia.org/wiki/Paging)功能，在后面我们将介绍这个功能。

保护模式提供了2种完全不同的内存关机机制：

* 段式内存管理
* 内存分页

在这一节中，我们只介绍段式内存管理，内存分页我们将在后面的章节进行介绍。

在上一节中我们说过，在实模式下，一个物理地址是由2个部分组成的：

* 内存段的基地址 
* 从基地址开始的偏移 
 
通过这2个信息，我们可以通过下面的公式计算出对应的物理地址 

```
PhysicalAddress = Segment * 16 + Offset
```

在保护模式中，内存段的定义和实模式完全不同。在保护模式中，每个内存段不再是64K大小，段的大小和起始位置是通过一个叫做`段描述符`的数据结构进行描述的。所有内存段的段描述符存储在一个叫做`全局描述符表`(GDT)的内存结构中。

全局描述符表示一个内存数据结构，但是它在内存中的位置并不是固定的，它的地址保存在一个特殊寄存器`GDTR`中。在后面的章节中，我们将在Linux内核代码中看到全局描述符表的地址是如何被保存到`GDTR`中的。具体的汇编代码看起来是这样的：

```assembly
lgdt gdt
```

`lgdt`汇编代码将把全局描述符表的基地址和大小保存到`GDTR`寄存器中。`GRTD`是一个48位的寄存器，这个寄存器中的保存了2部分的内容:

* 全局描述符表的大小 (16位）
* 全局描述符表的基址 (32位)
 
就像前面的段落说的，全局描述符表包含了所有内存段的`段描述符`。每个段描述符长度是64位，结构如下图描述：

```
31          24        19      16              7            0
------------------------------------------------------------
|             | |B| |A|       | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 | 4
|             | |D| |L| 19:16 | |   | |1|C|R|A|            |
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |       LIMIT 15:0           | 0
|                             |                            |
------------------------------------------------------------
```

粗粗一看，上面的结构非常吓人，不过实际上这个结构是非常容易理解的。比如在上图中的LIMIT 15:0表示这个数据结构的0到15位保存的内存段的大小的0到15位。相似的LIMITE 19:16表示上述数据结构的16到19位保存的是内存段大小的16到19位。从这个分析中，我们可以看出每个内存段的大小是通过20位进行描述的。下面我们将对这个数据结构进行仔细分析：

1. Limit[20位]被保存在上述内存结构的0-15和16-19位。根据上述内存结构中`G`位的设置，这20位内存定义的内存长度是不一样的。下面是一些具体的例子：

  * 如果`G`= 0, 并且Limit = 0， 那么表示段长度是1 byte
  * 如果`G` = 1, 并且Limit = 0, 那么表示段长度是4K bytes
  * 如果`G` = 0，并且Limit = 0xfffff，那么表示段长度是1M bytes
  * 如果`G` = 1，并且Limit = 0xfffff，那么表示段长度是4G bytes

  从上面的例子我们可以看出：
  
  * 如果G = 0, 那么内存段的长度是按照1 byte进行增长的 ( Limit每增加1，段长度增加1 byte )，最大的内存段长度将是1M bytes；
  * 如果G = 1, 那么内存段的长度是按照4K bytes ( Limit每增加1，段长度增加4K bytes )进行增长的，最大的内存段长度将是4G bytes;
  * 段长度的计算公司是 base_seg_length * ( LIMIT + 1)。
   
2. Base[32-bits]被保存在上述地址结构的0-15， 32-39以及56-63位。Base定义了段基址。

3. Type/Attribute (40-47 bits) 定义了内存段的类型以及支持的操作。
  * `S` 标记（ 第44位 ）定义了段的类型，`S` = 0说明这个内存段是一个系统段；`S` = 1说明这个内存段是一个代码段或者是数据段（ 堆栈段是一种特使类型的数据段，堆栈段必须是可以进行读写的段 ）。
   
在`S` = 1的情况下，上述内存结构的第43位决定了内存段是数据段还是代码段。如果43位 = 0，拿说明是一个数据段，否则就是一个代码段。

对于数据段和代码段，下面的表格给出了段类型定义

```
|           Type Field        | Descriptor Type | Description
|-----------------------------|-----------------|------------------
| Decimal                     |                 |
|             0    E    W   A |                 |
| 0           0    0    0   0 | Data            | Read-Only
| 1           0    0    0   1 | Data            | Read-Only, accessed
| 2           0    0    1   0 | Data            | Read/Write
| 3           0    0    1   1 | Data            | Read/Write, accessed
| 4           0    1    0   0 | Data            | Read-Only, expand-down
| 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed
| 6           0    1    1   0 | Data            | Read/Write, expand-down
| 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed
|                  C    R   A |                 |
| 8           1    0    0   0 | Code            | Execute-Only
| 9           1    0    0   1 | Code            | Execute-Only, accessed
| 10          1    0    1   0 | Code            | Execute/Read
| 11          1    0    1   1 | Code            | Execute/Read, accessed
| 12          1    1    0   0 | Code            | Execute-Only, conforming
| 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed
| 13          1    1    1   0 | Code            | Execute/Read, conforming
| 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed
```

从上面的表格我们可以看出，当第43位是`0`的时候，这个段描述符对应的是一个数据段，如果该位是`1`，那么表示这个段描述符对应的是一个代码段。对于数据段，第42，41，40位表示的是(*E*扩展，*W*可写，*A*可访问）；对于代码段，第42，41，40位表示的是(*C*一致，*R*可读，*A*可访问）。 *A*ccessible) or CRA(*C*onforming *R*eadable *A*ccessible)。
  * 如果`E` = 0，数据段是向上扩展数据段，反之为向下扩展数据段。关于向上扩展和向下扩展数据段，可以参考下面的[链接](http://www.sudleyplace.com/dpmione/expanddown.html)。在一般情况下，应该是不会使用向下扩展数据段的。
  * 如果`W` = 1，说明这个数据段是可写的，否则不可写。所有数据段都是可读的。
  * A位表示该内存段是否已经被CPU访问。
  * 如果`C` = 1，说明这个代码段可以被第优先级的代码访问，比如可以被用户态代码访问。反之如果`C` = 0，说明只能同优先级的代码段可以访问。
  * 如果`R` = 1，说明该代码段可读。代码段是永远没有写权限的。

4. DPL[2-bits, bit 45 和 46] (描述符优先级) 定义了该段的优先级。具体数值是0-3。

5. P 标志(bit 47) - 说明该内存段是否已经存在于内存中。如果`P` = 0，那么在访问这个内存段的时候将报错。

6. AVL 标志(bit 52) - 这个位在Linux内核中没有被使用。

7. L 标志(bit 53) - 只对代码段有意义，如果`L` = 1，说明该代码段需要运行在64位模式下。

8. D/B flag(bit 54) - 根据段描述符描述的是一个可执行代码段、下扩数据段还是一个堆栈段，这个标志具有不同的功能。（对于32位代码和数据段，这个标志应该总是设置为1；对于16位代码和数据段，这个标志被设置为0。）。

  * 可执行代码段。此时这个标志称为D标志并用于指出该段中的指令引用有效地址和操作数的默认长度。如果该标志置位，则默认值是32位地址和32位或8位的操作数；如果该标志为0，则默认值是16位地址和16位或8位的操作数。指令前缀0x66可以用来选择非默认值的操作数大小；前缀0x67可用来选择非默认值的地址大小。
  * 栈段（由SS寄存器指向的数据段）。此时该标志称为B（Big）标志，用于指明隐含堆栈操作（如PUSH、POP或CALL）时的栈指针大小。如果该标志置位，则使用32位栈指针并存放在ESP寄存器中；如果该标志为0，则使用16位栈指针并存放在SP寄存器中。如果堆栈段被设置成一个下扩数据段，这个B标志也同时指定了堆栈段的上界限。
  * 下扩数据段。此时该标志称为B标志，用于指明堆栈段的上界限。如果设置了该标志，则堆栈段的上界限是0xFFFFFFFF（4GB）；如果没有设置该标志，则堆栈段的上界限是0xFFFF（64KB）。

在保护模式下，段寄存器保存的不再是一个内存段的基地址，而是一个称为`段选择子`的结构。每个段描述符都对应一个`段选择子`。`段选择子`是一个16位的数据结构，下图显示了这个数据结构的内容：

```
-----------------------------
|       Index    | TI | RPL |
-----------------------------
```

其中，
* **Index** 表示在GDT中，对应段描述符的索引号。
* **TI** 表示要在GDT还是LDT中查找对应的段描述符
* **RPL** 表示请求者优先级。这个优先级将和段描述符中的优先级协同工作，共同确定访问是否合法。

在保护模式下，每个段寄存器实际上包含下面2部分内容：
* 可见部分 - 段选择子
* 隐藏部分 - 段描述符

在保护模式中，cpu是通过下面的步骤来找到一个具体的物理地址的：

* 代码必须将相应的`段选择子`装入某个段寄存器
* CPU根据`段选择子`从GDT中找到一个匹配的段描述符，然后将段描述符放入段寄存器的隐藏部分
* 在没有使用向下扩展段的时候，那么内存段的基地址就是`段描述符中的基地址`，段描述符的`limit + 1`就是内存段的长度。如果你知道一个内存地址的`偏移`，那么在没有开启分页机制的情况下，这个内存的物理地址就是`基地址+偏移`

![linear address](http://oi62.tinypic.com/2yo369v.jpg)

当代码要从实模式进入保护模式的时候，需要执行下面的操作： is:

* 禁止中断发生
* 使用命令`lgdt`将GDT表装入内存
* 设置CR0寄存器的PE位为1，是CPU进入保护模式
* 跳转开始执行保护模式代码

在后面的章节中，我们将看到Linux 内核中完整的转换代码。不过在系统进入保护模式之前，还有很多准备工作需要完成。

让我们代开C文件 [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)。这个文件包含了很多的函数，这些函数分别会执行键盘初始化，内存堆初始化等等操作...，下面让我们来具体看一些重要的函数。

将启动参数拷贝到"zeropage"
--------------------------------------------------------------------------------

让我们从`main`函数开始看起，这个函数中，首先调用了[`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L30)。 

这个函数将内存设置信息拷贝到`boot_params`结构的相应字段。大家可以在[arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L113)找到`boot_params`结构的定义。

1. 将[header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L281)中定义的`hdr`结构中的内容拷贝到`boot_params`结构的字段`struct setup_header hdr`中。

2. 如果内核是通过老的命令行协议运行起来的，那么就更新内核的命令行指针。

这里需要注意的是拷贝`hdr`数据结构的`memcpy`函数不是C语言中的函数，而是定义在 [copy.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S)。让我们来具体分析一下这段代码：

```assembly
GLOBAL(memcpy)
	pushw	%si          ;push si to stack
	pushw	%di          ;push di to stack
	movw	%ax, %di     ;move &boot_param.hdr to di
	movw	%dx, %si     ;move &hdr to si
	pushw	%cx          ;push cx to stack ( sizeof(hdr) )
	shrw	$2, %cx    
	rep; movsl           ;copy based on 4 bytes
	popw	%cx          ;pop cx
	andw	$3, %cx      ;cx = cx % 4
	rep; movsb           ;copy based on one byte
	popw	%di
	popw	%si
	retl
ENDPROC(memcpy)
```

在`copy.S`文件中，你可以看到所有的方法都开始于`GLOBAL`宏定义，而结束于`ENDPROC`宏定义。 

你可以在 [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/linkage.h)中找到`GLOBAL`宏定义。这个宏给代码段分配了一个名字标签，并且让这个名字全局可用。 

```assembly
#define GLOBAL(name)	\
	.globl name;	\
	name:
```

你可以在[include/linux/linkage.h](https://github.com/torvalds/linux/blob/master/include/linux/linkage.h)中找到`ENDPROC`宏的定义。 这个宏通过`END(name)`代码标识了汇编函数的结束，同时将函数名输出，从而静态分析工具可以找到这个函数。

```assembly
#define ENDPROC(name) \
	.type name, @function ASM_NL \
	END(name)
```

`memcpy`的实现代码是很容易理解的。首先，代码将`si`和`di`寄存器的值压入堆栈进行保存，这么做的原因是因为后续的代码将修改`si`和`di`寄存器的值。`memcpy`函数（也包括其他定义在copy.s中的其他函数）使用了`fastcall`调用规则，意味着所有的函数调用参数是通过`ax`, `dx`, `cx`寄存器传入的，而不是传统的通过堆栈传入。因此在使用下面的代码调用`memcpy`函数的时候

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```

函数的参数是这样传递的

* `ax` 寄存器指向`boot_param.hdr`的内存地址
* `dx` 寄存器指向`hdr`的内存地址
* `cx` 寄存器包含`hdr`结构的大小
 
`memcpy`函数在将`si`和`di`寄存器压栈之后，将`boot_param.hdr`的地址放入`di`寄存器，将`hdr`的地址放入`si`寄存器，并且将`hdr`数据结构的大小压栈。 接下来代码首先以4个字节为单位，将`si`寄存器指向的内存内容拷贝`di`寄存器指向的内存。当剩下的字节数不足4字节的时候，代码将原始的`hdr`数据结构大小出栈放入`cx`，然后对`cx`的值对4求模，接下来就是根据`cx`的值，以字节为单位将`si`寄存器指向的内存内容拷贝到`di`寄存器指向的内存。当拷贝操作完成之后，将保留的`si`以及`di`寄存器值出栈，函数返回。

控制台初始化
--------------------------------------------------------------------------------

在`hdr`结构体被拷贝到`boot_params.hdr`成员之后，系统接下来将进行控制台的初始化。控制台初始化时通过调用[arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/early_serial_console.c)中定义的`console_init`函数实现的。

这个函数首先查看命令行参数是否包含`earlyprintk`选项。如果命令行参数包含该选项，那么函数将分析这个选项的内容。得到控制台将使用的串口信息，然后进行串口的初始化。以下是`earlyprintk`选项可能的取值：

* serial,0x3f8,115200
* serial,ttyS0,115200
* ttyS0,115200

当串口初始化成功之后，我们将看到如下的输出如果命令行参数包含`debug`选项。

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

`puts`函数定义在[tty.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/tty.c)。这个函数只是简单的调用`putchar`函数将输入字符串中的内容按字节输出。下面让我们来看看`putchar`函数的实现：

```C
void __attribute__((section(".inittext"))) putchar(int ch)
{
    if (ch == '\n')
        putchar('\r');

    bios_putchar(ch);

    if (early_serial_base != 0)
        serial_putchar(ch);
}
```

`__attribute__((section(".inittext")))` 说明这段代码将被放入`.inittext`代码段。关于`.inittext`代码段的定义你可以在 [setup.ld](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L19)中找到。

首先如果需要输出的字符是`\n`，那么`putchar`函数将调用自己首先输出一个字符`\r`。接下来，就调用`bios_putchar`函数将字符输出到显示器（使用bios int10中断）：

```C
static void __attribute__((section(".inittext"))) bios_putchar(int ch)
{
    struct biosregs ireg;

    initregs(&ireg);
    ireg.bx = 0x0007;
    ireg.cx = 0x0001;
    ireg.ah = 0x0e;
    ireg.al = ch;
    intcall(0x10, &ireg, NULL);
}
```

在上面的代码中`initreg`函数接受一个`biosregs`结构的地址作为输入参数，该函数首先调用`memset`函数将`biosregs`结构体所有成员清0。

```C
    memset(reg, 0, sizeof *reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```

下面让我们来看看[memset](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S#L36)函数的实现 :

```assembly
GLOBAL(memset)
    pushw   %di
    movw    %ax, %di
    movzbl  %dl, %eax
    imull   $0x01010101,%eax
    pushw   %cx
    shrw    $2, %cx
    rep; stosl
    popw    %cx
    andw    $3, %cx
    rep; stosb
    popw    %di
    retl
ENDPROC(memset)
```

首先你会发现，`memset`函数和`memcpy`函数一样使用了`fastcall`调用规则，因此函数的参数是通过`ax`，`dx`以及`cx`寄存器传入函数内部的。

就像memcpy函数一样，`memset`函数一开始将`di`寄存器入栈，然后将`biosregs`结构的地址从`ax`寄存器拷贝到`di`寄存器。记下来，使用`movzbl`指令将`dl`寄存器的内容拷贝到`ax`寄存器的滴字节，到这里`ax`寄存器就包含了需要拷贝到`di`寄存器所指向的内存的值。

接下来的`imull`指令将`eax`寄存器的值乘上`0x01010101`。这么做的原因是代码每次将尝试拷贝4个字节内存的内容。下面让我们来看一个具体的例子，假设我们需要将`0x7`这个数值放到内存中，在执行`imull`指令之前，`eax`寄存器的值是`0x7`，在`imull`指令被执行之后，`eax`寄存器的内容变成了`0x07070707`（4个字节的`0x7`）。在`imull`指令之后，代码使用`rep; stosl`指令将`eax`寄存器的内容拷贝到`es:di`指向的内存。

在`bisoregs`结构体被`initregs`函数正确填充之后，`bios_putchar` 调用中断 [0x10](http://www.ctyme.com/intr/rb-0106.htm) 在显示器上输出一个字符。接下来`putchar`函数检查是否初始化了串口，如果串口被初始化了，那么将调用[serial_putchar](https://github.com/torvalds/linux/blob/master/arch/x86/boot/tty.c#L30)将字符输出到串口。

堆初始化
--------------------------------------------------------------------------------

当堆栈和bss段在[header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S)中被初始化之后 (细节请参考上一篇[part](linux-bootstrap-1.md)), 内核需要初始化全局堆，全局堆的初始化是通过 [`init_heap`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116) 函数实现的。

代码首先检查内核设置头中的[`loadflags`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321)是否设置了 [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L21)标志。 如果该标记被设置了，那么代码将计算堆栈的结束地址：:

```C
    char *stack_end;
    
    //%P1 is (-STACK_SIZE)
    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

换言之`stack_end = esp - STACK_SIZE`.

在计算了堆栈结束地址之后，代码计算了堆的结束地址：

```c

    //heap_end = heap_end_ptr + 512
    heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```

接下来代码判断`heap_end`是否大于`stack_end`，如果条件成立，将`stack_end`设置成`heap_end`（这么做是因为在大部分系统中全局堆和堆栈是相邻的）。

到这里位置，全局堆就被正确初始化了。在全局堆被初始化之后，我们就可以使用`GET_HEAP`方法。至于这个函数的实现和使用，我们将在后续的章节中看到。

检查CPU类型
--------------------------------------------------------------------------------

在堆栈初始化之后，内核代码通过调用[arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpu.c)提供的`validate_cpu`方法检查CPU类型以确定系统是否能够在当前的CPU上运行。

`validate_cpu`调用了[`check_cpu`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpucheck.c#L102)方法得到当前系统的CPU级别，并且和系统预设的最低CPU级别进行比较。如果不满足条件，则不允许系统运行。function and passes cpu level and required cpu level to it and checks that the kernel launches on the right cpu level.

```c
/*from cpu.c*/
check_cpu(&cpu_level, &req_level, &err_flags);
/*after check_cpu call, req_level = req_level defined in cpucheck.c*/
if (cpu_level < req_level) {
    printf("This kernel requires an %s CPU, ", cpu_name(req_level)); 
    printf("but only detected an %s CPU.\n", cpu_name(cpu_level));
    return -1;
}
```

`check_cpu`方法做了大量的检测和设置工作，下面就简单介绍一些：1）检查cpu标志，如果cpu是64位cpu，那么就设置[long mode](http://en.wikipedia.org/wiki/Long_mode), 2) 检查CPU的制造商，根据制造商的不同，设置不同的CPU选项。比如对于AMD出厂的cpu，如果不支持SSE+SSE2，那么就禁止这些选项。

内存侦测
--------------------------------------------------------------------------------

接下来，内核调用`detect_memory`方法进行内存侦测，以得到系统当前内存的使用分布。It uses different programming interfaces for memory detection like `0xe820`, `0xe801` and `0x88`. We will see only the implementation of **0xE820** here.

Let's look into the `detect_memory_e820` implementation from the [arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/memory.c) source file. First of all, the `detect_memory_e820` function initializes the `biosregs` structure as we saw above and fills registers with special values for the `0xe820` call:

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

* `ax` contains the number of the function (0xe820 in our case)
* `cx` register contains size of the buffer which will contain data about memory
* `edx` must contain the `SMAP` magic number
* `es:di` must contain the address of the buffer which will contain memory data
* `ebx` has to be zero.

Next is a loop where data about the memory will be collected. It starts from the call of the `0x15` BIOS interrupt, which writes one line from the address allocation table. For getting the next line we need to call this interrupt again (which we do in the loop). Before the next call `ebx` must contain the value returned previously:

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

Ultimately, it does iterations in the loop to collect data from the address allocation table and writes this data into the `e820entry` array:

* start of memory segment
* size  of memory segment
* type of memory segment (which can be reserved, usable and etc...).

You can see the result of this in the `dmesg` output, something like:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

Keyboard initialization
--------------------------------------------------------------------------------

The next step is the initialization of the keyboard with the call of the [`keyboard_init()`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L65) function. At first `keyboard_init` initializes registers using the `initregs` function and calling the [0x16](http://www.ctyme.com/intr/rb-1756.htm) interrupt for getting the keyboard status.
```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```
After this it calls [0x16](http://www.ctyme.com/intr/rb-1757.htm) again to set repeat rate and delay.
```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

Querying
--------------------------------------------------------------------------------

The next couple of steps are queries for different parameters. We will not dive into details about these queries, but will get back to it in later parts. Let's take a short look at these functions:

The [query_mca](https://github.com/torvalds/linux/blob/master/arch/x86/boot/mca.c#L18) routine calls the [0x15](http://www.ctyme.com/intr/rb-1594.htm) BIOS interrupt to get the machine model number, sub-model number, BIOS revision level, and other hardware-specific attributes:

```c
int query_mca(void)
{
    struct biosregs ireg, oreg;
    u16 len;

    initregs(&ireg);
    ireg.ah = 0xc0;
    intcall(0x15, &ireg, &oreg);

    if (oreg.eflags & X86_EFLAGS_CF)
        return -1;  /* No MCA present */

    set_fs(oreg.es);
    len = rdfs16(oreg.bx);

    if (len > sizeof(boot_params.sys_desc_table))
        len = sizeof(boot_params.sys_desc_table);

    copy_from_fs(&boot_params.sys_desc_table, oreg.bx, len);
    return 0;
}
```

It fills  the `ah` register with `0xc0` and calls the `0x15` BIOS interruption. After the interrupt execution it checks  the [carry flag](http://en.wikipedia.org/wiki/Carry_flag) and if it is set to 1, the BIOS doesn't support (**MCA**)[https://en.wikipedia.org/wiki/Micro_Channel_architecture]. If carry flag is set to 0, `ES:BX` will contain a pointer to the system information table, which looks like this:

```
Offset  Size    Description
 00h    WORD    number of bytes following
 02h    BYTE    model (see #00515)
 03h    BYTE    submodel (see #00515)
 04h    BYTE    BIOS revision: 0 for first release, 1 for 2nd, etc.
 05h    BYTE    feature byte 1 (see #00510)
 06h    BYTE    feature byte 2 (see #00511)
 07h    BYTE    feature byte 3 (see #00512)
 08h    BYTE    feature byte 4 (see #00513)
 09h    BYTE    feature byte 5 (see #00514)
---AWARD BIOS---
 0Ah  N BYTEs   AWARD copyright notice
---Phoenix BIOS---
 0Ah    BYTE    ??? (00h)
 0Bh    BYTE    major version
 0Ch    BYTE    minor version (BCD)
 0Dh  4 BYTEs   ASCIZ string "PTL" (Phoenix Technologies Ltd)
---Quadram Quad386---
 0Ah 17 BYTEs   ASCII signature string "Quadram Quad386XT"
---Toshiba (Satellite Pro 435CDS at least)---
 0Ah  7 BYTEs   signature "TOSHIBA"
 11h    BYTE    ??? (8h)
 12h    BYTE    ??? (E7h) product ID??? (guess)
 13h  3 BYTEs   "JPN"
 ```

Next we call the `set_fs` routine and pass the value of the `es` register to it. The implementation of `set_fs` is pretty simple:

```c
static inline void set_fs(u16 seg)
{
    asm volatile("movw %0,%%fs" : : "rm" (seg));
}
```

This function contains inline assembly which gets the value of the `seg` parameter and puts it into the `fs` register. There are many functions in [boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/boot/boot.h) like `set_fs`, for example `set_gs`, `fs`, `gs` for reading a value in it etc...

At the end of `query_mca` it just copies the table pointed to by `es:bx` to the `boot_params.sys_desc_table`.

The next step is getting [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) information by calling the `query_ist` function. First of all it checks the CPU level and if it is correct, calls `0x15` for getting info and saves the result to `boot_params`.

The following [query_apm_bios](https://github.com/torvalds/linux/blob/master/arch/x86/boot/apm.c#L21) function gets [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) information from the BIOS. `query_apm_bios` calls the `0x15` BIOS interruption too, but with `ah` = `0x53` to check `APM` installation. After the `0x15` execution, `query_apm_bios` functions check the `PM` signature (it must be `0x504d`), carry flag (it must be 0 if `APM` supported) and value of the `cx` register (if it's 0x02, protected mode interface is supported).

Next it calls `0x15` again, but with `ax = 0x5304` for disconnecting the `APM` interface and connecting the 32-bit protected mode interface. In the end it fills `boot_params.apm_bios_info` with values obtained from the BIOS.

Note that `query_apm_bios` will be executed only if `CONFIG_APM` or `CONFIG_APM_MODULE` was set in the configuration file:

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

The last is the [`query_edd`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/edd.c#L122) function, which queries `Enhanced Disk Drive` information from the BIOS. Let's look into the `query_edd` implementation.

First of all it reads the [edd](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt#L1023) option from the kernel's command line and if it was set to `off` then `query_edd` just returns.

If EDD is enabled, `query_edd` goes over BIOS-supported hard disks and queries EDD information in the following loop:

```C
for (devno = 0x80; devno < 0x80+EDD_MBR_SIG_MAX; devno++) {
    if (!get_edd_info(devno, &ei) && boot_params.eddbuf_entries < EDDMAXNR) {
        memcpy(edp, &ei, sizeof ei);
        edp++;
        boot_params.eddbuf_entries++;
    }
    ...
    ...
    ...
```

where `0x80` is the first hard drive and the value of `EDD_MBR_SIG_MAX` macro is 16. It collects data into the array of [edd_info](https://github.com/torvalds/linux/blob/master/include/uapi/linux/edd.h#L172) structures. `get_edd_info` checks that EDD is present by invoking the `0x13` interrupt with `ah` as `0x41` and if EDD is present, `get_edd_info` again calls the `0x13` interrupt, but with `ah` as `0x48` and `si` containing the address of the buffer where EDD information will be stored.

Conclusion
--------------------------------------------------------------------------------

This is the end of the second part about Linux kernel insides. In the next part we will see video mode setting and the rest of preparations before transition to protected mode and directly transitioning into it.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me a PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Protected mode](http://wiki.osdev.org/Protected_Mode)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
* [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
* [earlyprintk documentation](http://lxr.free-electrons.com/source/Documentation/x86/earlyprintk.txt)
* [Kernel Parameters](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt)
* [Serial console](https://github.com/torvalds/linux/blob/master/Documentation/serial-console.txt)
* [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
* [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
* [EDD specification](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
* [TLDP documentation for Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (old)
* [Previous Part](linux-bootstrap-1.md)