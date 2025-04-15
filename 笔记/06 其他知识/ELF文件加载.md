---
share: "true"
---

# ELF

# ELF文件结构

在[官方文档ELF](https://www.uclibc.org/specs.html)我们可以找到关于[ELF格式的说明书](https://www.uclibc.org/docs/elf.pdf)，书中介绍了ELF的文件结构如下：

![[笔记/01 附件/ELF文件加载/image-20250331160215707.png|image-20250331160215707]]

![[笔记/01 附件/ELF文件加载/elf结构.drawio.png|笔记/01 附件/ELF文件加载/elf结构.drawio.png]]

这里`链接视图`与`执行视图`的区别是，`链接视图`关注的是链接过程，而`执行视图`主要是关注最终的执行。

![[笔记/01 附件/ELF文件加载/elf链接过程.drawio.png|笔记/01 附件/ELF文件加载/elf链接过程.drawio.png]]

![[笔记/01 附件/elf链接过程(详细).png|笔记/01 附件/elf链接过程(详细).png]]

从上面可以看到：

+ `链接视图`主要关注`section`，`执行视图`关注的是程序段
+ 链接时，相同的`section`会被放入到相同的`段(segment)`

# ELF载入到RAM

我们知道程序的运行栈如下：

```c
内存高地址
+----------------------------+ 
|         栈段 (Stack)        |  ← 栈从高地址向低地址增长
+----------------------------+
|            ...              |
+----------------------------+
|         堆段 (Heap)         |  ← 堆从低地址向高地址增长
+----------------------------+
|         .bss 段             |  ← 未初始化数据
+----------------------------+
|         .data 段            |  ← 已初始化的全局变量
+----------------------------+
|        .rodata 段           |  ← 只读数据
+----------------------------+
|         .text 段            |  ← 可执行代码
+----------------------------+
内存低地址
```

那么我们的`执行视图`的ELF文件会被这样加载到`RAM`中。

# ELF数据结构

这里主要是从`ELF Loader`程序来理解ELF的结构。`ELF Loader`是一个程序，它并不是一个具体的代码，而是根据上面的代码进行解读数据结构。为了能够实际的理解，这里我们使用[elfload项目](https://github.com/erincandescent/elfload)来说明。

## ELF头

```c
typedef uint8_t     Elf_Byte;

typedef uint32_t    Elf32_Addr;    /* Unsigned program address */
typedef uint32_t    Elf32_Off;     /* Unsigned file offset */
typedef int32_t     Elf32_Sword;   /* Signed large integer */
typedef uint32_t    Elf32_Word;    /* Unsigned large integer */
typedef uint16_t    Elf32_Half;    /* Unsigned medium integer */

typedef uint64_t    Elf64_Addr;
typedef uint64_t    Elf64_Off;
typedef int32_t     Elf64_Shalf;

#ifdef __alpha__
typedef int64_t     Elf64_Sword;
typedef uint64_t    Elf64_Word;
#else
typedef int32_t     Elf64_Sword;
typedef uint32_t    Elf64_Word;
#endif

typedef int64_t     Elf64_Sxword;
typedef uint64_t    Elf64_Xword;

typedef uint32_t    Elf64_Half;
typedef uint16_t    Elf64_Quarter;
```

```c
typedef struct {
    unsigned char    e_ident[EI_NIDENT];    /* 标识字节（ELF 魔数和元信息） */
    Elf64_Quarter    e_type;                /* 文件类型（如可执行文件、目标文件、共享库等） */
    Elf64_Quarter    e_machine;             /* 机器架构类型（如 x86_64, ARM 等） */
    Elf64_Half       e_version;             /* 版本号（通常为 1） */
    Elf64_Addr       e_entry;               /* 程序入口地址（可执行文件的入口点） */
    Elf64_Off        e_phoff;               /* 程序头表（Program Header Table）偏移 */
    Elf64_Off        e_shoff;               /* 段表（Section Header Table）偏移 */
    Elf64_Half       e_flags;               /* 处理器相关标志 */
    Elf64_Quarter    e_ehsize;              /* ELF 头部大小（sizeof(Elf64_Ehdr)） */
    Elf64_Quarter    e_phentsize;           /* 单个程序头表项的大小 */
    Elf64_Quarter    e_phnum;               /* 程序头表中的条目数 */
    Elf64_Quarter    e_shentsize;           /* 单个段表项的大小 */
    Elf64_Quarter    e_shnum;               /* 段表中的条目数 */
    Elf64_Quarter    e_shstrndx;            /* 段名称字符串表所在的段索引 */
} Elf64_Ehdr;
```

同时，我们可以通过如下命令读取：

```shell
# demo为elf可执行文件
readelf -h demo
```

![[笔记/01 附件/ELF文件加载/image-20250331165553947.png|image-20250331165553947]]

![[笔记/01 附件/ELF文件加载/image-20250331165715540.png|image-20250331165715540]]

## ELF 节(section)头

```c
typedef struct {
    Elf64_Half    sh_name;       /* 节名称（在字符串表中的索引） */
    Elf64_Half    sh_type;       /* 节类型（如代码节、数据节、符号表等） */
    Elf64_Xword   sh_flags;      /* 节标志（如可执行、可写、只读等） */
    Elf64_Addr    sh_addr;       /* 节在内存中的虚拟地址 */
    Elf64_Off     sh_offset;     /* 节在文件中的偏移 */
    Elf64_Xword   sh_size;       /* 节的大小（字节数） */
    Elf64_Half    sh_link;       /* 关联的节索引（如符号表关联的字符串表） */
    Elf64_Half    sh_info;       /* 额外信息（具体含义取决于节类型） */
    Elf64_Xword   sh_addralign;  /* 内存对齐要求（必须是 2 的幂） */
    Elf64_Xword   sh_entsize;    /* 表项大小（如符号表、重定位表的单个条目大小） */
} Elf64_Shdr;
```

使用如下命令读取：

```shell
readelf -S demo
```

![[笔记/01 附件/ELF文件加载/image-20250331170251467.png|image-20250331170251467]]

可以参考下面的信息(其他程序的节表)：

```shell
There are 30 section headers, starting at offset 0x1a50:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000  # 空 Section，用于占位
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238  # 程序解释器路径（动态链接器）
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254  # ABI 版本信息
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000400274  00000274  # 构建 ID，唯一标识可执行文件
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .gnu.hash         GNU_HASH         0000000000400298  00000298  # GNU 哈希表，加速符号查找
       0000000000000024  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000004002c0  000002c0  # 动态符号表
       00000000000000f0  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           00000000004003b0  000003b0  # 动态字符串表（符号名称）
       000000000000008b  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           000000000040043c  0000043c  # 符号版本信息
       0000000000000014  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          0000000000400450  00000450  # 符号版本需求信息
       0000000000000020  0000000000000000   A       6     1     8
  [ 9] .rela.dyn         RELA             0000000000400470  00000470  # 动态重定位表
       0000000000000030  0000000000000018   A       5     0     8
  [10] .rela.plt         RELA             00000000004004a0  000004a0  # PLT（过程链接表）重定位表
       00000000000000c0  0000000000000018  AI       5    23     8
  [11] .init             PROGBITS         0000000000400560  00000560  # 程序初始化代码
       000000000000001a  0000000000000000  AX       0     0     4
  [12] .plt              PROGBITS         0000000000400580  00000580  # 过程链接表（PLT）
       0000000000000090  0000000000000010  AX       0     0     16
  [13] .text             PROGBITS         0000000000400610  00000610  # 程序代码段
       00000000000001e2  0000000000000000  AX       0     0     16
  [14] .fini             PROGBITS         00000000004007f4  000007f4  # 程序终止代码
       0000000000000009  0000000000000000  AX       0     0     4
  [15] .rodata           PROGBITS         0000000000400800  00000800  # 只读数据段
       0000000000000024  0000000000000000   A       0     0     8
  [16] .eh_frame_hdr     PROGBITS         0000000000400824  00000824  # 异常处理框架头
       0000000000000034  0000000000000000   A       0     0     4
  [17] .eh_frame         PROGBITS         0000000000400858  00000858  # 异常处理框架数据
       00000000000000f4  0000000000000000   A       0     0     8
  [18] .init_array       INIT_ARRAY       0000000000600de0  00000de0  # 初始化函数指针数组
       0000000000000008  0000000000000008  WA       0     0     8
  [19] .fini_array       FINI_ARRAY       0000000000600de8  00000de8  # 终止函数指针数组
       0000000000000008  0000000000000008  WA       0     0     8
  [20] .jcr              PROGBITS         0000000000600df0  00000df0  # Java 类注册信息
       0000000000000008  0000000000000000  WA       0     0     8
  [21] .dynamic          DYNAMIC          0000000000600df8  00000df8  # 动态链接信息
       0000000000000200  0000000000000010  WA       6     0     8
  [22] .got              PROGBITS         0000000000600ff8  00000ff8  # 全局偏移表（GOT）
       0000000000000008  0000000000000008  WA       0     0     8
  [23] .got.plt          PROGBITS         0000000000601000  00001000  # PLT 相关的 GOT
       0000000000000058  0000000000000008  WA       0     0     8
  [24] .data             PROGBITS         0000000000601058  00001058  # 数据段
       0000000000000004  0000000000000000  WA       0     0     1
  [25] .bss              NOBITS           0000000000601060  0000105c  # 未初始化数据段
       0000000000000010  0000000000000000  WA       0     0     16
  [26] .comment          PROGBITS         0000000000000000  0000105c  # 编译器注释信息
       000000000000002d  0000000000000001  MS       0     0     1
  [27] .symtab           SYMTAB           0000000000000000  00001090  # 符号表
       0000000000000678  0000000000000018          28    46     8
  [28] .strtab           STRTAB           0000000000000000  00001708  # 字符串表（符号名称）
       000000000000023f  0000000000000000           0     0     1
  [29] .shstrtab         STRTAB           0000000000000000  00001947  # Section 名称字符串表
       0000000000000108  0000000000000000           0     0     1
```

![[笔记/01 附件/ELF文件加载/image-20250331170529817.png|image-20250331170529817]]

## ELF 程序头

```c
typedef struct {
    Elf64_Half    p_type;     /* 段类型 (Program Header Type)，如 PT_LOAD、PT_DYNAMIC 等 */
    Elf64_Half    p_flags;    /* 段标志 (Flags)，如可读、可写、可执行 (R/W/X) */
    Elf64_Off     p_offset;   /* 段在文件中的偏移 (Offset in the ELF file) */
    Elf64_Addr    p_vaddr;    /* 段在内存中的虚拟地址 (Virtual Address in memory) */
    Elf64_Addr    p_paddr;    /* 段在内存中的物理地址 (Physical Address, 一般用于无 MMU 系统) */
    Elf64_Xword   p_filesz;   /* 段在文件中的大小 (Size of segment in file) */
    Elf64_Xword   p_memsz;    /* 段在内存中的大小 (Size of segment in memory) */
    Elf64_Xword   p_align;    /* 段在文件和内存中的对齐 (Alignment of segment) */
} Elf64_Phdr;
```

使用命令：

```shell
readelf -l demo
```

![[笔记/01 附件/ELF文件加载/image-20250331170919123.png|image-20250331170919123]]

可以下面由一个`节->段`的映射表，这就是在链接时，部分`section`会被整合在一起。

```shell
- `LOAD` 段：需要加载到内存的代码和数据段，可能包含 `.text`、`.data` 等 Section。  
- `DYNAMIC` 段：用于动态链接，包含动态库加载信息。  
- `GNU_STACK` 段：指定栈的权限（通常可读写）。
```

![[笔记/01 附件/ELF文件加载/image-20250331171116082.png|image-20250331171116082]]

参考理解(其他程序)：

```shell
Elf file type is EXEC (Executable file)
Entry point 0x4003e0  # 程序入口地址
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040  # 程序头表信息
                 0x00000000000001f8 0x00000000000001f8  R E    8
  INTERP         0x0000000000000238 0x0000000000400238 0x0000000000400238  # 程序解释器路径
                 0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]  # 动态链接器路径
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000  # 可加载段（代码段）
                 0x0000000000000744 0x0000000000000744  R E    200000
  LOAD           0x0000000000000e10 0x0000000000600e10 0x0000000000600e10  # 可加载段（数据段）
                 0x0000000000000218 0x0000000000000220  RW     200000
  DYNAMIC        0x0000000000000e28 0x0000000000600e28 0x0000000000600e28  # 动态链接信息
                 0x00000000000001d0 0x00000000000001d0  RW     8
  NOTE           0x0000000000000254 0x0000000000400254 0x0000000000400254  # 注释信息（ABI、构建 ID）
                 0x0000000000000044 0x0000000000000044  R      4
  GNU_EH_FRAME   0x00000000000005a0 0x00000000004005a0 0x00000000004005a0  # 异常处理框架信息
                 0x000000000000004c 0x000000000000004c  R      4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000  # 栈权限（RW，不可执行）
                 0x0000000000000000 0x0000000000000000  RW     10
  GNU_RELRO      0x0000000000000e10 0x0000000000600e10 0x0000000000600e10  # 重定位只读段
                 0x00000000000001f0 0x00000000000001f0  R      1

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp  # 程序解释器路径
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .plt.got .text .fini .rodata .eh_frame_hdr .eh_frame  # 代码段
   03     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss  # 数据段
   04     .dynamic  # 动态链接信息
   05     .note.ABI-tag .note.gnu.build-id  # 注释信息
   06     .eh_frame_hdr  # 异常处理框架信息
   07     
   08     .init_array .fini_array .jcr .dynamic .got  # 初始化相关段
```

**Section 合并为 Segment 的原因与意义**:

- **减少页面碎片**：未合并时，小块` Section `可能分散占用多个内存页面，导致浪费（页面大小为 4KB 的整数倍）。合并后，连续的` Segment `减少页面使用，优化内存效率。
- **权限管理和安全**：不同` Segment `可定义不同的访问权限（如 `.text` 为可执行只读，`.data` 为可读写），由操作系统通过内存保护机制（如` MMU`）实现，提高程序安全性。
- **性能优化**：连续的` Segment `加载更快，减少虚拟内存映射和页面故障（`page fault`）。
- 知识点扩展：    
  - 现代操作系统使用虚拟内存管理单元（`MMU`）将` Segment `映射到物理内存，并通过页表实现地址转换。
  - 如果程序使用动态库，加载时动态链接器（如 `ld-linux.so`）会解析 `.dynamic` 和 `.got.plt` Section，加载共享库并绑定符号。

## ELF 重定向表

```c
typedef struct {
    Elf64_Xword    r_offset;    /* 需要进行重定位的地址（相对于目标节/段的偏移） */
    Elf64_Xword    r_info;      /* 重定位信息（符号索引和重定位类型的组合） */
    Elf64_Sxword   r_addend;    /* 加数（用于计算最终的重定位值） */
} Elf64_Rela;

typedef struct {
    Elf64_Xword    r_offset;    /* 需要进行重定位的地址（相对于目标节/段的偏移） */
    Elf64_Xword    r_info;      /* 重定位信息（符号索引和重定位类型的组合） */
} Elf64_Rel;
```

`Rel`与`RelA`区别在于：

| 结构体       | r_offset | r_info | r_addend | 适用场景                                                |
| ------------ | -------- | ------ | -------- | ------------------------------------------------------- |
| `Elf64_Rel`  | ✅        | ✅      | ❌        | 目标地址中原本存储了 `addend`，只需计算符号值           |
| `Elf64_Rela` | ✅        | ✅      | ✅        | addend  存储在重定位条目中，适用于位置无关代码`（PIE）` |

**`r_offset`**（重定位地址）
 指定 **需要修改的地址**，即该重定位条目影响的目标位置：

- 对于 **目标文件（ET_REL）**，`r_offset` 是相对于某个节的偏移量。
- 对于 **可执行文件（ET_EXEC）** 或 **共享库（ET_DYN）**，`r_offset` 是 **虚拟地址（VMA）**。

**`r_info`**（重定位信息）
 存储了 **符号索引** 和 **重定位类型**：

- 符号索引：表示该重定位涉及的符号，在符号表（`.symtab`）中查找。
- 重定位类型：定义了如何执行重定位（例如 `R_X86_64_RELATIVE`、`R_AARCH64_JUMP_SLOT`）。

这个字段由 **两个部分** 组成：

```c
#define ELF64_R_SYM(i)    ((i) >> 32)         // 获取符号索引
#define ELF64_R_TYPE(i)   ((i) & 0xFFFFFFFF)  // 获取重定位类型
```

**`r_addend`**（加数）

+ 这个字段是 ELF **Rela** 格式（带 addend）独有的，用于存储计算目标地址时的修正值（addend）。
+ ELF 也有一种 **Rel** 格式，它的 `r_addend` 被省略，改由目标地址中原本存储的值提供。

注意：更加详细的重定向信息请参考https://www.52pojie.cn/thread-1985443-1-1.html#51841344_relocation-table

# ELF 加载示例(代码)

上面提及的`elfload`项目中会进行对`elf`文件--`sample`的加载，加载程序是：`elfloader`

![[笔记/01 附件/ELF文件加载/image-20250331173408923.png|image-20250331173408923]]

通过命令可以加载`sample`，并执行。

```shell
./elfloader sample
```

![[笔记/01 附件/ELF文件加载/image-20250331173534419.png|image-20250331173534419]]

需要注意的一点是，根据上面程序的`RAM`结构，加载`sample`按照正常情况是需要分配`stack`和`heap`的，但是现在的情况是使用`elfloader`的。

而`sample`加载只是将代码加载到指定位置，然后处理重定向代码，最终跳转到`sample`进行执行。

`elfloader`主要由如下几个步骤：

```c
f = fopen(argv[1], "rb"); //读取sample文件

check(el_init(&ctx), "initialising"); //遍历程序头的LOAD段，计算总体需要的内存大小

if (posix_memalign(&buf, ctx.align, ctx.memsz)) // 实际分配内存
    
if (mprotect(buf, ctx.memsz, PROT_READ | PROT_WRITE | PROT_EXEC)) //设置权限
    
check(el_load(&ctx, alloccb), "loading"); //将sample中的LOAD段加载到RAM(buf)中

check(el_relocate(&ctx), "relocating"); //处理重定向段

go(ep); //跳转到sample执行
```

当前主要研究重定向部分。

需要重定向的只有：

![[笔记/01 附件/ELF文件加载/image-20250331174741337.png|image-20250331174741337]]

这里的：

```c
offset = 0x000000004000
Info = 0x000000000008
Type = R_X86_64_RELATIVE
Addend = 4008
```

这里可以使用`objdump`反编译来找到偏移位置是什么：

```shell
objdump -D sample
```

![[笔记/01 附件/ELF文件加载/image-20250331180451215.png|image-20250331180451215]]

可以看到，这里的`0x4000`储存了一个数值`0x4008`，该数值在该函数代码中属于一个外部函数`xputs`的，我们可以看看C源码：

```c
typedef int (*puts_t)(const char *s);
puts_t xputs;
puts_t *pxputs = &xputs;
```

这里`xputs`没有定义，只有声明，所以会认为是外部的函数，需要进行加载到RAM时，进行重新修改该地址，使得指向`xputs`。

那么重定向的工作就是修改`offset`中储存的数值，使其指向正确位置。

从`Info`中取出`type=8（低32位）`，查表可得：

![[笔记/01 附件/ELF文件加载/image-20250331181254914.png|image-20250331181254914]]

相对地址偏移，使用`B+A`，`B`为装入到`RAM`时，程序的基地址，`A`表示`Addend`。所以我们可以看到代码：

```c
el_status el_applyrela(el_ctx *ctx, Elf_RelA *rel)
{
    uint64_t *p = (uint64_t*) (rel->r_offset + ctx->base_load_vaddr);
    uint32_t type = ELF_R_TYPE(rel->r_info);

    switch (type) {
        case R_AMD64_NONE: break;
        case R_AMD64_RELATIVE:
            EL_DEBUG("Applying R_AMD64_RELATIVE reloc @%p\n", p);
            *p = rel->r_addend + ctx->base_load_vaddr;
            break;
        default:
            EL_DEBUG("Bad relocation %u\n", type);
            return EL_BADREL;

    }

    return EL_OK;
}
```

这里先找到`offset`指向的位置`p`，然后修改`*p`的值为`base_load_vaddr + r_addend`。那么重定向则修复完成。



# 参考

https://www.cnblogs.com/revercc/p/16290945.html
https://www.jianshu.com/p/2055bd794e58
https://blog.csdn.net/roger_ranger/article/details/78907308
https://www.jianshu.com/p/46297256d658
https://blog.csdn.net/dai_xiangjun/article/details/123629743
https://blog.csdn.net/zhangmiaoping23/article/details/111314353
https://www.cnblogs.com/sayhellowen/p/802b5b0ad648e1a343dcd0f85513065f.html
https://blog.csdn.net/weixin_28771751/article/details/142357452
https://zhuanlan.zhihu.com/p/678575725
https://blog.csdn.net/make_day_day_up/article/details/146072877 https://developer.aliyun.com/article/1141990
https://blog.csdn.net/weixin_45896211/article/details/138139225?spm=1001.2101.3001.6650.3&utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-3-138139225-blog-146072877.235^v43^pc_blog_bottom_relevance_base1&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~Rate-3-138139225-blog-146072877.235^v43^pc_blog_bottom_relevance_base1&utm_relevant_index=6
https://www.cnblogs.com/wlm-198/articles/17451825.html
https://www.52pojie.cn/thread-1985443-1-1.html
https://www.cnblogs.com/lihaoxiang/p/18418022

https://cloud.tencent.com/developer/article/2503066?policyId=20240001&frompage=seopage

## 重定向参考

https://mp.weixin.qq.com/s/4ZsNOxHUHOeTk9eI1X0Tcg
https://zhuanlan.zhihu.com/p/22497875075
https://blog.csdn.net/u014100559/article/details/132220623

## ELF Loader源码
https://github.com/erincandescent/elfload
https://github.com/embedded2014/elf-loader/blob/master/loader.c
https://github.com/MikhailProg/elf
https://github.com/shinh/tel_ldr