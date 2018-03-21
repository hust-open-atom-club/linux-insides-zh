内核初始化 第六部分
===========================================================

仍旧是与系统架构有关的初始化
===========================================================  

在之前的[章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-5.html)我们从 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c)了解了特定系统架构 (在我们的例子中是 `x86_64` )的初始化内容，并且通过 `x86_configure_nx` 函数根据对[NX bit](http://en.wikipedia.org/wiki/NX_bit)的支持设置 `_PAGE_NX` 标志。正如我之前写的, `setup_arch` 函数和 `start_kernel` 都非常庞大，所以在这个和下个章节我们将继续学习关于系统架构初始化进程的内容。`x86_configure_nx` 函数的下面是 `parse_early_param`。这个函数的定义在 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 并且你可以从它的名字中了解到，这个函数解析内核命令行并且基于给定的参数 (所有的内核命令行参数你都可以在 [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt) 找到)。 你可能记得在最前面的 [章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-2.html) 我们怎样安装 `earlyprintk`。在早期我们在 [arch/x86/boot/cmdline.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cmdline.c)里面的 `cmdline_find_option` 函数和 `__cmdline_find_option`, `__cmdline_find_option_bool` 的帮助下寻找内核参数和他们的值。我们在通用内核部分不依赖于系统架构，在这里我们使用另一种方法。 如果你正在阅读linux内核源代码，你可能注意到这样的调用：

```C
early_param("gbpages", parse_direct_gbpages_on);
```

`early_param` 宏需要两个参数:

* 命令行参数的名称
* 如果给定的参数通过，函数将被调用 

并且定义如下: 

```C
#define early_param(str, fn) \
        __setup_param(str, fn, fn, 1)
```

这个定义可以在 [include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h)中可以找到.  
正如你所看到的  `early_param` 宏只是调用了 `__setup_param` 宏:

```C
#define __setup_param(str, unique_id, fn, early)                \
        static const char __setup_str_##unique_id[] __initconst \
                __aligned(1) = str; \
        static struct obs_kernel_param __setup_##unique_id      \
                __used __section(.init.setup)                   \
                __attribute__((aligned((sizeof(long)))))        \
                = { __setup_str_##unique_id, fn, early }
```

这个宏定义了 `__setup_str_*_id` 变量 (这里的 `*` 取决于被给定的函数名称) 并且把给定的命令行参数赋值给这个变量。在下一行中，我们可以看到类型为 `obs_kernel_param` 的变量 `__setup_ *` 的定义及其初始化。   

`obs_kernel_param` 结构体定义如下:

```C
struct obs_kernel_param {
        const char *str;
        int (*setup_func)(char *);
        int early;
};
```

这个结构体包含三个字段:   

* 内核参数的名称  
* 功能取决于参数的函数  
* 决定参数是否为 early 的标记位 (译者注:这个参数是个标记位，有0和1两种值，两种值的后续调用是不一样的)

注意 `__set_param` 宏定义有 `__section(.init.setup)` 属性。这意味着所有 `__setup_str_ *` 将被放置在 `.init.setup` 区段中，此外正如我们在 [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/linux/blob/master/include/asm-generic/vmlinux.lds.h) 中看到的，他们将被放置在 `__setup_start` 和 `__setup_end` 之间:   

```
#define INIT_SETUP(initsetup_align)                \
                . = ALIGN(initsetup_align);        \
                VMLINUX_SYMBOL(__setup_start) = .; \
                *(.init.setup)                     \
                VMLINUX_SYMBOL(__setup_end) = .;
```

现在我们知道了参数是怎样定义的，让我们一起回到 `parse_early_param` 的实现:   

```C
void __init parse_early_param(void)
{
        static int done __initdata;
        static char tmp_cmdline[COMMAND_LINE_SIZE] __initdata;

        if (done)
                return;

        /* All fall through to do_early_param. */
        strlcpy(tmp_cmdline, boot_command_line, COMMAND_LINE_SIZE);
        parse_early_options(tmp_cmdline);
        done = 1;
}
```
`parse_early_param` 函数定义了两个静态变量。首先第一个变量 `done` 用来检查 `parse_early_param` 函数是否已经被调用，第二个变量是用来临时存储内核命令行的。在这之后我们把 `boot_command_line` 赋值到我们刚才定义的临时命令行变量中( `tmp_cmdline` ) 并且从相同的源代码文件 `main.c` 中调用 `parse_early_options` 函数。 `parse_early_options`函数从 [kernel/params.c](https://github.com/torvalds/linux/blob/master/) 中调用 `parse_args` 函数, `parse_args` 解析传入的命令行并且调用 `do_early_param` 函数。 这个 [函数](https://github.com/torvalds/linux/blob/master/init/main.c#L413) 从 ` __setup_start` 循环到 `__setup_end` ，并且如果 `obs_kernel_param` 中的 `early` 字段值为early(1) ,从 `obs_kernel_param` 调用函数(译者注:调用结构体中的第二个参数，这个参数是个函数)。在这之后所有基于早期命令行参数的服务都被创建，在 `parse_early_param` 之后的下一个调用是 `x86_report_nx` 。 正如我在这章开头所写的，我们已经用 `x86_configure_nx` 配置了 `NX-bit` 位。接下来的 [arch/x86/mm/setup_nx.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/setup_nx.c)中的 `x86_report_nx`函数仅仅打印出关于 `NX` 的信息。注意我们 `x86_report_nx` 不一定在 `x86_configure_nx` 之后调用，但是一定在 `parse_early_param` 之后调用。答案很简单: 因为内核支持 `noexec` 参数，所以我们在 `parse_early_param` 之后调用 `x86_report_nx` :

```
noexec		[X86]
			On X86-32 available only on PAE configured kernels.
			noexec=on: enable non-executable mappings (default)
			noexec=off: disable non-executable mappings
```

我们可以在启动的时候看到:

![NX](http://oi62.tinypic.com/swwxhy.jpg)

在这之后我们可以看到下面函数的调用:   

```C
	memblock_x86_reserve_range_setup_data();
```
