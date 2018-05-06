Per-cpu 变量
================================================================================

Per-cpu 变量是一项内核特性。从它的名字你就可以理解这项特性的意义了。我们可以创建一个变量，然后每个 CPU 上都会有一个此变量的拷贝。本节我们来看下这个特性，并试着去理解它是如何实现以及工作的。

内核提供了一个创建 per-cpu 变量的 API - `DEFINE_PER_CPU` 宏：

```C
#define DEFINE_PER_CPU(type, name) \
        DEFINE_PER_CPU_SECTION(type, name, "")
```

正如其它许多处理 per-cpu 变量的宏一样，这个宏定义在 [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h) 中。现在我们来看下这个特性是如何实现的。

看下 `DECLARE_PER_CPU` 的定义，可以看到它使用了 2 个参数：`type` 和 `name`，因此我们可以这样创建 per-cpu 变量：

```C
DEFINE_PER_CPU(int, per_cpu_n)
```

我们传入要创建变量的类型和名字，`DEFINE_PER_CPU` 调用 `DEFINE_PER_CPU_SECTION`，将两个参数和空字符串传递给后者。让我们来看下 `DEFINE_PER_CPU_SECTION` 的定义：

```C
#define DEFINE_PER_CPU_SECTION(type, name, sec)    \
         __PCPU_ATTRS(sec) PER_CPU_DEF_ATTRIBUTES  \
         __typeof__(type) name
```

```C
#define __PCPU_ATTRS(sec)                                                \
         __percpu __attribute__((section(PER_CPU_BASE_SECTION sec)))     \
         PER_CPU_ATTRIBUTES
```

其中 `section` 是:

```C
#define PER_CPU_BASE_SECTION ".data..percpu"
```

当所有的宏展开之后，我们得到一个全局的 per-cpu 变量：

```C
__attribute__((section(".data..percpu"))) int per_cpu_n
```

这意味着我们在 `.data..percpu` 段有了一个 `per_cpu_n` 变量，可以在 `vmlinux` 中找到它：

```
.data..percpu 00013a58  0000000000000000  0000000001a5c000  00e00000  2**12
              CONTENTS, ALLOC, LOAD, DATA
```

好，现在我们知道了，当我们使用 `DEFINE_PER_CPU` 宏时，一个在 `.data..percpu` 段中的 per-cpu 变量就被创建了。内核初始化时，调用 `setup_per_cpu_areas` 函数多次加载 `.data..percpu` 段，每个 CPU 一次。

让我们来看下 per-cpu 区域初始化流程。它从 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 中调用 `setup_per_cpu_areas` 函数开始，这个函数定义在 [arch/x86/kernel/setup_percpu.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup_percpu.c) 中。

```C
pr_info("NR_CPUS:%d nr_cpumask_bits:%d nr_cpu_ids:%d nr_node_ids:%d\n",
        NR_CPUS, nr_cpumask_bits, nr_cpu_ids, nr_node_ids);
```

 `setup_per_cpu_areas` 开始输出在内核配置中以 `CONFIG_NR_CPUS` 配置项设置的最大 CPUs 数，实际的 CPU 个数，`nr_cpumask_bits`（对于新的 `cpumask` 操作来说和 `NR_CPUS` 是一样的），还有 `NUMA` 节点个数。

我们可以在 `dmesg` 中看到这些输出：

```
$ dmesg | grep percpu
[    0.000000] setup_percpu: NR_CPUS:8 nr_cpumask_bits:8 nr_cpu_ids:8 nr_node_ids:1
```

然后我们检查 `per-cpu` 第一个块分配器。所有的 per-cpu 区域都是以块进行分配的。第一个块用于静态 per-cpu 变量。Linux 内核提供了决定第一个块分配器类型的命令行：`percpu_alloc` 。我们可以在内核文档中读到它的说明。

```
percpu_alloc=	选择要使用哪个 per-cpu 第一个块分配器。
		当前支持的类型是 "embed" 和 "page"。
        不同架构支持这些类型的子集或不支持。
        更多分配器的细节参考 mm/percpu.c 中的注释。
        这个参数主要是为了调试和性能比较的。
```

[mm/percpu.c](https://github.com/torvalds/linux/blob/master/mm/percpu.c) 包含了这个命令行选项的处理函数：

```C
early_param("percpu_alloc", percpu_alloc_setup);
```

其中 `percpu_alloc_setup` 函数根据 `percpu_alloc` 参数值设置 `pcpu_chosen_fc` 变量。默认第一个块分配器是 `auto`：

```C
enum pcpu_fc pcpu_chosen_fc __initdata = PCPU_FC_AUTO;
```

如果内核命令行中没有设置 `percpu_alloc` 参数，就会使用 `embed` 分配器，将第一个 per-cpu 块嵌入进带 [memblock](http://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-1.html) 的 bootmem。最后一个分配器和第一个块 `page` 分配器一样，只是将第一个块使用 `PAGE_SIZE` 页进行了映射。

如我上面所写，首先我们在 `setup_per_cpu_areas` 中对第一个块分配器检查，检查到第一个块分配器不是 page 分配器：

```C
if (pcpu_chosen_fc != PCPU_FC_PAGE) {
    ...
    ...
    ...
}
```

如果不是 `PCPU_FC_PAGE`，我们就使用 `embed` 分配器并使用 `pcpu_embed_first_chunk` 函数分配第一块空间。

```C
rc = pcpu_embed_first_chunk(PERCPU_FIRST_CHUNK_RESERVE,
					    dyn_size, atom_size,
					    pcpu_cpu_distance,
					    pcpu_fc_alloc, pcpu_fc_free);
```

如前所述，函数 `pcpu_embed_first_chunk` 将第一个 per-cpu 块嵌入 bootmen，因此我们传递一些参数给 `pcpu_embed_first_chunk`。参数如下：

* `PERCPU_FIRST_CHUNK_RESERVE` - 为静态变量 `per-cpu` 保留空间的大小；
* `dyn_size` - 动态分配的最少空闲字节；
* `atom_size` - 所有的分配都是这个的整数倍，并以此对齐；
* `pcpu_cpu_distance` - 决定 cpus 距离的回调函数；
* `pcpu_fc_alloc` - 分配 `percpu` 页的函数；
* `pcpu_fc_free` - 释放 `percpu` 页的函数。

在调用 `pcpu_embed_first_chunk` 前我们计算好所有的参数：

```C
const size_t dyn_size = PERCPU_MODULE_RESERVE + PERCPU_DYNAMIC_RESERVE - PERCPU_FIRST_CHUNK_RESERVE;
size_t atom_size;
#ifdef CONFIG_X86_64
		atom_size = PMD_SIZE;
#else
		atom_size = PAGE_SIZE;
#endif
```

如果第一个块分配器是 `PCPU_FC_PAGE`，我们用 `pcpu_page_first_chunk` 而不是 `pcpu_embed_first_chunk`。 `per-cpu` 区域准备好以后，我们用 `setup_percpu_segment` 函数设置 `per-cpu` 的偏移和段（只针对 `x86` 系统），并将前面的数据从数组移到 `per-cpu` 变量（`x86_cpu_to_apicid`, `irq_stack_ptr` 等等）。当内核完成初始化进程后，我们就有了N个 `.data..percpu` 段，其中 N 是 CPU 个数，bootstrap 进程使用的段将会包含用 `DEFINE_PER_CPU` 宏创建的未初始化的变量。

内核提供了操作 per-cpu 变量的API：

* get_cpu_var(var)
* put_cpu_var(var)

让我们来看看 `get_cpu_var` 的实现：

```C
#define get_cpu_var(var)     \
(*({                         \
         preempt_disable();  \
         this_cpu_ptr(&var); \
}))
```

Linux 内核是抢占式的，获取 per-cpu 变量需要我们知道内核运行在哪个处理器上。因此访问 per-cpu 变量时，当前代码不能被抢占，不能移到其它的 CPU。如我们所见，这就是为什么首先调用 `preempt_disable` 函数然后调用 `this_cpu_ptr` 宏，像这样：

```C
#define this_cpu_ptr(ptr) raw_cpu_ptr(ptr)
```

以及

```C
#define raw_cpu_ptr(ptr)        per_cpu_ptr(ptr, 0)
```

`per_cpu_ptr` 返回一个指向给定 CPU（第 2 个参数） per-cpu 变量的指针。当我们创建了一个 per-cpu 变量并对其进行了修改时，我们必须调用 `put_cpu_var` 宏通过函数 `preempt_enable` 使能抢占。因此典型的 per-cpu 变量的使用如下：

```C
get_cpu_var(var);
...
//用这个 'var' 做些啥
...
put_cpu_var(var);
```

让我们来看下这个 `per_cpu_ptr` 宏：

```C
#define per_cpu_ptr(ptr, cpu)                             \
({                                                        \
        __verify_pcpu_ptr(ptr);                           \
         SHIFT_PERCPU_PTR((ptr), per_cpu_offset((cpu)));  \
})
```

就像我们上面写的，这个宏返回了一个给定 cpu 的 per-cpu 变量。首先它调用了 `__verify_pcpu_ptr`：

```C
#define __verify_pcpu_ptr(ptr)
do {
	const void __percpu *__vpp_verify = (typeof((ptr) + 0))NULL;
	(void)__vpp_verify;
} while (0)
```

该宏声明了 `ptr` 类型的 `const void __percpu *`。

之后，我们可以看到带两个参数的 `SHIFT_PERCPU_PTR` 宏的调用。第一个参数是我们的指针，第二个参数是传给 `per_cpu_offset` 宏的CPU数：

```C
#define per_cpu_offset(x) (__per_cpu_offset[x])
```

该宏将 `x` 扩展为 `__per_cpu_offset` 数组：

```C
extern unsigned long __per_cpu_offset[NR_CPUS];
```

其中 `NR_CPUS` 是 CPU 的数目。`__per_cpu_offset` 数组以 CPU 变量拷贝之间的距离填充。例如，所有 per-cpu 变量是 `X` 字节大小，所以我们通过 `__per_cpu_offset[Y]` 就可以访问 `X*Y`。让我们来看下 `SHIFT_PERCPU_PTR` 的实现：

```C
#define SHIFT_PERCPU_PTR(__p, __offset)                                 \
         RELOC_HIDE((typeof(*(__p)) __kernel __force *)(__p), (__offset))
```

`RELOC_HIDE` 只是取得偏移量 `(typeof(ptr)) (__ptr + (off))`，并返回一个指向该变量的指针。

就这些了！当然这不是全部的 API，只是一个大概。开头是比较艰难，但是理解 per-cpu 变量你只需理解 [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h) 的奥秘。

让我们再看下获得 per-cpu 变量指针的算法：

* 内核在初始化流程中创建多个 `.data..percpu` 段（一个 per-cpu 变量一个）；
* 所有 `DEFINE_PER_CPU` 宏创建的变量都将重新分配到首个扇区或者 CPU0；
* `__per_cpu_offset` 数组以 (`BOOT_PERCPU_OFFSET`) 和 `.data..percpu` 扇区之间的距离填充；
* 当 `per_cpu_ptr` 被调用时，例如取一个 per-cpu 变量的第三个 CPU 的指针，将访问 `__per_cpu_offset` 数组，该数组的索引指向了所需 CPU。

就这么多了。
