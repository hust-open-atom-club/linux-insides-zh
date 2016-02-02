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

`lgdt`汇编代码将把全局描述符表的基地址和大小保存到`GDTR`寄存器中。`GRTD`是一个48位的寄存器，where the `lgdt` instruction loads the base address and limit(size) of global descriptor table to the `GDTR` register. `GDTR` is a 48-bit register and consists of two parts: