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