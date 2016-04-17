# 内核初始化流程

读者在这章可以了解到整个内核初始化的完整周期，从内核解压之后的首要步骤到内核运行其自身的第一个进程。

*Note* 这里可能不是各种内核初始化的步骤的介绍。这里也只能讲述下主要的部分，不会涉及到中断控制、ACPI、以及其它部分。所有没有详细的部分，都会在其它章节有所描述。

* [在内核解压之后的首要步骤](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-1.md) - 描述内核中的首要步骤。
* [早期的中断和异常控制](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-2.md) - 描述了早期的中断初始化和早期的页面错误控制。
* [在到达内核端点之前最后的准备](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-3.md) - 描述了在调用 `start_kernel` 之前最后的准备工作。
* [内核端点](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-4.md) - 描述了内核通用代码中的第一步。
* [继续指定架构的初始化](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-5.md) - 描述了指定架构的初始化。
* [再次初始化指定架构](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-6.md) - 描述了再一次的指定架构初始化流程。
* [指定架构初始化的最后部分](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-7.md) - 描述了最后的 `setup_arch` 相关部分。
* [调度初始化](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-8.md) - 描述了调度初始化之前的准备部分，以及调度初始化。
* [RCU 初始化](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-9.md) - 描述了 [RCU](http://en.wikipedia.org/wiki/Read-copy-update)的初始化。
* [初始化结束](https://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-10.md) - Linux内核初始化的最后部分。
