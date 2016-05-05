中断和中断处理。 第九部分。
================================================================================

延后中断(软中断，Tasklets和Workqueues)介绍
--------------------------------------------------------------------------------

这是[linux内核揭密](https://www.gitbook.com/book/xinqiu/linux-insides-cn/details)中断部分的第九小节，在[之前章节](https://www.gitbook.com/book/xinqiu/linux-insides-cn/content/interrupts/interrupts-8.html)我们了解了源文件[arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/irqinit.c)中`init_IRQ`的实现。接下来的这一节我们将继续深入学习和外部硬件中断相关的初始化。

在[init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c)中我们可以看到在`init_IRQ`函数后面调用了`softirq_init`函数。这个函数在源文件[kernel/softirq.c](https://github.com/torvalds/linux/blob/master/kernel/softirq.c)中定义，从名字我们可以看出，它的作用是初始化`软中断`或者也可以说是初始化`延后中断`。那么什么是延后中断？在讲解内核初始化过程的[部分](https://www.gitbook.com/book/xinqiu/linux-insides-cn/content/Initialization/linux-initialization-9.html)第九小结我们已经对他有了一些了解，Linux内核中一共有三种'延后中断'：

* `软中断`;
* `tasklets`;
* `工作队列`;

在这一小节我们将详细介绍这三种实现。就像我说的，我们对这个主题有一些了解。那么，现在是时间深入了解一下了。

延后中断
----------------------------------------------------------------------------------

对中断处理有一些严格的要求，总的来说有两种：

* 中断处理必须快速执行完毕
* 有时中断处理必须做很多冗长的事情

就像你所想到的，我们几乎不可能同时做到这两点，之前的中断被分为两部分：

* 前半部
* 后半部

`后半部`曾经是Linux内核延后中断执行的一种方式，但现在的实际情况已经不是这样了。这种遗留称谓现在作为名词代表所有延后中断执行的方式。伴随着内核对并行处理的支持，出于性能考虑，所有新的下半部实现方案都基于被称之为`ksoftirqd`(稍后将详细讨论)的内核线程。`ksoftirqd`中断处理方式几乎和硬件中断处理一样重要。中断延后处理会在系统负载较低的时候才执行一个中断的具体实现行为。如你所知，中断处理代码运行于禁止响应后续中断的中断处理上下文中，所以要避免长时间执行。但有时中断处理却又有很多的工作需要执行，所以中断处理有时会被分为两部分。第一部分中，中断处理先只做少量的最重要工作，接下来提交第二部分到内核调度，然后就结束运行。当系统比较空闲并且处理器上下文允许处理中断时，第二部分就会开始执行被延后的剩余中断任务。以上是对延后中断处理的简要介绍。
