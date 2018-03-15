中断和中断处理(十)
=====================
终结篇
-------------------------
本文是 Linux 内核[中断和中断处理](https://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/index.html)的第十节。在[上一节](https://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/linux-interrupts-9.html)，我们了解了延后中断及其相关概念，如 `softirq`，`tasklet`，`workqueue`。本节我们继续深入这个主题，现在是见识真正的硬件驱动的时候了。

以 [StringARM** SA-100/21285 评估板](http://netwinder.osuosl.org/pub/netwinder/docs/intel/datashts/27813501.pdf)串行驱动为例，我们来观察驱动程序如何请求一个 [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 线，一个中断被触发时会发生什么之类的。驱动程序代码位于 [drivers/tty/serial/21285.c](https://github.com/torvalds/linux/blob/master/drivers/tty/serial/21285.c) 源文件。好啦，源码在手，说走就走！

一个内核模块的初始化
-----------------------------------------------
与本书其他新概念类似，为了考察这个驱动程序，我们从考察它的初始化过程开始。如你所知，Linux 内核为驱动程序或者内核模块的初始化和终止提供了两个宏：

* `module_init`
* `module_exit`

可以在驱动程序的源代码中查阅这些宏的用法:

```C
module_init(serial21285_init);
module_exit(serial21285_exit);
```

大多数驱动程序都能编译成一个可装载的内核[模块](https://en.wikipedia.org/wiki/Loadable_kernel_module)，亦或被静态地链入 Linux 内核。前一种情况下，一个设备驱动程序的初始化由 `module_init` 与 `module_exit` 宏触发。这些宏定义在 [include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h) 中: 

```C
#define module_init(initfn)                                     \
        static inline initcall_t __inittest(void)               \
        { return initfn; }                                      \
        int init_module(void) __attribute__((alias(#initfn)));

#define module_exit(exitfn)                                     \
        static inline exitcall_t __exittest(void)               \
        { return exitfn; }                                      \
        void cleanup_module(void) __attribute__((alias(#exitfn)));
```

并被 [initcall](http://kernelnewbies.org/Documents/InitcallMechanism) 函数调用：

* `early_initcall`
* `pure_initcall`
* `core_initcall`
* `postcore_initcall`
* `arch_initcall`
* `subsys_initcall`
* `fs_initcall`
* `rootfs_initcall`
* `device_initcall`
* `late_initcall`

这些函数又被 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 中的 `do_initcalls` 函数调用。然而，如果设备驱动程序被静态链入 Linux 内核，那么这些宏的实现则如下所示：

```C
#define module_init(x)  __initcall(x);
#define module_exit(x)  __exitcall(x);
```

这种情况下，模块装载的实现位于 [kernel/module.c](https://github.com/torvalds/linux/blob/master/kernel/module.c) 源文件中，而初始化发生在 `do_init_module` 函数内。我们不打算在本章深入探讨可装载模块的细枝末节，而会在一个专门介绍 Linux 内核模块的章节中窥其真容。话说回来，`module_init` 宏接受一个参数 - 本例中这个值是 `serial21285_init`。从函数名可以得知，这个函数做了一些驱动程序初始化的相关工作。请看：

```C
static int __init serial21285_init(void)
{
	int ret;

	printk(KERN_INFO "Serial: 21285 driver\n");

	serial21285_setup_ports();

	ret = uart_register_driver(&serial21285_reg);
	if (ret == 0)
		uart_add_one_port(&serial21285_reg, &serial21285_port);

	return ret;
}
```

如你所见，首先它把驱动程序相关信息写入内核缓冲区，然后调用 `serial21285_setup_ports` 函数。该函数设置了 `serial21285_port` 设备的基本 [uart](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver/transmitter) 时钟：

```C
unsigned int mem_fclk_21285 = 50000000;

static void serial21285_setup_ports(void)
{
	serial21285_port.uartclk = mem_fclk_21285 / 4;
}
```

此处的 `serial21285` 是描述 `uart` 驱动程序的结构体：

```C
static struct uart_driver serial21285_reg = {
	.owner			= THIS_MODULE,
	.driver_name	= "ttyFB",
	.dev_name		= "ttyFB",
	.major			= SERIAL_21285_MAJOR,
	.minor			= SERIAL_21285_MINOR,
	.nr			    = 1,
	.cons			= SERIAL_21285_CONSOLE,
};
```

如果驱动程序注册成功，我们借助 [drivers/tty/serial/serial_core.c](https://github.com/torvalds/linux/blob/master/drivers/tty/serial/serial_core.c) 源文件中的 `uart_add_one_port` 函数添加由驱动程序定义的端口 `serial21285_port` 结构体，然后从 `serial21285_init` 函数返回：

```C
if (ret == 0)
	uart_add_one_port(&serial21285_reg, &serial21285_port);

return ret;
```

到此为止，我们的驱动程序初始化完毕。当一个 `uart` 端口被 [drivers/tty/serial/serial_core.c](https://github.com/torvalds/linux/blob/master/drivers/tty/serial/serial_core.c) 中的 `uart_open` 函数打开，该函数会调用 `uart_startup` 函数来启动这个串行端口，后者会调用 `startup` 函数。它是 `uart_ops` 结构体的一部分。每个 `uart` 驱动程序都会定义这样一个结构体。在本例中，它是这样的：

```C
static struct uart_ops serial21285_ops = {
	...
	.startup	= serial21285_startup,
	...
}
```

可以看到，`.startup` 字段是对 `serial21285_startup` 函数的引用。这个函数的实现是我们的关注重点，因为它与中断和中断处理密切相关。

请求中断线
---------------------------------------

我们来看看 `serial21285_startup` 函数的实现：

```C
static int serial21285_startup(struct uart_port *port)
{
	int ret;

	tx_enabled(port) = 1;
	rx_enabled(port) = 1;

	ret = request_irq(IRQ_CONRX, serial21285_rx_chars, 0,
			  serial21285_name, port);
	if (ret == 0) {
		ret = request_irq(IRQ_CONTX, serial21285_tx_chars, 0,
				  serial21285_name, port);
		if (ret)
			free_irq(IRQ_CONRX, port);
	}

	return ret;
}
```

首先是`TX`和`RX`。一个设备的串行总线仅由两条线组成：一条用于发送数据，另一条用于接收数据。与此对应，串行设备应该有两个串行引脚：接收器 - `RX` 和发送器 - `TX`。通过调用 `tx_enabled` 和 `rx_enalbed` 这两个宏来激活这些线。函数接下来的部分是我们最感兴趣的。注意 `request_irq` 这个函数。它注册了一个中断处理程序，然后激活一条给定的中断线。看一下这个函数的实现细节。该函数定义在 [include/linux/interrupt.h](https://github.com/torvalds/linux/blob/master/include/linux/interrupt.h) 头文件中，如下所示：

```C
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
            const char *name, void *dev)
{
        return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}
```

可以看到，`request_irq` 函数接受五个参数：

* `irq` - 被请求的中断号
* `handler` - 中断处理程序指针
* `flags` - 掩码选项
* `name` - 中断拥有者的名称
* `dev` - 用于共享中断线的指针

现在我们来考察 `request_irq` 函数的调用。可以看到，第一个参数是 `IRQ_CONRX`。我们知道它是中断号，但 `CONRX` 又是什么东西？这个宏定义在 [arch/arm/mach-footbridge/include/mach/irqs.h](https://github.com/torvalds/linux/blob/master/arch/arm/mach-footbridge/include/mach/irqs.h) 头文件中。我们可以在这里找到 `21285` 主板能够产生的全部中断。注意，在第二次调用 `request_irq` 函数时，我们传入了 `IRQ_CONTX` 中断号。我们的驱动程序会在这些中断中处理 `RX` 和 `TX` 事件。这些宏的实现很简单：

```C
#define IRQ_CONRX               _DC21285_IRQ(0)
#define IRQ_CONTX               _DC21285_IRQ(1)
...
...
...
#define _DC21285_IRQ(x)         (16 + (x))
```

这个主板的 [ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture) 中断号分布在`0`到`15`这个范围内。因此，我们的中断号就是在此之后的头两个值：`16` 和 `17`。在 `request_irq` 函数的两次调用中，第二个参数分别是 `serial21285_rx_chars` 和 `serial21285_tx_chars` 函数。当一个 `RX` 或 `TX` 中断发生时，这些函数就会被调用。我们不会在此深入探究这些函数，因为本章讲述的是中断与中断处理，而并非设备和驱动。下一个参数是 `flags`，`request_irq` 函数的两次调用中，它的值都是零。所有合法的 `flags` 都在 [include/linux/interrupt.h](https://github.com/torvalds/linux/blob/master/include/linux/interrupt.h) 中定义成诸如 `IRQF_*` 此类的宏。一些例子：

* `IRQF_SHARED` - 允许多个设备共享此中断号
* `IRQF_PERCPU` - 此中断号属于单独cpu的(per cpu)
* `IRQF_NO_THREAD` - 中断不能线程化
* `IRQF_NOBALANCING` - 此中断步参与irq平衡时
* `IRQF_IRQPOLL` - 此中断用于轮询
* 等等

这里，我们传入的是 `0`，也就是 `IRQF_TRIGGER_NONE`。这个标志是说，它不配置任何水平触发或边缘触发的中断行为。至于第四个参数(`name`)，我们传入 `serial21285_name` ，它定义如下：

```C
static const char serial21285_name[] = "Footbridge UART";
```

它会显示在 `/proc/interrupts` 的输出中。针对最后一个参数，我们传入一个指向 `uart_port` 结构体的指针。对 `request_irq` 函数及其参数有所了解后，我们来看看它的实现。从上文可知，`request_irq` 函数内部只是调用了定义在 [kernel/irq/manage.c](https://github.com/torvalds/linux/blob/master/kernel/irq/manage.c) 源文件中的 `request_threaded_irq` 函数，并分配了一个给定的中断线。该函数起始部分是 `irqaction` 和 `irq_desc` 的定义：

```C
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
                         irq_handler_t thread_fn, unsigned long irqflags,
                         const char *devname, void *dev_id)
{
        struct irqaction *action;
        struct irq_desc *desc;
        int retval;
		...
		...
		...
}
```

在本章，我们已经见识过 `irqaction` 和 `irq_desc` 结构体了。第一个结构体表示一个中断动作描述符，它包含中断处理程序指针，设备名称，中断号等等。第二个结构体表示一个中断描述符，包含指向 `irqaction` 的指针，中断标志等等。注意，`request_threaded_irq` 函数被 `request_irq` 调用时，带了一个额外的参数：`irq_handler_t thread_fn`。如果这个参数不为 `NULL`，它会创建 `irq` 线程，并在该线程中执行给定的 `irq` 处理程序。下一步，我们要做如下检查：

```C
if (((irqflags & IRQF_SHARED) && !dev_id) ||
            (!(irqflags & IRQF_SHARED) && (irqflags & IRQF_COND_SUSPEND)) ||
            ((irqflags & IRQF_NO_SUSPEND) && (irqflags & IRQF_COND_SUSPEND)))
               return -EINVAL;
```

首先，我们确保共享中断时传入了真正的 `dev_id`(译者注：不然后面搞不清楚哪台设备产生了中断)，而且 `IRQF_COND_SUSPEND` 仅对共享中断生效。否则退出函数，返回 `-EINVAL` 错误。之后，我们借助 [kernel/irq/irqdesc.c](https://github.com/torvalds/linux/blob/master/kernel/irq/irqdesc.c) 源文件中定义的 `irq_to_desc` 函数将给定的 `irq` 中断号转换成 `irq` 中断描述符。如果不成功，则退出函数，返回 `-EINVAL` 错误：

```C
desc = irq_to_desc(irq);
if (!desc)
    return -EINVAL;
```

`irq_to_desc` 函数检查给定的 `irq` 中断号是否小于最大中断号，并且返回中断描述符。这里，`irq` 中断号就是 `irq_desc` 数组的偏移量：

```C
struct irq_desc *irq_to_desc(unsigned int irq)
{
        return (irq < NR_IRQS) ? irq_desc + irq : NULL;
}
```

由于我们已经把 `irq` 中断号转换成了 `irq` 中断描述符，现在来检查描述符的状态，确保我们可以请求中断：

```C
if (!irq_settings_can_request(desc) || WARN_ON(irq_settings_is_per_cpu_devid(desc)))
    return -EINVAL;
```

失败则返回 `-EINVAL` 错误。接着，我们检查给定的中断处理程序(译者注：是指 `handler` 变量)。如果它没被传入 `request_irq` 函数，我们就检查 `thread_fn`。两个都是 `NULL` 则返回 `-EINVAL`。如果中断处理程序没有被传入 `request_irq` 函数而 `thread_fn` 不为空，则把 `handler` 设为 `irq_default_primary_handler`：

```C
if (!handler) {
    if (!thread_fn)
        return -EINVAL;
	handler = irq_default_primary_handler;
}
```

下一步，我们通过 `kzalloc` 函数为 `irqaction` 分配内存，若不成功则返回：

```C
action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
if (!action)
    return -ENOMEM;
```

欲知 `kzalloc` 详情，请查阅专门介绍 Linux 内核[内存管理](https://xinqiu.gitbooks.io/linux-insides-cn/content/MM/index.html)的章节。为 `irqaction` 分配空间后，我们即对这个结构体进行初始化，设置它的中断处理程序，中断标志，设备名称等等：

```C
action->handler = handler;
action->thread_fn = thread_fn;
action->flags = irqflags;
action->name = devname;
action->dev_id = dev_id;
```

在 `request_threaded_irq` 函数末尾，我们调用 [kernel/irq/manage.c](https://github.com/torvalds/linux/blob/master/kernel/irq/manage.c) 中的 `__setup_irq` 函数，并注册一个给定的 `irqaction`。然后释放 `irqaction` 内存并返回：

```C
chip_bus_lock(desc);
retval = __setup_irq(irq, desc, action);
chip_bus_sync_unlock(desc);

if (retval)
	kfree(action);

return retval;
```

注意，`__setup_irq` 函数的调用位于 `chip_bus_lock` 和 `chip_bus_sync_unlock` 函数之间。这些函数对慢速总线(如 [i2c](https://en.wikipedia.org/wiki/I%C2%B2C))芯片进行锁定／解锁。现在来看看 `__setup_irq` 函数的实现。`__setup_irq` 函数开头是各种检查。首先我们检查给定的中断描述符不为 `NULL`，`irqchip` 不为 `NULL`，以及给定的中断描述符模块拥有者不为 `NULL`。接下来我们检查中断是否嵌套在其他中断线程中。如果是的，我们则以 `irq_nested_primary_handler` 替换 `irq_default_priamry_handler`。

下一步，如果给定的中断不是嵌套的，并且 `thread_fn` 不为空，我们就通过 `kthread_create` 创建了一个中断处理线程。

```C
if (new->thread_fn && !nested) {
	struct task_struct *t;
	t = kthread_create(irq_thread, new, "irq/%d-%s", irq, new->name);
	...
}
```

并在最后为给定的中断描述符的剩余字段赋值。于是，我们的 `16` 和 `17` 号中断请求线注册完毕。当一个中断控制器获得这些中断的相关事件时，`serial21285_rx_chars` 和`serial21285_tx_chars` 函数会被调用。现在我们来看一看一个中断发生时到底发生了什么。

准备处理中断
------------------------------

通过上文，我们观察了为给定的中断描述符请求中断号，为给定的中断注册 `irqaction` 结构体的过程。我们已经知道，当一个中断事件发生时，中断控制器向处理器通知该事件，处理器尝试为这个中断找到一个合适的中断门。如果你已阅读本章[第八节](https://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/linux-interrupts-8.html)，你应该还记得 `native_init_IRQ` 函数。这个函数会初始化本地 [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)。这个函数的如下部分是我们现在最感兴趣的地方：

```C
for_each_clear_bit_from(i, used_vectors, first_system_vector) {
	set_intr_gate(i, irq_entries_start +
		8 * (i - FIRST_EXTERNAL_VECTOR));
}
```

这里，我们从第 `first_system_vector` 位开始，依次向后迭代 `used_vectors` 位图中所有被清除的位：

```C
int first_system_vector = FIRST_SYSTEM_VECTOR; // 0xef
```

并且设置中断门，`i` 是向量号，`irq_entries_start + 8 * (i - FIRST_EXTERNAL_VECTOR)` 是起始地址。仅有一处尚不明了 - `irq_entries_start`。这个符号定义在 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry_entry_64.S) 汇编文件中，并提供了 `irq` 入口。一起来看：

```assembly
	.align 8
ENTRY(irq_entries_start)
    vector=FIRST_EXTERNAL_VECTOR
    .rept (FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR)
	pushq	$(~vector+0x80)
    vector=vector+1
	jmp	common_interrupt
	.align	8
    .endr
END(irq_entries_start)
```

这里我们可以看到 [GNU 汇编器](https://en.wikipedia.org/wiki/GNU_Assembler)的 `.rept` 指令。这条指令会把 `.endr` 之前的这几行代码重复 `FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR` 次。我们已经知道 `FIRST_SYSTEM_VECTOR` 的值是 `0xef`，而 `FIRST_EXTERNAL_VECTOR` 等于 `0x20`。于是，它将运行：

```python
>>> 0xef - 0x20
207
```

次。在 `.rept` 指令主体中，我们把入口程序地址压入栈中(注意，我们使用负数表示中断向量号，因为正数留作标识[系统调用](https://en.wikipedia.org/wiki/System_call)之用)，将 `vector` 变量加 1，并跳转到 `common_interrupt` 标签。在 `common_interrupt` 中，我们调整了栈中向量号，执行 `interrupt` 指令，参数是 `do_IRQ`：

```assembly
common_interrupt:
	addq	$-0x80, (%rsp)
	interrupt do_IRQ
```

`interrupt` 宏定义在同一个源文件中。它把[通用](https://en.wikipedia.org/wiki/Processor_register)寄存器的值保存在栈中。如果需要，它还会通过 `SWAPGS` 汇编指令在内核中改变用户空间 `gs` 寄存器。它会增加 [per-cpu](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html) 的 `irq_count` 变量，来表明我们处于中断状态，然后调用 `do_IRQ` 函数。该函数定义于 [arch/x86/kernel/irq.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/irq.c) 源文件中，作用是处理我们的设备中断。让我们一起考察这个函数。`do_IRQ` 函数接受一个参数 - `pt_regs` 结构体，它存放着用户空间寄存器的值：

```C
__visible unsigned int __irq_entry do_IRQ(struct pt_regs *regs)
{
    struct pt_regs *old_regs = set_irq_regs(regs);
    unsigned vector = ~regs->orig_ax;
    unsigned irq;

	irq_enter();
    exit_idle();
	...
	...
	...
}
```

函数开头调用了 `set_irq_regs` 函数，后者返回被保存的 `per-cpu` 中断寄存器指针。然后又调用 `irq_enter` 和 `exit_idle` 函数。第一个函数 `irq_enter` 进入到一个中断上下文，更新 `__preempt_count` 变量。第二个函数 `exit_idle` 检查当前进程是否是 [pid](https://en.wikipedia.org/wiki/Process_identifier) 为 `0` 的 `idle` 进程，然后把 `IDLE_END` 传送给 `idle_notifier`。

接下来，我们从当前 cpu 中读取 `irq` 值，并调用 `handle_irq` 函数：

```C
irq = __this_cpu_read(vector_irq[vector]);

if (!handle_irq(irq, regs)) {
	...
	...
	...
}
...
...
...
```

`handle_irq` 函数定义于 [arch/x86/kernel/irq_64.c](https://github.com/torvalds/linux/blob/arch/x86/kernel/irq_64.c) 源文件中，它检查给定的中断描述符，然后调用 `generic_handle_irq_desc` 函数：

```C
desc = irq_to_desc(irq);
	if (unlikely(!desc))
		return false;
generic_handle_irq_desc(irq, desc);
```

该函数又调用中断处理程序：

```C
static inline void generic_handle_irq_desc(unsigned int irq, struct irq_desc *desc)
{
       desc->handle_irq(irq, desc);
}
```

但是，停一停……`handle_irq` 是何方神圣，为什么在知道 `irqaction` 指向真正的中断处理程序的情况下，偏偏通过中断描述符调用我们的中断处理程序？实际上，`irq_desc->handle_irq` 是一个用来调用中断处理程序的上层 API。它在[设备树](https://en.wikipedia.org/wiki/Device_tree) 和 [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) 的初始化过程中就设定好了。内核通过它选择正确的函数以及 `irq->actions(s)` 的调用链。就这样，当一个中断发生时，`serial21285_tx_chars` 或者 `serial21285_rx_chars` 函数会被调用。

在 `do_IRQ` 函数末尾，我们调用 `irq_exit` 函数来退出中断上下文，调用 `set_irq_regs` 函数并传入先前的用户空间寄存器，最后返回：

```C
irq_exit();
set_irq_regs(old_regs);
return 1;
```

我们已经知道，当一个 `IRQ` 工作结束之后，如果有延后中断，它们会被执行。

退出中断
---------------------

好了，中断处理程序执行完毕，我们必须从中断中返回。在 `do_IRQ` 函数将工作处理完毕后，我们将回到 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/entry_entry_64.S) 汇编代码的 `ret_from_intr` 标签处。首先，我们通过 `DISABLE_INTERRUPTS` 宏禁止中断，这个宏被扩展成 `cli` 指令，将 [per-cpu](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html) 的 `irq_count` 变量值减 1。记住，当我们处于中断上下文的时候，这个变量的值是 `1`：

```assembly
DISABLE_INTERRUPTS(CLBR_NONE)
TRACE_IRQS_OFF
decl	PER_CPU_VAR(irq_count)
```

最后一步，我们检查之前的上下文(用户空间或者内核空间)，正确地恢复它，然后通过指令退出中断：

```assembly
INTERRUPT_RETURN
```

此处的 `INTERRUPT_RETURN` 宏是：

```C
#define INTERRUPT_RETURN	jmp native_iret
```

而

```assembly
ENTRY(native_iret)

.global native_irq_return_iret
native_irq_return_iret:
	iretq
```

本节到此结束。

总结
--------------------------

这里是[中断和中断处理](https://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/index.html) 章节的第十节的结尾。如你在本节开头读到的那样，这是本章的最后一节。本章开篇阐述了中断理论，我们于是明白了什么是中断，中断的类型，然后也了解了异常以及对这种类型中断的处理，延后中断。最后在本节，我们考察了硬件中断和对这些中断的处理。当然，本节甚至本章都未能覆盖到 Linux 内核中断和中断处理的所有方面。这样并不现实，至少对我而言如此。这是一项浩大工程，不知你作何感想，对我来说，它确实浩大。这个主题远远超出本章讲述的内容，我不确定地球上能否找到一本书可以涵盖这个主题。我们漏掉了关于中断和中断处理的很多内容，但我相信，深入研究中断和中断处理相关的内核源码是个不错的点子。

如果有任何疑问或者建议，撰写评论或者在 [twitter](https://twitter.com/0xAX) 上联系我。


**请注意，英语并非我的母语。任何不便之处，我深感抱歉。如果发现任何错误，请在 [linux-insides](https://github.com/0xAX/linux-insides) 向我发送 PR。(译者注：翻译问题请发送 PR 到 [linux-insides-cn](https://www.gitbook.com/book/xinqiu/linux-insides-cn))**

链接
---------------------------------
* [串行驱动文档](https://www.kernel.org/doc/Documentation/serial/driver)
* [StrongARM** SA-110/21285 评估板](http://netwinder.osuosl.org/pub/netwinder/docs/intel/datashts/27813501.pdf)
* [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [模块](https://en.wikipedia.org/wiki/Loadable_kernel_module)
* [initcall](http://kernelnewbies.org/Documents/InitcallMechanism)
* [uart](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver/transmitter) 
* [ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture) 
* [内存管理](https://xinqiu.gitbooks.io/linux-insides-cn/content/MM/index.html)
* [i2c](https://en.wikipedia.org/wiki/I%C2%B2C)
* [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [GNU 汇编器](https://en.wikipedia.org/wiki/GNU_Assembler)
* [处理器寄存器](https://en.wikipedia.org/wiki/Processor_register)
* [per-cpu](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html)
* [pid](https://en.wikipedia.org/wiki/Process_identifier)
* [设备树](https://en.wikipedia.org/wiki/Device_tree)
* [系统调用](https://en.wikipedia.org/wiki/System_call)
* [上一节](https://xinqiu.gitbooks.io/linux-insides-cn/content/Interrupts/linux-interrupts-9.html)































