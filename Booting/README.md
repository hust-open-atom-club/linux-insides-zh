# 内核引导过程

本章介绍了Linux内核引导过程。此处你将在这看到一些描述内核加载过程的整个周期的文章：

* [从引导程序到内核](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-1.html) - 介绍了从启动计算机到内核执行第一条指令之前的所有阶段;
* [在内核设置代码的第一步](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-2.html) - 介绍了在内核设置代码的第一个步骤。你会看到堆的初始化，查询不同的参数，如 EDD，IST 等...
* [视频模式初始化和保护模式切换](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-3.html) - 介绍了内核设置代码中的视频模式初始化，并切换到保护模式。
* [切换 64 位模式](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-4.html) - 介绍切换到 64 位模式的准备工作以及切换的细节。
* [内核解压缩](http://xinqiu.gitbooks.io/linux-inside-zh/content/Booting/linux-bootstrap-5.html) - 介绍了内核解压缩之前的准备工作以及直接解压缩的细节。
