内核内存管理. 第四部分.
================================================================================

swap机制和演进
--------------------------------------------------------------------------------

在深入探索内核 Swap 机制的奥秘之前，我们必须首先明确其存在的根基——物理内存管理。系统的所有活动都围绕着对物理页面（在现代内核中更常被称为 page/folio）的分配与使用。这些以 4KB 为基本单位的物理内存页，在系统上电初始化后，便由内核的伙伴系统（Buddy System）统一管理，等待着为进程和内核自身提供服务。

然而，物理内存是一种有限且宝贵的资源。当系统面临以下两种典型场景时，挑战便随之而来：

内存超载 (Overcommit): 系统中所有进程请求的内存总量，超过了物理内存的实际容量。

内存闲置 (Inactive Memory): 大量物理页面被分配出去，但其上的数据长时间未被访问，处于“沉睡”状态，造成了资源浪费。

为了应对这些挑战，Linux 内核引入了一套精巧的页面换出（Swap-out）与换入（Swap-in）机制。其核心思想并非简单地“扩充内存”，而更像是一种内存的“时分复用”：将那些暂时“沉睡”的内存页面转移到一种速度较慢但容量巨大的存储介质上，从而腾出宝贵的物理内存，供给当前更活跃的任务使用。当未来某个时刻需要访问被转移的页面时，再将其重新加载回物理内存。

交换介质：页面的临时家园

这个用于临时安置内存页面的“后备存储介质”，就是我们通常所说的交换空间（Swap Space）。在现代系统中，它通常由我们所熟知的块存储设备来充当，最常见的便是：

固态硬盘 (SSD): 凭借其出色的随机读写性能和极低的延迟，SSD 已成为交换空间的首选介质，它能显著缩短页面的换入换出时间，减轻对系统响应能力的影响。

机械硬盘 (HDD): 尽管其随机读写性能远逊于 SSD，但在成本和容量上仍具优势。在内存压力不极端或对性能要求不高的场景下，HDD 依然是可行的选择。

无论是 SSD 还是 HDD，无论是作为一个专用的交换分区还是一个灵活的交换文件，内核都需要一套统一的机制来识别、管理和操作这些形形色色的物理介质。


swap_info_struct：交换设备的抽象身份证
--------------------------------------------------------------------------------

那么，内核是如何管理这些可能同时存在的、特性各异的交换设备的呢？答案就是通过一个核心的数据结构——struct swap_info_struct。

```c
struct swap_info_struct {
	struct percpu_ref users;	/* indicate and keep swap device valid. */
	unsigned long	flags;		/* SWP_USED etc: see above */
	signed short	prio;		/* swap priority of this type */
	struct plist_node list;		/* entry in swap_active_head */
	signed char	type;		/* strange name for an index */
	unsigned int	max;		/* extent of the swap_map */
	unsigned char *swap_map;	/* vmalloc『ed array of usage counts */
	unsigned long *zeromap;		/* kvmalloc』ed bitmap to track zero pages */
	struct swap_cluster_info *cluster_info; /* cluster info. Only for SSD */
	struct list_head free_clusters; /* free clusters list */
	struct list_head full_clusters; /* full clusters list */
	struct list_head nonfull_clusters[SWAP_NR_ORDERS];
					/* list of cluster that contains at least one free slot */
	struct list_head frag_clusters[SWAP_NR_ORDERS];
					/* list of cluster that are fragmented or contented */
	unsigned int pages;		/* total of usable pages of swap */
	atomic_long_t inuse_pages;	/* number of those currently in use */
	struct swap_sequential_cluster *global_cluster; /* Use one global cluster for rotating device */
	spinlock_t global_cluster_lock;	/* Serialize usage of global cluster */
	struct rb_root swap_extent_root;/* root of the swap extent rbtree */
	struct block_device *bdev;	/* swap device or bdev of swap file */
	struct file *swap_file;		/* seldom referenced */
	struct completion comp;		/* seldom referenced */
	spinlock_t lock;		/*
					 * protect map scan related fields like
					 * swap_map, inuse_pages and all cluster
					 * lists. other fields are only changed
					 * at swapon/swapoff, so are protected
					 * by swap_lock. changing flags need
					 * hold this lock and swap_lock. If
					 * both locks need hold, hold swap_lock
					 * first.
					 */
	spinlock_t cont_lock;		/*
					 * protect swap count continuation page
					 * list.
					 */
	struct work_struct discard_work; /* discard worker */
	struct work_struct reclaim_work; /* reclaim worker */
	struct list_head discard_clusters; /* discard clusters list */
	struct plist_node avail_lists[]; /*
					   * entries in swap_avail_heads, one
					   * entry per node.
					   * Must be last as the number of the
					   * array is nr_node_ids, which is not
					   * a fixed value so have to allocate
					   * dynamically.
					   * And it has to be an array so that
					   * plist_for_each_* can work.
					   */
};

```

struct swap_info_struct 历经多年的演进，其内部成员已变得相当丰富。为了拨开迷雾，让我们先抛开所有复杂的细节，回归到最朴素、最直观的设计思路。

既然交换空间的核心任务是临时存放从内存中换出的物理页面，一个自然而然的想法便是：我们能否用一个巨大的数组来直接映射交换区中的每一个 4KB 页面（slot）？这个数组的下标（index）直接对应交换区中的页面偏移量，数组的元素则记录着该槽位的状态。这种设计无疑是最高效、最直观的。

这个“巨大数组”的思想，正是 swap_info_struct 内部核心成员之一——swap_map 的设计雏形。swap_map 本质上就是这个“大数组”在内核中的逻辑体现，它为每一个交换槽位都维护着一个元数据条目。

然而，当我们顺着这个思路继续深入，一个问题便浮出水面：真实世界的操作系统是一个高度并发的环境。

一个看似简单的 swap_map，为何在其结构体附近总是伴随着一把自旋锁（spinlock_t）的身影？这把锁究竟在保护什么？

答案在于，一个物理页面在被换出之前，它可能并不仅仅属于单个进程。由于写时复制（Copy-on-Write, CoW）机制的存在，一个只读的物理页面可能被多个进程同时共享映射。例如，父进程 fork() 出一个子进程，在子进程写入共享内存之前，父子进程的页表项（PTE）可能指向同一个物理页面。

当这个共享页面被换出时，所有共享它的进程的 PTE 都会被修改，从指向物理页面，转而指向同一个 swp_entry（交换项）。这意味着，swap_map 中代表这个 swp_entry 的那个条目，将同时被多个 PTE 所引用。

因此，swap_map 的职责变得复杂起来。它不仅要记录一个槽位是否被使用，更需要精确地追踪它被多少个 PTE 所引用。为此，swap_map 的每个条目中都必须包含一个引用计数（swap_count）。当一个新的 PTE 指向它时，计数加一；当一个 PTE 被销毁时，计数减一。只有当引用计数归零时，这个交换槽位才能被安全地释放和重用。

现在，锁的必要性就显而易见了。在多核 CPU 环境下，两个不同的进程可能在两个不同的核心上同时退出，并尝试去递减同一个 swap_count。如果没有锁的保护，这种并发的读-修改-写操作将不可避免地导致计数错误，最终造成交换空间的管理混乱——要么是仍在被引用的槽位被错误释放（导致数据损坏），要么是本该释放的槽位永远无法回收（导致空间泄漏）。

不同的设备会被 swap_info 数组管理，通过解码的 type 对应：

```c
/*
 * Callers of all helpers below must ensure the entry, type, or offset is
 * valid, and protect the swap device with reference count or locks.
 */
static inline struct swap_info_struct *__swap_type_to_info(int type)
{
	struct swap_info_struct *si;

	si = READ_ONCE(swap_info[type]); /* rcu_dereference() */
	VM_WARN_ON_ONCE(percpu_ref_is_zero(&si->users)); /* race with swapoff */
	return si;
}

```

swap_entry_t(include/Linux/mm_types.h)
--------------------------------------------------------------------------------

```c
/*
 * A swap entry has to fit into a 「unsigned long」, as the entry is hidden
 * in the 「index」 field of the swapper address space.
 */
typedef struct {
	unsigned long val;
} swp_entry_t;

```
这是一个寻址的关键，就像 pte 的解码可以找到 page 一样，entry 的解码也是一样我们来看 arm64 的 encode：

```c
/*
 * Encode and decode a swap entry:
 *	bits 0-1:	present (must be zero)
 *	bits 2:		remember PG_anon_exclusive
 *	bit  3:		remember uffd-wp state
 *	bits 6-10:	swap type
 *	bit  11:	PTE_PRESENT_INVALID (must be zero)
 *	bits 12-61:	swap offset
 */

 ```
 对应 pte 这里 present 必须置零代表不在内存中，其他两个重要的自然是 swap type 用于寻址不同的交换设备，offset 找到对应的交换设备里面的偏移量寻址到那个 swap map 对应的 slot。


swap_cluster_info（mm/swap.h）
--------------------------------------------------------------------------------

```c
/*
 * We use this to track usage of a cluster. A cluster is a block of swap disk
 * space with SWAPFILE_CLUSTER pages long and naturally aligns in disk. All
 * free clusters are organized into a list. We fetch an entry from the list to
 * get a free cluster.
 *
 * The flags field determines if a cluster is free. This is
 * protected by cluster lock.
 */
struct swap_cluster_info {
	spinlock_t lock;	/*
				 * Protect swap_cluster_info fields
				 * other than list, and swap_info_struct->swap_map
				 * elements corresponding to the swap cluster.
				 */
	u16 count;
	u8 flags;
	u8 order;
	atomic_long_t __rcu *table;	/* Swap table entries, see mm/swap_table.h */
	struct list_head list;
};

```
这部分的设计主要是将swap map的管理精细到64M的单位，同时和社区mthp（通过页面组装和拆分的操作提供灵活的2^n阶次的页面）的新特性相结合来演进的。新的改进和工作主要是在这些部分：

1.精细化的 Cluster 状态管理：双向链表的威力
新架构的核心，是使用多个双向链表，根据 Cluster 的占用状态和页面连续性，对其进行精细化的分类管理。每个 Cluster 在其生命周期中，都会根据自身状态的变化，在这些不同的链表之间迁移。

主要的链表分级如下：

- free_clusters(全空闲链表):
成员: 完全没有被使用的 Cluster。这些 Cluster 是分配器的首选资源。
作用: 为新的换出操作提供“干净”的、可立即使用的交换空间。

- nonfull_clusters(非满链表):
成员: 部分槽位已被占用，但仍有空闲槽位的 Cluster。
作用: 这是最常见的状态。分配器会优先利用这些 Cluster 的剩余空间，以提高交换空间的利用率，避免产生过多碎片。

- full_clusters(全满链表):
成员: 所有 512 个槽位都已被占用的 Cluster。
作用: 这些 Cluster 被暂时“搁置”起来，分配器不会再扫描它们，从而避免了无效的搜索开销。当其中有页面被换回内存导致槽位被释放时，它们会重新迁移回 nonfull_clusters 链表。

2.面向大页（mTHP）的优化：按阶（Order）管理的独立链表
为了更好地支持大页（Multi-size THP, mTHP）的换出，新架构在上述分类的基础上，为 nonfull_clusters 和 fragmented_clusters 引入了按阶（per-order）管理的机制。

- 独立的nonfull_clusters[order]链表:
系统会为不同大小的页面（order-0, order-1, ... order-9）维护独立的非满链表。
当需要换出一个 64KB (order-4) 的页面时，分配器会直接去 nonfull_clusters 链表中寻找能够容纳 64KB 连续空间的 Cluster。
这种设计对 mTHP 极其友好，它避免了为了分配一个大页而去扫描大量不相关的、只包含小碎片的 Cluster，极大地提高了大页换出的分配效率。

- fragmented_clusters[order](碎片化链表 - 可选/高级特性):
用于管理那些虽然有足够空闲槽位、但无法满足特定 order 连续分配要求的 Cluster。
通过将这些碎片化的 Cluster 单独管理，可以进一步优化高 order 页面的分配路径。
它彻底消除了对整个 swap_map 进行全局扫描的昂贵过程。 所有的操作都被限定在了一个个独立的 Cluster 内部，并且通过分类链表实现了高效的目标定位。这从根本上解决了旧架构的分配冲突问题，为高并发环境下的高性能交换操作奠定了坚实的基础。最后实现的各个情况的cluster的enum如下：

```c
/* All on-list cluster must have a non-zero flag. */
enum swap_cluster_flags {
	CLUSTER_FLAG_NONE = 0, /* For temporary off-list cluster */
	CLUSTER_FLAG_FREE,
	CLUSTER_FLAG_NONFULL,
	CLUSTER_FLAG_FRAG,
	/* Clusters with flags above are allocatable */
	CLUSTER_FLAG_USABLE = CLUSTER_FLAG_FRAG,
	CLUSTER_FLAG_FULL,
	CLUSTER_FLAG_DISCARD,
	CLUSTER_FLAG_MAX,
};

```

swap table
--------------------------------------------------------------------------------

1.从“巨锁”到“微锁”：Swap 管理架构的演进
至此，我们已经铺垫了所有必要的背景知识，正式进入本次探索的核心——剖析新一代 Swap 架构的革命性优化。

一切故事的开端，都源于那个最原始、最简单的设计思想：用一个巨大的 swap_map 数组来管理所有交换槽位。然而，为了保护这个大数组在多核环境下的数据一致性，一把全局的自旋锁（spinlock 是不可避免的。在现代众核处理器上，这把“巨锁”成为了一个灾难性的性能瓶颈，所有需要与 Swap 子系统交互的 CPU 都必须在此串行排队。

2.中间方案：借用 Page Cache 框架的 64MB address_space
为了挣脱“巨锁”的束缚，内核开发者们迈出了演进的第一步。他们借用了文件系统页面缓存（Page Cache）的管理框架，将庞大的交换空间逻辑上切分为以 64MB 为单位的管理单元，即 swap_address_space。

每一个 64MB 的 address_space 都由一棵独立的 XArray 树 来管理。这成功地将一把“全局锁”打碎成了多把“64MB 区域锁”，显著缓解了锁竞争（注：关于 XArray 的精妙设计，强烈推荐阅读 Kairui 哥的深度解析文章）

让我们来做一个简单的估算：一个 64MB 的空间包含 64MB / 4KB = 16384 个槽位，需要 14 位的索引来覆盖 (2^14 = 16384)。假设 XArray 的每个节点管理 6 位的索引空间（即拥有 64 个槽位），那么：

两层树 最多能覆盖 6 + 6 = 12 位索引，即 2^12 = 4096 个槽位，不足以覆盖 14 位的范围。
三层树 则能覆盖 6 + 6 + 6 = 18 位索引，即 2^18 个槽位，足以轻松覆盖 14 位的需求。
因此，在这个架构下，一次 Swap Cache 的查找操作，通常需要 3 次 指针走树遍历。

3.终极革命：2MB Cluster 与 Swap Table 的诞生
尽管 64MB 的 address_space 方案有所改善，但它本质上仍是“借来”的架构，锁粒度依然偏大，且树形查找（O(logN)）的开销不容忽视。为此，新一代架构应运而生，它采用了全新的、专为 Swap 设计的管理模式：

管理单元的精细化： 新架构以 Cluster 作为基础管理单元，其大小被巧妙地设计为 2MB。这并非一个随意的数字，它精确地对应了 x86-64 架构下一个 PMD 页表项所能映射的透明大页（THP）的大小。
- O(1) 寻址的Swap Table:
一个 2MB 的 Cluster 包含 512 个 4KB 的槽位。
为了索引这 512 个槽位的 Swap Cache 状态，内核不再使用复杂的树，而是回归到最快的扁平数组——Swap Table。
这个 Swap Table 就是一个拥有 512 项的 unsigned long 数组，其元数据大小恰好是 512 * 8B = 4KB，可以完美地放入一个物理页面中，极大地增强了内存访问的局部性。状态编码的

- 传承与创新：
Swap Table 中的每一个 8 字节条目，都承袭了 XArray 的精妙设计，通过指针的最低位来编码三种不同的状态：
folio指针: 指向真正在 Swap Cache 中的物理页面，其地址最低位必为 0。
shadow条目: 用于追踪近期换出页面的“影子”元数据，其最低位被置为 1。
NULL: 代表槽位为空，或页面不在 Swap Cache 中。
这种设计在不引入任何额外空间开销的前提下，实现了信息的高度压缩与复用。
通过这一系列变革，Swap 机制的查找过程从 O(logN) 的树形遍历，进化为了 O(1) 的直接地址计算，锁的粒度也从 64MB 精细化到了 2MB，这正是新架构带来巨大性能飞跃的根源所在。并且 kairui清理和提供了一些接口便于使用：

```c
/*
 * All swap cache helpers below require the caller to ensure the swap entries
 * used are valid and stablize the device by any of the following ways:
 * - Hold a reference by get_swap_device(): this ensures a single entry is
 *   valid and increases the swap device『s refcount.
 * - Locking a folio in the swap cache: this ensures the folio』s swap entries
 *   are valid and pinned, also implies reference to the device.
 * - Locking anything referencing the swap entry: e.g. PTL that protects
 *   swap entries in the page table, similar to locking swap cache folio.
 * - See the comment of get_swap_device() for more complex usage.
 */
struct folio *swap_cache_get_folio(swp_entry_t entry);
void *swap_cache_get_shadow(swp_entry_t entry);
void swap_cache_add_folio(struct folio *folio, swp_entry_t entry, void **shadow);
void swap_cache_del_folio(struct folio *folio);
/* Below helpers require the caller to lock and pass in the swap cluster. */
void __swap_cache_del_folio(struct swap_cluster_info *ci,
			    struct folio *folio, swp_entry_t entry, void *shadow);
void __swap_cache_replace_folio(struct swap_cluster_info *ci,
				struct folio *old, struct folio *new);
void __swap_cache_clear_shadow(swp_entry_t entry, int nr_ents);

```
至于 swap table 中的寻址则在这里，先找到 cluster 再通过 offset 找到我们的 slot：

```c
static inline struct swap_cluster_info *__swap_offset_to_cluster(
		struct swap_info_struct *si, pgoff_t offset)
{
	VM_WARN_ON_ONCE(percpu_ref_is_zero(&si->users)); /* race with swapoff */
	VM_WARN_ON_ONCE(offset >= si->max);
	return &si->cluster_info[offset / SWAPFILE_CLUSTER];
}

static inline unsigned int swp_cluster_offset(swp_entry_t entry)
{
	return swp_offset(entry) % SWAPFILE_CLUSTER;
}

```

到结尾我们最后来思考一个小的问题，就是管理这部分内容的元数据也是需要开辟宝贵的内存进行存储的，现在的 swap table 刚好是对应一个页面：

```c

static struct swap_table *swap_table_alloc(gfp_t gfp)
{
	struct folio *folio;

	if (!SWP_TABLE_USE_PAGE)
		return kmem_cache_zalloc(swap_table_cachep, gfp);

	folio = folio_alloc(gfp | __GFP_ZERO, 0);
	if (folio)
		return folio_address(folio);
	return NULL;
}

```
结束语
--------------------------------------------------------------------------------

讲解内核内存管理的第四部分到此结束，如果你有任何的问题或者建议，你可以直接发消息给我[twitter](https://twitter.com/0xAX)，也可以给我发[邮件](mailto:anotherworldofworld@gmail.com)或是直接创建一个 [issue](https://github.com/hust-open-atom-club/linux-insides-zh/issues/new)。

**英文不是我的母语。如果你发现我的英文描述有任何问题，请提交一个PR到[linux-insides](https://github.com/hust-open-atom-club/linux-insides-zh).**

相关连接：
--------------------------------------------------------------------------------

* [xarray](https://zhuanlan.zhihu.com/p/11591870676)
* [cluster_alloctor](https://lkml.kernel.org/r/20250313165935.63303-1-ryncsn@gmail.com)
* [swap_table phase I](https://lore.kernel.org/all/20250916160100.31545-3-ryncsn@gmail.com)
