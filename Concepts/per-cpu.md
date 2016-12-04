每CPU变量
================================================================================

每CPU变量是一项内核特性。从它的名字你就可以理解这项特性的意义了。我们可以创建一个变量，然后每个CPU上都会有一个此变量的拷贝。本节我们来看下这个特性，并试着去理解它是如何实现以及工作的。

内核提供了一个创建每CPU变量的API - `DEFINE_PER_CPU` 宏：

```C
#define DEFINE_PER_CPU(type, name) \
        DEFINE_PER_CPU_SECTION(type, name, "")
```

像其它许多每CPU变量一样，这个宏定义在 [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h) 中。现在我们来看下这个特性是如何实现的。

看下 `DECLARE_PER_CPU` 的定义，可以看到它使用了 2 个参数：`type` 和 `name`，因此我们可以这样创建每CPU变量：

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

展开所有的宏，我们得到一个全局的每CPU变量：

```C
__attribute__((section(".data..percpu"))) int per_cpu_n
```

这意味着我们在 `.data..percpu` 段有了一个 `per_cpu_n` 变量，可以在 `vmlinux` 中找到它：

```
.data..percpu 00013a58  0000000000000000  0000000001a5c000  00e00000  2**12
              CONTENTS, ALLOC, LOAD, DATA
```

好，现在我们知道了，当我们使用 `DEFINE_PER_CPU` 宏时，一个在 `.data..percpu` 段中的每CPU变量就被创建了。当内核初始化时，调用 `setup_per_cpu_areas` 函数加载几次 `.data..percpu` 段，每个CPU上对每个段都加载一次。

让我们来看下每CPU区域初始化流程。它从 [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c) 中调用 `setup_per_cpu_areas` 函数开始，这个函数定义在 [arch/x86/kernel/setup_percpu.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup_percpu.c) 中。

```C
pr_info("NR_CPUS:%d nr_cpumask_bits:%d nr_cpu_ids:%d nr_node_ids:%d\n",
        NR_CPUS, nr_cpumask_bits, nr_cpu_ids, nr_node_ids);
```

 `setup_per_cpu_areas` 开始输出CPUs集合的最大个数（这个在内核配置中以 `CONFIG_NR_CPUS` 配置项设置），实际的CPU个数，`nr_cpumask_bits`（对于新的 `cpumask` 操作来说和 `NR_CPUS` 是一样的），还有 `NUMA` 节点个数。

我们可以在dmesg中看到这些输出：

```
$ dmesg | grep percpu
[    0.000000] setup_percpu: NR_CPUS:8 nr_cpumask_bits:8 nr_cpu_ids:8 nr_node_ids:1
```

然后我们检查 `percpu` 第一个块分配器。所有的每CPU区域都是以块进行分配的。第一个块用于静态每CPU变量。Linux内核提供了决定第一个块分配器类型的命令行：`percpu_alloc` 。我们可以在内核文档中读到它的说明。

```
percpu_alloc=	选择要使用哪个每CPU第一个块分配器。
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

如果内核命令行中没有设置 `percpu_alloc` 参数，就会使用 `embed` 分配器，将第一个每CPU块嵌入进带 [memblock](http://0xax.gitbooks.io/linux-insides/content/mm/linux-mm-1.html) 的bootmem。最后一个分配器和第一个块 `page` 分配器一样，只是将第一个块使用 `PAGE_SIZE` 页进行了映射。

如我上面所写，首先我们在 `setup_per_cpu_areas` 中对第一个块分配器检查，检查到第一个块分配器不是page分配器：

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

如前所述，函数 `pcpu_embed_first_chunk` 将第一个percpu块嵌入bootmen，因此我们传递一些参数给 `pcpu_embed_first_chunk`。参数如下：

* `PERCPU_FIRST_CHUNK_RESERVE` - 为静态变量 `percpu` 的保留空间大小；
* `dyn_size` - 动态分配的最少空闲字节；
* `atom_size` - 所有的分配都是这个的整数倍，并以此对齐；
* `pcpu_cpu_distance` - 决定cpus距离的回调函数；
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

If the first chunk allocator is `PCPU_FC_PAGE`, we will use the `pcpu_page_first_chunk` instead of the `pcpu_embed_first_chunk`. After that `percpu` areas up, we setup `percpu` offset and its segment for every CPU with the `setup_percpu_segment` function (only for `x86` systems) and move some early data from the arrays to the `percpu` variables (`x86_cpu_to_apicid`, `irq_stack_ptr` and etc...). After the kernel finishes the initialization process, we will have loaded N `.data..percpu` sections, where N is the number of CPUs, and the section used by the bootstrap processor will contain an uninitialized variable created with the `DEFINE_PER_CPU` macro.

The kernel provides an API for per-cpu variables manipulating:

* get_cpu_var(var)
* put_cpu_var(var)


Let's look at the `get_cpu_var` implementation:

```C
#define get_cpu_var(var)     \
(*({                         \
         preempt_disable();  \
         this_cpu_ptr(&var); \
}))
```

The Linux kernel is preemptible and accessing a per-cpu variable requires us to know which processor the kernel is running on. So, current code must not be preempted and moved to the another CPU while accessing a per-cpu variable. That's why, first of all we can see a call of the `preempt_disable` function then a call of the `this_cpu_ptr` macro, which looks like:

```C
#define this_cpu_ptr(ptr) raw_cpu_ptr(ptr)
```

and

```C
#define raw_cpu_ptr(ptr)        per_cpu_ptr(ptr, 0)
```

where `per_cpu_ptr` returns a pointer to the per-cpu variable for the given cpu (second parameter). After we've created a per-cpu variable and made modifications to it, we must call the `put_cpu_var` macro which enables preemption with a call of `preempt_enable` function. So the typical usage of a per-cpu variable is as follows:

```C
get_cpu_var(var);
...
//Do something with the 'var'
...
put_cpu_var(var);
```

Let's look at the `per_cpu_ptr` macro:

```C
#define per_cpu_ptr(ptr, cpu)                             \
({                                                        \
        __verify_pcpu_ptr(ptr);                           \
         SHIFT_PERCPU_PTR((ptr), per_cpu_offset((cpu)));  \
})
```

As I wrote above, this macro returns a per-cpu variable for the given cpu. First of all it calls `__verify_pcpu_ptr`:

```C
#define __verify_pcpu_ptr(ptr)
do {
	const void __percpu *__vpp_verify = (typeof((ptr) + 0))NULL;
	(void)__vpp_verify;
} while (0)
```

which makes the given `ptr` type of `const void __percpu *`,

After this we can see the call of the `SHIFT_PERCPU_PTR` macro with two parameters. As first parameter we pass our ptr and for second parameter we pass the cpu number to the `per_cpu_offset` macro:

```C
#define per_cpu_offset(x) (__per_cpu_offset[x])
```

which expands to getting the `x` element from the `__per_cpu_offset` array:


```C
extern unsigned long __per_cpu_offset[NR_CPUS];
```

where `NR_CPUS` is the number of CPUs. The `__per_cpu_offset` array is filled with the distances between cpu-variable copies. For example all per-cpu data is `X` bytes in size, so if we access `__per_cpu_offset[Y]`, `X*Y` will be accessed. Let's look at the `SHIFT_PERCPU_PTR` implementation:

```C
#define SHIFT_PERCPU_PTR(__p, __offset)                                 \
         RELOC_HIDE((typeof(*(__p)) __kernel __force *)(__p), (__offset))
```

`RELOC_HIDE` just returns offset `(typeof(ptr)) (__ptr + (off))` and it will return a pointer to the variable.

That's all! Of course it is not the full API, but a general overview. It can be hard to start with, but to understand per-cpu variables you mainly need to understand the  [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/master/include/linux/percpu-defs.h) magic.

Let's again look at the algorithm of getting a pointer to a per-cpu variable:

* The kernel creates multiple `.data..percpu` sections (one per-cpu) during initialization process;
* All variables created with the `DEFINE_PER_CPU` macro will be relocated to the first section or for CPU0;
* `__per_cpu_offset` array filled with the distance (`BOOT_PERCPU_OFFSET`) between `.data..percpu` sections;
* When the `per_cpu_ptr` is called, for example for getting a pointer on a certain per-cpu variable for the third CPU, the `__per_cpu_offset` array will be accessed, where every index points to the required CPU.

That's all.
