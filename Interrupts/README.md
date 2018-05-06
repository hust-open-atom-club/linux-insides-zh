# 中断和中断处理

在 linux 内核中你会发现很多关于中断和异常处理的话题

* [中断和中断处理第一部分](linux-interrupts-1.md) - 描述中断处理主题
* [深入 Linux 内核中的中断](linux-interrupts-2.md) - 这部分开始描述和初步步骤相关的中断和异常处理。
* [初步中断处理](linux-interrupts-3.md) - 描述初步中断处理。
* [中断处理](linux-interrupts-4.md) - fourth part describes first non-early interrupt handlers.
* [异常处理的实现](linux-interrupts-5.md) - 一些异常处理的实现，比如双重错误、除零等等。
* [处理不可屏蔽中断](linux-interrupts-6.md) - 描述了如何处理不可屏蔽的中断和剩下的一些与特定架构相关的中断。
* [深入外部硬件中断](linux-interrupts-7.md) - 这部分讲述了关于处理外部硬件中断的一些早期初始化代码。
* [IRQs的非早期初始化](linux-interrupts-8.md) - 这部分讲述了处理外部硬件中断的非早期初始化代码。
* [Softirq, Tasklets and Workqueues](linux-interrupts-9.md) - 这部分讲述了softirqs、tasklets 和 workqueues 的内容。
* [最后一部分](linux-interrupts-10.md) - 这是中断和中断处理的最后一部分，并且我们将会看到一个真实的硬件驱动和中断。
