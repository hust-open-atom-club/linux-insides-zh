# Linux中的虚拟文件系统. 第一部分.

## 介绍

Linux 内核支持极其丰富的文件系统，从标准的 ext4 到 NFS 等网络文件系统，再到 procfs 等伪文件系统。文件系统格式种类繁多,但是用户在使用系统时,却能对所有文件进行一致的打开,读写等各种操作,通过统一的系统调用接(比如open,read,write,close)来进行各种文件操作,并且能通过mount来挂载各式各样的文件系统。这背后必然有一个统一的抽象层负责对用户屏蔽掉各个文件系统的差异,将统一的接口暴露给用户上层使用。这就是`VFS`发挥作用的地方。

在这一部分中，我们将探讨使这种抽象成为可能的核心概念和数据结构。

## 虚拟文件系统中的数据结构

既然`VFS`作为抽象层,它的数据结构设计上就必须具备抽象能力。由于Linux内核是用C撰写的,不像OOP语言能通过class很容易实现多态,C语言实现多态的方式比较特殊,需要通过函数指针实现多态。接下来我们介绍`VFS`几大核心数据结构和它们抽象代表的实体。

VFS 围绕四种主要的对象类型构建：**超级块 (Superblock)**、**索引节点 (Inode)**、**目录项 (Dentry)** 和 **文件 (File)**。这些结构体中的每一个都包含一个指向操作函数指针表的指针，每个函数指针可以在运行时绑定到任何一个接口匹配的函数上。(有点类似C++的虚函数,不是吗) 这使得内核可以根据具体的文件系统实现（如 ext4 或 xfs）调用特定的函数。

### 超级块 (Superblock)

`super_block` 结构体代表一个已挂载的文件系统。它存储关于文件系统本身的元数据，例如块大小、最大文件大小和挂载的根目录。

这是定义在 `include/linux/fs.h` 中的 `super_block` 结构体的精简版：

```C
struct super_block {
	struct list_head	s_list;		/* Keep this first */
	dev_t			s_dev;		/* search index; _not_ kdev_t */
	unsigned long		s_blocksize;
	struct file_system_type	*s_type;
	const struct super_operations	*s_op;
	unsigned long		s_magic;
	struct dentry		*s_root;
    // ...
	struct list_head	s_mounts;	/* list of mounts; _not_ for fs use */
    // ...
	struct list_head	s_inodes;	/* all inodes */
} __randomize_layout;
```

让我们来看几个关键字段：
*   `s_magic`: 包含一个“魔数”，允许内核验证磁盘是否包含它预期的文件系统。
*   `s_root`: 指向该文件系统根目录的 `dentry`（例如挂载分区的 `/`）。
*   `s_inodes`: 属于该文件系统实例的所有 inode 的链表。

这里对于抽象最重要的字段是 `s_op`。它指向 `struct super_operations`，定义了 VFS 如何与这个特定的文件系统实例进行交互。而s_type抽象了文件系统的类型,并且当s_type决定好之后,后续的数据结构对应的文件类型也就被决定了。它们就只需要绑定对用的操作函数表就行。

```C
struct super_operations {
   	struct inode *(*alloc_inode)(struct super_block *sb);
	void (*destroy_inode)(struct inode *);
   	void (*dirty_inode) (struct inode *, int flags);
	int (*write_inode) (struct inode *, struct writeback_control *wbc);
	int (*drop_inode) (struct inode *);
	void (*put_super) (struct super_block *);
	int (*sync_fs)(struct super_block *sb, int wait);
    // ...
};
```

因为文件系统的具体类型被sb决定了,所以构造,销毁和其他对inode本身的操作会放进super_blockoperations里而不是inode_operations里。

### 索引节点 (Inode)

`inode`（索引节点）代表文件系统中的一个具体对象，例如文件或目录。它包含关于文件的所有元数据，**除了** 它的名字。这包括权限、所有者、大小和时间戳。

```C
struct inode {
	umode_t			i_mode;
	const struct inode_operations	*i_op;
	struct super_block	*i_sb;
	struct address_space	*i_mapping;
	unsigned long		i_ino;
	loff_t			i_size;
    // ...
	struct list_head	i_lru;		/* inode LRU list */
	union {
		struct hlist_head	i_dentry;
		struct rcu_head		i_rcu;
	};
} __randomize_layout;
```
*   `i_ino`: **Inode 编号**。这是一个在所属文件系统中唯一的数字标识符，内核通过它来识别 `inode`。
*   `i_size`: **文件大小**。表示文件内容的字节数。
*   `i_dentry`: **目录项列表头**。一个 `inode` 可以有多个文件名（通过硬链接），因此可以被多个 `dentry` 结构体指向。这个 `hlist_head` 就是一个哈希链表的头部，用于链接所有指向此 `inode` 的 `dentry`。
*   `i_lru`: **最近最少使用列表**。内核会将活跃的 `inode` 缓存在内存中以提高性能。`i_lru` 字段用于将这个 `inode` 放入一个 LRU 列表中。当内存紧张时，内核可以从这个列表的尾部回收那些长时间未被使用的 `inode`。
*   `i_op`: 指向一个 `inode_operations` 结构体。它定义的不是对 `inode` 元数据本身的操作，而是**对该 `inode` 所代表的文件系统对象进行的操作**。例如：
    *   `create`: 在一个目录（由一个 inode 代表）下创建一个新文件。
    *   `lookup`: 在一个目录中查找一个文件。
    *   `mkdir`: 创建一个新目录。
    *   `rename`: 重命名。

```C
struct inode_operations {
	struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);
	int (*create) (struct mnt_idmap *, struct inode *,struct dentry *,
		       umode_t, bool);
	int (*link) (struct dentry *,struct inode *,struct dentry *);
	int (*unlink) (struct inode *,struct dentry *);
	struct dentry *(*mkdir) (struct mnt_idmap *, struct inode *,
				 struct dentry *, umode_t);
	int (*rename) (struct mnt_idmap *, struct inode *, struct dentry *,
			struct inode *, struct dentry *, unsigned int);
    // ...
} ____cacheline_aligned;
```

### 目录项 (Dentry)

你可能已经注意到 `inode` 结构体不包含文件名。这是 Linux 中的一个特意设计。文件名和 inode 之间的映射由 **Dentry**（目录项）处理。

`dentry` 结构体将特定的名称（如 "home"）连接到特定的 inode。它还通过指向其父目录来维护目录树结构。

```C
struct dentry {
	struct dentry *d_parent;	/* parent directory */
	struct qstr d_name;
	struct inode *d_inode;		/* Where the name belongs to - NULL is
					 * negative */
	const struct dentry_operations *d_op;
	struct super_block *d_sb;	/* The root of the dentry tree */
	struct hlist_head d_children;	/* our children */
    // ...
};
```

让我们来看看 dentry 的结构：
*   `d_inode`: 指向与此文件名关联的具体 `inode`。
*   `d_parent`: 指向父目录的 dentry，允许我们向上遍历树。
*   `d_children`: 该目录的子项（子目录或文件）的 dentry 链表。
*   `d_name`: 包含文件名的实际字符串（例如 "foo.txt"）。
*   `d_sb`: 指回超级块，表明该 dentry 属于哪个文件系统。

这种分离允许硬链接的存在：**多个可以具备不同名称的 `dentry` 的 `inode` 指针** 可以指向同一个 inode。这种设计也为重命名等操作带来了很大的好处。要重命名一个文件，内核只需要修改 `dentry` **中的 d_name 成员**，而无需移动磁盘上的实际数据或创建新的 inode。

## Struct File, 进程视角的文件

`file`结构体我我打算单独拿出来放到这一段讲解

或许你可能会问：*如果我们已经有了 inode，为什么还需要 file 结构体？*

这种区别至关重要，因为 `inode` 描述的是文件本身，而 `file` 结构体描述的是 **进程与该文件的交互**。`file` 结构体代表了进程视角下的文件。

首先，让我们看看 `struct file` 的定义：

```C
struct file {
	struct path			f_path;
	struct inode			*f_inode;
	const struct file_operations	*f_op;
    // ...
	spinlock_t			f_lock;
	atomic_long_t			f_count;
	unsigned int			f_flags;
	fmode_t				f_mode;
	loff_t				f_pos;
} __randomize_layout
```

*   `f_path`: 包含 `dentry` 和 `vfsmount`，用于在命名空间中定位文件。命名空间会在后续章节详细介绍。
*   `f_inode`: 指向和该 file 结构体关联的 inode。
*   `f_op`: 此打开文件的操作函数表（read, write 等）。
*   `f_lock`: 保护文件结构体字段的自旋锁。
*   `f_count`: 引用计数。只有当它降为零时，文件结构体才会被释放。
*   `f_flags`: 打开时传入的标志（例如 `O_NONBLOCK`）。
*   `f_mode`: 文件打开的模式（例如 `FMODE_READ`, `FMODE_WRITE`）。
*   `f_pos`: 当前文件的读写偏移量（游标）。

多个`inode`指针指向同一个`inode`的`file`结构体，其`f_mode`、`f_pos`字段的值完全可以不同，因为不同的进程完全可以用不同的读写模式、不同的读写偏移位置打开同一个文件实例。也就是说，`file`结构体是`VFS`针对进程所做的一层抽象。相比之下，`inode`、`super_block`、`dentry`结构体则更接近对具体文件系统实例的表示。

`f_op` 定义了进程如何与打开的文件进行交互。当用户调用 `read()` 或 `write()` 等系统调用时，内核最终会调用此结构体中对应的函数。

```C
struct file_operations {
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	int (*open) (struct inode *, struct file *);
	int (*release) (struct inode *, struct file *);
    // ...
} __randomize_layout;
```

## 结束语

在这一部分中，我们建立了虚拟文件系统的基础。我们了解到 VFS 使用四个主要对象——超级块、索引节点、目录项和文件——来抽象具体文件系统的细节，并通过operation结构体定义抽象函数指针来实现对具体文件系统的不同具体操作。

在下一部分中,我会从一个文件系统的挂载开始,开始逐步介绍这些通用数据结构和操作是怎么针对具体文件系统进行具体实现的。

## 链接

*   [Linux Kernel Documentation - VFS](https://www.kernel.org/doc/html/latest/filesystems/vfs.html)