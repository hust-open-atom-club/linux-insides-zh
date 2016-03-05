Linux 内核里的数据结构——双向链表
================================================================================

双向链表
--------------------------------------------------------------------------------

Linux 内核自己实现了双向链表，可以在 [include/linux/list.h](https://github.com/torvalds/linux/blob/master/include/linux/list.h) 找到定义。我们将会从双向链表数据结构开始`内核的数据结构`。为什么？因为它在内核里使用的很广泛，你只需要在 [free-electrons.com](http://lxr.free-electrons.com/ident?i=list_head) 检索一下就知道了。

首先让我们看一下在 [include/linux/types.h](https://github.com/torvalds/linux/blob/master/include/linux/types.h) 里的主结构体：

```C
struct list_head {
	struct list_head *next, *prev;
};
```

你可能注意到这和你以前见过的双向链表的实现方法是不同的。举个例子来说，在 [glib](http://www.gnu.org/software/libc/)  库里是这样实现的：

```C
struct GList {
  gpointer data;
  GList *next;
  GList *prev;
};
```

通常来说一个链表会包含一个指向某个项目的指针。但是内核的实现并没有这样做。所以问题来了：`链表在哪里保存数据呢？`。实际上内核里实现的链表实际上是`侵入式链表`。侵入式链表并不在节点内保存数据-节点仅仅包含指向前后节点的指针，然后把数据是附加到链表的。这就使得这个数据结构是通用的，使用起来就不需要考虑节点数据的类型了。

比如：

```C
struct nmi_desc {
    spinlock_t lock;
    struct list_head head;
};
```

让我们看几个例子来理解一下在内核里是如何使用 `list_head` 的。如上所述，在内核里有实在很多不同的地方用到了链表。我们以杂项字符驱动为例来说明双向链表的使用。在 [drivers/char/misc.c](https://github.com/torvalds/linux/blob/master/drivers/char/misc.c) 的杂项字符驱动API 被用来编写处理小型硬件和虚拟设备的小驱动。这些驱动共享相同的主设备号：

```C
#define MISC_MAJOR              10
```

但是都有各自不同的次设备号。比如：

```
ls -l /dev |  grep 10
crw-------   1 root root     10, 235 Mar 21 12:01 autofs
drwxr-xr-x  10 root root         200 Mar 21 12:01 cpu
crw-------   1 root root     10,  62 Mar 21 12:01 cpu_dma_latency
crw-------   1 root root     10, 203 Mar 21 12:01 cuse
drwxr-xr-x   2 root root         100 Mar 21 12:01 dri
crw-rw-rw-   1 root root     10, 229 Mar 21 12:01 fuse
crw-------   1 root root     10, 228 Mar 21 12:01 hpet
crw-------   1 root root     10, 183 Mar 21 12:01 hwrng
crw-rw----+  1 root kvm      10, 232 Mar 21 12:01 kvm
crw-rw----   1 root disk     10, 237 Mar 21 12:01 loop-control
crw-------   1 root root     10, 227 Mar 21 12:01 mcelog
crw-------   1 root root     10,  59 Mar 21 12:01 memory_bandwidth
crw-------   1 root root     10,  61 Mar 21 12:01 network_latency
crw-------   1 root root     10,  60 Mar 21 12:01 network_throughput
crw-r-----   1 root kmem     10, 144 Mar 21 12:01 nvram
brw-rw----   1 root disk      1,  10 Mar 21 12:01 ram10
crw--w----   1 root tty       4,  10 Mar 21 12:01 tty10
crw-rw----   1 root dialout   4,  74 Mar 21 12:01 ttyS10
crw-------   1 root root     10,  63 Mar 21 12:01 vga_arbiter
crw-------   1 root root     10, 137 Mar 21 12:01 vhci
```

现在让我们看看它是如何使用链表的。首先看一下结构体 `miscdevice` ：

```C
struct miscdevice
{
      int minor;
      const char *name;
      const struct file_operations *fops;
      struct list_head list;
      struct device *parent;
      struct device *this_device;
      const char *nodename;
      mode_t mode;
};
```

我们可以看到结构体的第四个变量 `list` 是所有注册过的设备的链表。在源代码文件的开始可以看到这个链表的定义：

```C
static LIST_HEAD(misc_list);
```

它扩展开来实际上就是定义了一个 `list_head` 类型的变量：

```C
#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)
```

然后使用宏 `LIST_HEAD_INIT` 进行初始化，这会使用变量 `name` 的地址来填充 `prev` 和 `next` 结构体的两个变量。

```C
#define LIST_HEAD_INIT(name) { &(name), &(name) }
```

现在来看看注册杂项设备的函数 `misc_register` 。它在开始就用 `INIT_LIST_HEAD` 初始化了`miscdevice->list`。

```C
INIT_LIST_HEAD(&misc->list);
```

作用和宏 `LIST_HEAD_INIT`一样。

```C
static inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}
```

下一步在函数 `device_create` 创建了设备后我们就用下面的语句将设备添加到设备链表：

```
list_add(&misc->list, &misc_list);
```

内核文件 `list.h` 提供了向链表添加新项的接口函数。我们来看看它的实现：


```C
static inline void list_add(struct list_head *new, struct list_head *head)
{
	__list_add(new, head, head->next);
}
```

实际上就是使用3个指定的参数来调用了内部函数 `__list_add`：

* new  - 新项。
* head - 新项将会被添加到`head` 之后.
* head->next - `head` 之后的项。

`__list_add`的实现非常简单：

```C
static inline void __list_add(struct list_head *new,
			      struct list_head *prev,
			      struct list_head *next)
{
	next->prev = new;
	new->next = next;
	new->prev = prev;
	prev->next = new;
}
```

我们会在 `prev` 和 `next` 之间添加一个新项。所以我们用宏 `LIST_HEAD_INIT` 定义的 `misc` 链表会包含指向 `miscdevice->list` 的向前指针和向后指针。

这里仍有一个问题：如何得到列表的内容呢？这里有一个特殊的宏：

```C
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)
```

使用了三个参数：

* ptr - 指向链表头的指针；
* type - 结构体类型;
* member - 在结构体内类型为 `list_head` 的变量的名字；

比如说：

```C
const struct miscdevice *p = list_entry(v, struct miscdevice, list)
```

然后我们就可以使用 `p->minor` 或者 `p->name`来访问 `miscdevice`。让我们来看看 `list_entry` 的实现：
 
```C
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)
```

如我们所见，它仅仅使用相同的参数调用了宏 `container_of`。初看这个宏挺奇怪的：

```C
#define container_of(ptr, type, member) ({                      \
    const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
    (type *)( (char *)__mptr - offsetof(type,member) );})
```

首先你可以注意到花括号内包含两个表达式。编译器会执行花括号内的全部语句，然后返回最后的表达式的值。

举个例子来说：

```
#include <stdio.h>

int main() {
	int i = 0;
	printf("i = %d\n", ({++i; ++i;}));
	return 0;
}
```

最终会打印 `2`

下一点就是 `typeof`,它也很简单。就如你从名字所理解的，它仅仅返回了给定变量的类型。当我第一次看到宏 `container_of` 的实现时，让我觉得最奇怪的就是 `container_of` 中的 0 。实际上这个指针巧妙的计算了从结构体特定变量的偏移，这里的 `0` 刚好就是位宽里的零偏移。让我们看一个简单的例子：

```C
#include <stdio.h>

struct s {
        int field1;
        char field2;
	char field3;
};

int main() {
	printf("%p\n", &((struct s*)0)->field3);
	return 0;
}
```

结果显示 `0x5`。

下一个宏 `offsetof` 会计算从结构体的某个变量的相对于结构体起始地址的偏移。它的实现和上面类似：

```C
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

现在我们来总结一下宏 `container_of`。只需要知道结构体里面类型为 `list_head` 的变量的名字和结构体容器的类型，它可以通过结构体的变量 `list_head` 获得结构体的起始地址。在宏定义的第一行，声明了一个指向结构体成员变量 `ptr` 的指针 `__mptr` ，并且把 `ptr` 的地址赋给它。现在 `ptr` 和 `__mptr` 指向了同一个地址。从技术上讲我们并不需要这一行，但是它可以方便的进行类型检查。第一行保证了特定的结构体（参数 `type`）包含成员变量 `member`。第二行代码会用宏 `offsetof` 计算成员变量相对于结构体起始地址的偏移，然后从结构体的地址减去这个偏移，最后就得到了结构体的起始地址。

当然了 `list_add` 和 `list_entry` 不是 `<linux/list.h>` 提供的唯一函数。双向链表的实现还提供了如下API：

* list_add
* list_add_tail
* list_del
* list_replace
* list_move
* list_is_last
* list_empty
* list_cut_position
* list_splice
* list_for_each
* list_for_each_entry

等等很多其它 API。
