ELF文件格式
================================================================================

ELF (Executable and Linkable Format)是一种为可执行文件，目标文件，共享链接库和内核转储(core dumps)准备的标准文件格式。 Linux和很多类Unix操作系统都使用这个格式。 让我们来看一下64位ELF文件格式的结构以及内核源码中有关于它的一些定义。

一个ELF文件由以下三部分组成：

* ELF头(ELF header) - 描述文件的主要特性：类型，CPU架构，入口地址，现有部分的大小和偏移等等；

* 程序头表(Program header table) 列举了所有有效的段(segments)和他们的属性。 程序头表需要加载器将文件中的节加载到虚拟内存段中；

* 节头表(Section header table) - 包含对节(sections)的描述。

现在让我们对这些部分有一些更深的了解。

**ELF头(ELF header)**

ELF头(ELF header)位于文件的开始位置。它的主要目的是定位文件的其他部分。文件头主要包含以下字段：

* ELF文件鉴定 - 一个字节数组用来确认文件是否是一个ELF文件，并且提供普通文件特征的信息；
* 文件类型 - 确定文件类型。这个字段描述文件是一个重定位文件，或可执行文件,或...；
* 目标结构；
* ELF文件格式的版本；
* 程序入口地址；
* 程序头表的文件偏移；
* 节头表的文件偏移；
* ELF头(ELF header)的大小；
* 程序头表的表项大小；
* 其他字段...

你可以在内核源码种找到表示ELF64 header的结构体 `elf64_hdr`：

```C
typedef struct elf64_hdr {
	unsigned char	e_ident[EI_NIDENT];
	Elf64_Half e_type;
	Elf64_Half e_machine;
	Elf64_Word e_version;
	Elf64_Addr e_entry;
	Elf64_Off e_phoff;
	Elf64_Off e_shoff;
	Elf64_Word e_flags;
	Elf64_Half e_ehsize;
	Elf64_Half e_phentsize;
	Elf64_Half e_phnum;
	Elf64_Half e_shentsize;
	Elf64_Half e_shnum;
	Elf64_Half e_shstrndx;
} Elf64_Ehdr;
```

这个结构体定义在 [elf.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/elf.h#L220)

**节(sections)**

所有的数据都存储在ELF文件的节(sections)中。我们通过节头表中的索引(index)来确认节(sections)。节头表表项包含以下字段：

* 节的名字；
* 节的类型；
* 节的属性；
* 内存地址；
* 文件中的偏移；
* 节的大小；
* 到其他节的链接；
* 各种各样的信息；
* 地址对齐；
* 这个表项的大小，如果有的话；

而且，在linux内核中结构体 `elf64_shdr` 如下所示:

```C
typedef struct elf64_shdr {
	Elf64_Word sh_name;
	Elf64_Word sh_type;
	Elf64_Xword sh_flags;
	Elf64_Addr sh_addr;
	Elf64_Off sh_offset;
	Elf64_Xword sh_size;
	Elf64_Word sh_link;
	Elf64_Word sh_info;
	Elf64_Xword sh_addralign;
	Elf64_Xword sh_entsize;
} Elf64_Shdr;
```

[elf.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/elf.h#L312)

**程序头表(Program header table)**

All sections are grouped into segments in an executable or shared object file. Program header is an array of structures which describe every segment. It looks like:

```C
typedef struct elf64_phdr {
	Elf64_Word p_type;
	Elf64_Word p_flags;
	Elf64_Off p_offset;
	Elf64_Addr p_vaddr;
	Elf64_Addr p_paddr;
	Elf64_Xword p_filesz;
	Elf64_Xword p_memsz;
	Elf64_Xword p_align;
} Elf64_Phdr;
```

in the linux kernel source code.

`elf64_phdr` defined in the same [elf.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/elf.h#L254).

The ELF object file also contains other fields/structures which you can find in the [Documentation](http://www.uclibc.org/docs/elf-64-gen.pdf). Now let's a look at the `vmlinux` ELF object.

vmlinux
--------------------------------------------------------------------------------

`vmlinux` is also a relocatable ELF object file . We can take a look at it with the `readelf` util. First of all let's look at the header:

```
$ readelf -h  vmlinux
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1000000
  Start of program headers:          64 (bytes into file)
  Start of section headers:          381608416 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         5
  Size of section headers:           64 (bytes)
  Number of section headers:         73
  Section header string table index: 70
```

Here we can see that `vmlinux` is a 64-bit executable file.

We can read from the [Documentation/x86/x86_64/mm.txt](https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/mm.txt#L19):

```
ffffffff80000000 - ffffffffa0000000 (=512 MB)  kernel text mapping, from phys 0
```

We can then look this address up in the `vmlinux` ELF object with:

```
$ readelf -s vmlinux | grep ffffffff81000000
     1: ffffffff81000000     0 SECTION LOCAL  DEFAULT    1 
 65099: ffffffff81000000     0 NOTYPE  GLOBAL DEFAULT    1 _text
 90766: ffffffff81000000     0 NOTYPE  GLOBAL DEFAULT    1 startup_64
```

Note that the address of the `startup_64` routine is not `ffffffff80000000`, but `ffffffff81000000` and now I'll explain why.

We can see following definition in the [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S):

```
    . = __START_KERNEL;
	...
	...
	..
	/* Text and read-only data */
	.text :  AT(ADDR(.text) - LOAD_OFFSET) {
		_text = .;
		...
		...
		...
	}
```

Where `__START_KERNEL` is:

```
#define __START_KERNEL		(__START_KERNEL_map + __PHYSICAL_START)
```

`__START_KERNEL_map` is the value from the documentation - `ffffffff80000000` and `__PHYSICAL_START` is `0x1000000`. That's why address of the `startup_64` is `ffffffff81000000`.

And at last we can get program headers from `vmlinux` with the following command:

```
readelf -l vmlinux

Elf file type is EXEC (Executable file)
Entry point 0x1000000
There are 5 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
                 0x0000000000cfd000 0x0000000000cfd000  R E    200000
  LOAD           0x0000000001000000 0xffffffff81e00000 0x0000000001e00000
                 0x0000000000100000 0x0000000000100000  RW     200000
  LOAD           0x0000000001200000 0x0000000000000000 0x0000000001f00000
                 0x0000000000014d98 0x0000000000014d98  RW     200000
  LOAD           0x0000000001315000 0xffffffff81f15000 0x0000000001f15000
                 0x000000000011d000 0x0000000000279000  RWE    200000
  NOTE           0x0000000000b17284 0xffffffff81917284 0x0000000001917284
                 0x0000000000000024 0x0000000000000024         4

 Section to Segment mapping:
  Segment Sections...
   00     .text .notes __ex_table .rodata __bug_table .pci_fixup .builtin_fw
          .tracedata __ksymtab __ksymtab_gpl __kcrctab __kcrctab_gpl
		  __ksymtab_strings __param __modver 
   01     .data .vvar 
   02     .data..percpu 
   03     .init.text .init.data .x86_cpu_dev.init .altinstructions
          .altinstr_replacement .iommu_table .apicdrivers .exit.text
		  .smp_locks .data_nosave .bss .brk
```

Here we can see five segments with sections list. You can find all of these sections in the generated linker script at - `arch/x86/kernel/vmlinux.lds`.

That's all. Of course it's not a full description of ELF (Executable and Linkable Format), but if you want to know more, you can find the documentation - [here](http://www.uclibc.org/docs/elf-64-gen.pdf)
