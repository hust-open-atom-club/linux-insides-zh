#内核初始化流程

读者在这章可以了解到整个内核初始化的完整周期，从内核解压之后的第一步到内核自身运行的第一个进程。

*注意* 这里不是所有内核初始化步骤的介绍。这里只有通用的内核内容，不会涉及到中断控制、 ACPI 、以及其它部分。此处没有详述的部分，会在其它章节中描述。

* [内核解压之后的首要步骤](linux-initialization-1.md) - 描述内核中的首要步骤。
* [早期的中断和异常控制](linux-initialization-2.md) - 描述了早期的中断初始化和早期的缺页处理函数。
* [在到达内核入口之前最后的准备](linux-initialization-3.md) - 描述了在调用 start_kernel 之前最后的准备工作。
* [内核入口 - start_kernel](linux-initialization-4.md) - 描述了内核通用代码中初始化的第一步。
* [体系架构初始化](linux-initialization-5.md) - 描述了特定架构的初始化。
* [进一步初始化指定体系架构](linux-initialization-6.md) - 描述了再一次的指定架构初始化流程。
* [最后对指定体系架构初始化](linux-initialization-7.md) - 描述了指定架构初始化流程的结尾。
* [调度器初始化](linux-initialization-8.md) - 描述了调度初始化之前的准备工作，以及调度初始化。
* [RCU 初始化](linux-initialization-9.md) - 描述了 RCU 的初始化。
* [初始化结束](linux-initialization-10.md) - Linux内核初始化的最后部分。
