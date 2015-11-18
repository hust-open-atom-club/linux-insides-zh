Linux内核中的数据结构
================================================================================

基数树
--------------------------------------------------------------------------------
正如你所知道的 Linux 内核通过许多不同库以及函数提供各种数据结构以及算法实现。
这个部分我们将介绍其中一个数据结构 [Radix tree](http://en.wikipedia.org/wiki/Radix_tree)。Linux 内核中有两个文件与 `radix tree` 的实现和API相关：

* [include/linux/radix-tree.h](https://github.com/torvalds/linux/blob/master/include/linux/radix-tree.h)
* [lib/radix-tree.c](https://github.com/torvalds/linux/blob/master/lib/radix-tree.c)

首先说明一下什么是 `radix tree` 。Radix tree 是一种 `压缩 trie`，其中 [trie](http://en.wikipedia.org/wiki/Trie) 是一种通过保存关联数组（associative array）来提供 `关键字-值（key-value）` 存储与查找的数据结构。通常关键字是字符串，不过也可以是其他数据类型。 

trie 结构的节点与 `n-tree` 不同，其节点中并不存储关键字，取而代之的是存储单个字符标签。关键字查找时，通过从树的根开始遍历关键字相关的所有字符标签节点，直至到达最终的叶子节点。下面是个例子：


```
               +-----------+
               |           |
               |    " "    |
               |           |
        +------+-----------+------+
        |                         |
        |                         |
   +----v------+            +-----v-----+
   |           |            |           |
   |    g      |            |     c     |
   |           |            |           |
   +-----------+            +-----------+
        |                         |
        |                         |
   +----v------+            +-----v-----+
   |           |            |           |
   |    o      |            |     a     |
   |           |            |           |
   +-----------+            +-----------+
                                  |
                                  |
                            +-----v-----+
                            |           |
                            |     t     |
                            |           |
                            +-----------+
```

这个例子中，我们可以看到 `trie` 所存储的关键字信息 `go` 与 `cat`，压缩 trie 或 `radix tree` 与 `trie` 所不同的是，所有只存在单个孩子的中间节点将被压缩。

Linux 内核中的 Radix 树将值映射为整型关键字，Radix 的数据结构定义在 [include/linux/radix-tree.h](https://github.com/torvalds/linux/blob/master/include/linux/radix-tree.h) 文件中 :

```C
struct radix_tree_root {
         unsigned int            height;
         gfp_t                   gfp_mask;
         struct radix_tree_node  __rcu *rnode;
};
```

上面这个是 radix 树的 root 节点的结构体，它包括三个成员：

* `height`   - 从叶节点向上计算出的树高度。
* `gfp_mask` - 内存分配标识。
* `rnode`    - 子节点指针。

这里我们先讨论的结构体成员是 `gfp_mask` : 

Linux 底层的内存申请接口需要提供一类标识（flag） - `gfp_mask` ，用于描述内存申请的行为。这个以 `GFP_` 前缀开头的内存申请控制标识主要包括，`GFP_NOIO` 禁止所有IO操作但允许睡眠等待内存，`__GFP_HIGHMEM` 允许申请内核的高端内存，`GFP_ATOMIC` 高优先级申请内存且操作不允许被睡眠。


接下来说的结构体成员是`rnode`：

```C
struct radix_tree_node {
        unsigned int    path;
        unsigned int    count;
        union {
                struct {
                        struct radix_tree_node *parent;
                        void *private_data;
                };
                struct rcu_head rcu_head;
        };
        /* For tree user */
        struct list_head private_list;
        void __rcu      *slots[RADIX_TREE_MAP_SIZE];
        unsigned long   tags[RADIX_TREE_MAX_TAGS][RADIX_TREE_TAG_LONGS];
};
```

这个结构体中包括这几个内容，节点与父节点的偏移以及到树底端的高度，子节点的个数，节点的存储数据域，具体描述如下：

* `path` - 从叶节点
* `count` - 子节点的个数。
* `parent` - 父节点的指针。
* `private_data` - 存储数据内容缓冲区。
* `rcu_head` - 用于节点释放的RCU链表。
* `private_list` - 存储数据。

结构体 `radix_tree_node` 的最后两个成员 `tags` 与 `slots` 是非常重要且需要特别注意的。每个 Radix 树节点都可以包括一个指向存储数据指针的 slots 集合，空闲 slots 的指针指向 NULL。 Linux 内核的 Radix 树结构体中还包含用于记录节点存储状态的标签 `tags` 成员，标签通过位设置指示 Radix 树的数据存储状态。

至此，我们了解到 radix 树的结构，接下来看一下 radix 树所提供的 API。


Linux 内核基数树 API
---------------------------------------------------------------------------------

我们从数据结构的初始化开始看，radix 树支持两种方式初始化。

第一个是使用宏 `RADIX_TREE` ：

```C
RADIX_TREE(name, gfp_mask);
````

正如你看到，只需要提供 `name` 参数，就能够使用 `RADIX_TREE` 宏完成 radix 的定义以及初始化，`RADIX_TREE` 宏的实现非常简单：

```C
#define RADIX_TREE(name, mask) \
         struct radix_tree_root name = RADIX_TREE_INIT(mask)

#define RADIX_TREE_INIT(mask)   { \
        .height = 0,              \
        .gfp_mask = (mask),       \
        .rnode = NULL,            \
}
```

`RADIX_TREE` 宏首先使用 `name` 定义了一个 `radix_tree_root` 实例并用 `RADIX_TREE_INIT` 宏带参数 `mask` 进行初始化。宏 `RADIX_TREE_INIT` 将 `radix_tree_root` 初始化为默认属性并将 gfp_mask 初始化为入参 `mask` 。
第二种方式是手工定义 `radix_tree_root` 变量，之后再使用 `mask` 调用 `INIT_RADIX_TREE` 宏对变量进行初始化。
```C
struct radix_tree_root my_radix_tree;
INIT_RADIX_TREE(my_tree, gfp_mask_for_my_radix_tree);
```

`INIT_RADIX_TREE` 宏定义：

```C
#define INIT_RADIX_TREE(root, mask)  \
do {                                 \
        (root)->height = 0;          \
        (root)->gfp_mask = (mask);   \
        (root)->rnode = NULL;        \
} while (0)
```
宏 `INIT_RADIX_TREE` 所初始化的属性与 `RADIX_TREE_INIT` 一致


接下来是 radix 树的节点插入以及删除，这两个函数：

* `radix_tree_insert`;
* `radix_tree_delete`.

第一个函数 `radix_tree_insert` 需要三个入参：

* radix 树 root 节点结构
* 索引关键字
* 需要插入存储的数据

第二个函数 `radix_tree_delete` 除了不需要存储数据参数外，其他与 `radix_tree_insert` 一致。

radix 树的查找实现有以下几个函数：The search in a radix tree implemented in two ways:

* `radix_tree_lookup`;
* `radix_tree_gang_lookup`;
* `radix_tree_lookup_slot`.

第一个函数 `radix_tree_lookup` 需要两个参数：

* radix  树 root 节点结构
* 索引关键字

这个函数通过给定的关键字查找 radix 树，并返关键字所对应的结点。

第二个函数 `radix_tree_gang_lookup` 具有以下特征：

```C
unsigned int radix_tree_gang_lookup(struct radix_tree_root *root,
                                    void **results,
                                    unsigned long first_index,
                                    unsigned int max_items);
```

函数返回查找到记录的条目数，并根据关键字进行排序，返回的总结点数不超过入参 `max_items` 的大小。

最后一个函数 `radix_tree_lookup_slot` 返回结点 slot 中所存储的数据。


链接
---------------------------------------------------------------------------------

* [Radix tree](http://en.wikipedia.org/wiki/Radix_tree)
* [Trie](http://en.wikipedia.org/wiki/Trie)


