---
share: "true"
---

> 我们以往使用的`printf`，大多支持`%d,%l,%u`，复杂一些的如`%p`，这些写法。但是在内核调试，它并不能完美的符合其多变的场景，因为许多内核的注册方式让我们并不能直接从代码中看出函数的名称。
> 这里使用的是`kernel-6.1.75`版本进行讲解

这里我们不深入研究`printk`的实现原理，这里主要针对的是服务与应用情况，我们可以直接看文件`kernel/lib/vsprintf.c`来分析用法。

# 使用方法
这部分翻译自代码`kernel-source/lib/vsprintf.c:pointer()`中的注释。
这部分是讲解`%p`在内核的扩展用法，下面会讲解`%p`后面的字符的使用方法。
如今我们会处理以下选项：

| 选项                | 简单说明                                            |
| ----------------- | ----------------------------------------------- |
| `S`               | 带偏移量的直接符号指针                                     |
| `s`               | 不带偏移量的直接符号指针                                    |
| `[Ss]R`           | 在上面的基础上加上`__builtin_extract_return_addr`的转换     |
| `S[R]b`           | 在`sS`的基础上加上模块的构建ID号(似乎是需要开启某些`CONFIG`，不然没有额外效果) |
| `[Ff]`            | 早期版本使用的打印指针的符号名，但现在被舍弃，推荐使用`sS`                 |
| `B`               | 针对的是回溯信息直接指向指针，并带有偏移量                           |
| `Bb`              | 上面的基础加上构建ID号                                    |
| `R`               | 解码`struct resource`结构体信息                        |
| `r`               | 与上面一样功能，只是部分打印为`flasgs`                         |
| `b[l]`            | 位图打印                                            |
| `M`               | 打印6字节的MAC地址，有冒号                                 |
| `m`               | 打印6字节的MAC地址，无冒号                                 |
| `MF`              | 打印6字节的MAC地址，短线                                  |
| `[mM]R`           | 打印6字节的MAC地址，反序列                                 |
| `I`               | 打印`IPV4`或者`IPV6`                                |
| `i`               | 打印`IPV4`或者`IPV6`                                |
| `[Ii][4S][hnbl]`  |                                                 |
| `I[6S]c`          | `IPV6`的缩略写法                                     |
| `E[achnops]`      | 转义字符的处理                                         |
| `U`               | 打印`UUID`                                        |
| `V`               | 打印`va_format`中的字符串                              |
| `K`               |                                                 |
| `NF`              | `netdev_features_t`                             |
| `4cc`             | `V4L2`与`DRM`中常用的颜色格式`FourCC`，格式化打印它             |
| `h[CDN]`          | 打印数据以16进制，可选择中间的填充字符：`:`，`-`，`.`                |
| `a[pd]`           | `phys_addr_t`与`dma_addr_t`                      |
| `d[234]`          | 处理路径用的                                          |
| `D[234]`          | 与上面一样                                           |
| `g`               | 打印块(`block_device`)设备的名称                        |
| `t[RT][dt][r][s]` | 打印`rt_time`与`time64_t`                          |
| `C`               | 时钟相关，用来打印`struct clock`                         |
| `Cn`              | 同上                                              |
| `G`               | 页面的`flags`                                      |
| `OF[fnpPcCF]`     | 处理`device_node`用                                |
| `fw[fP]`          | 处理`fw_node`用                                    |
| `x`               | 打印成16进制                                         |
| `[ku]s`           | 与`BPF`相关                                        |
|                   |                                                 |
## 测试示例
### `S s`
```c
void test_function(void) 
{ 
	// 一个简单的函数用于测试 
}
// 使用 %pS 和 %ps 输出函数地址的符号信息 
printk(KERN_INFO "Function symbol with %%pS: %pS\n", test_function);
printk(KERN_INFO "Function symbol with %%ps: %ps\n", test_function);
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106113508389.png|Linux小技巧-printk的扩展打印/file-20241106113508389.png]]
上面的`0x0`表示函数的执行位置(可以参考`call trace`)，然后`0x18`表示函数名(`void test_function(void)`)的长度。

### `R`
```c
printk(KERN_INFO "Function symbol with %%pSR (with offset): %pSR\n", test_function);
printk(KERN_INFO "Function symbol with %%pSr (with offset): %pSr\n", test_function);
printk(KERN_INFO "Function symbol with %%psR (with offset): %psR\n", test_function);
printk(KERN_INFO "Function symbol with %%psr (with offset): %psr\n", test_function);
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106114504800.png|Linux小技巧-printk的扩展打印/file-20241106114504800.png]]

### `Sb`
```c
// 使用 %pSb 和 %pSRb 输出带模块构建 ID 的符号信息 
printk(KERN_INFO "Function symbol with %%pSb (backtrace with build ID): %pSb\n", test_function); 
printk(KERN_INFO "Function symbol with %%pSRb (backtrace with offset and build ID): %pSRb\n", test_function);
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106115145133.png|Linux小技巧-printk的扩展打印/file-20241106115145133.png]]

### `Ff`
与`Ss`基本一致。

### `B`
```c
void (*func_ptr)(void) = test_function;
printk(KERN_INFO "Pointer symbol with %%pB: %pB\n", func_ptr);
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106133606220.png|Linux小技巧-printk的扩展打印/file-20241106133606220.png]]
这里不知道为啥会只有一个地址，可能是没有开启特定的`CONFIG`。

### `Bb`
> 略，与上面的`B`加上`b`一致，`b`表示带上`ID`号


### `R`和`r`
```c
#include <linux/ioport.h>

static struct resource my_resource = { 
	.start = 0x0, 
	.end = 0x1f, 
	.flags = IORESOURCE_MEM | IORESOURCE_PREFETCH, 
};

printk(KERN_INFO "Resource info with %%pR: %pR\n", &my_resource);
printk(KERN_INFO "Resource info with %%pr: %pr\n", &my_resource);
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106134310974.png|Linux小技巧-printk的扩展打印/file-20241106134310974.png]]
这个是用来打印`struct resource` 的，具体各部分：
- **`mem`**：表示这是一个内存资源（Memory Resource）。这个标志通常用于区分不同类型的资源，例如内存、I/O 区域、DMA 通道等。
- **`0x00000000-0x0000001f`**：这是该内存资源的地址范围。从 `0x00000000` 到 `0x0000001f`。表示这块资源从内存地址 `0x00000000` 开始，到 `0x0000001f` 结束，总共有 32 字节的内存区域。
- **`pref`**：这是一个附加标志，表示该内存资源是 **预取（Prefetchable）** 的。预取内存通常指的是那些可以被 CPU 或 DMA 系统预取的内存区域。这样的内存区域通常会有较低的延迟，或者用于存放高优先级的数据。
+ **`flags 0x2200`**：这表示与该资源相关的标志（flags）。`0x2200` 是一个 16 位的十六进制数，表示资源的类型和属性。为了更好地理解它，我们需要将其拆解成不同的标志位。

### `b[l]`
```c
unsigned int bitmap = 0b11001010101100110011001100110011; // 示例位图（32位）
printk(KERN_INFO "二进制格式位图（32位）：%32b\n", bitmap); // 打印32位二进制位图
printk(KERN_INFO "范围列表格式位图：%32bl\n", bitmap); // 打印为范围列表格式
// 使用 * 动态指定字段宽度为 32 位
printk(KERN_INFO "动态指定宽度的位图：%*b\n", 32, bitmap); // 打印为动态宽度二进制位图
```
!!!!!!!!!!!!!!!!!!!!!!!暂时先跳过

### `M`和`m`和`MF`和`[mM]R`
```c
unsigned char mac_addr[6] = {0x00, 0x14, 0x22, 0x01, 0x23, 0x45}; // 示例 MAC 地址
printk(KERN_INFO "MAC 地址: %pM\n", mac_addr);
printk(KERN_INFO "MAC 地址(无冒号): %pm\n", mac_addr);
printk(KERN_INFO "MAC 地址(短线): %pMF\n", mac_addr);
printk(KERN_INFO "反向顺序的 MAC 地址: %pMR\n", mac_addr);  // 蓝牙反向顺序格式
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106144613054.png|Linux小技巧-printk的扩展打印/file-20241106144613054.png]]

### 网络地址
#### `I`与`i`
```c
#include <linux/in.h> 
#include <linux/in6.h>
/* IPv4 地址示例 */
struct in_addr ipv4_addr;
ipv4_addr.s_addr = cpu_to_be32(0xC0A80101);  // 192.168.1.1

/* IPv6 地址示例 */
struct in6_addr ipv6_addr = {
	.s6_addr32 = {
		cpu_to_be32(0x20010db8),
		cpu_to_be32(0x00000000),
		cpu_to_be32(0x00000000),
		cpu_to_be32(0x00000001)
	}
};  // 2001:db8::1

printk(KERN_INFO "IPv4 地址: %pI4\n", &ipv4_addr);  // 使用 %pI4 格式化 IPv4 地址
printk(KERN_INFO "IPv6 地址: %pI6\n", &ipv6_addr);  // 使用 %pI6 格式化 IPv6 地址
printk(KERN_INFO "IPv4 地址: %pi4\n", &ipv4_addr);  // 使用 %pi4 格式化 IPv4 地址
printk(KERN_INFO "IPv6 地址: %pi6\n", &ipv6_addr);  // 使用 %pi6 格式化 IPv6 地址
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106150924348.png|Linux小技巧-printk的扩展打印/file-20241106150924348.png]]
这里`i`表示的是`RAW`格式的IP地址，如果是`IPv4`，那么是使用`.`分开，并加上前导`0`。如果是`IPv6`，则是没有分隔符的样子。

#### `I`与`[S][pfs]`
```c
#include <linux/in.h> 
#include <linux/in6.h>

/* IPv4 socket 地址示例 */
struct sockaddr_in sin4 = {
	.sin_family = AF_INET,
	.sin_port = cpu_to_be16(8080),
	.sin_addr.s_addr = cpu_to_be32(0xC0A80101)  // 192.168.1.1
};

/* IPv6 socket 地址示例 */
struct sockaddr_in6 sin6 = {
	.sin6_family = AF_INET6,
	.sin6_port = cpu_to_be16(8080),
	.sin6_flowinfo = cpu_to_be32(1),
	.sin6_scope_id = 2,
	.sin6_addr = {
		.s6_addr32 = {
			cpu_to_be32(0x20010db8),
			cpu_to_be32(0x00000000),
			cpu_to_be32(0x00000000),
			cpu_to_be32(0x00000001)
		}
	}  // 2001:db8::1
};

/* 1. 基本打印（自动检测IPv4/IPv6） */
printk("Basic sockaddr: %pIS\n", &sin4);
printk("Basic sockaddr: %pIS\n", &sin6);

/* 2. 带端口打印 [p] */
printk("Address with port: %pISp\n", &sin4);
printk("Address with port: %pISp\n", &sin6);

/* 3. IPv6地址带flowinfo打印 [f] */
//[x] printk("IPv4 with flowinfo: %pISf\n", &sin4); IPv4没有该控制字段
printk("IPv6 with flowinfo: %pISf\n", &sin6);

/* 4. IPv6地址带scope打印 [s] */
//[x] printk("IPv4 with scope: %pISs\n", &sin4); IPv4没有该控制字段
printk("IPv6 with scope: %pISs\n", &sin6);

/* 5. 组合所有修饰符 [pfs] */
printk("IPv6 all modifiers: %pISpfs\n", &sin6);
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106152056763.png|Linux小技巧-printk的扩展打印/file-20241106152056763.png]]
格式说明：
+ `%pIS` - 基本格式，自动检测是IPv4还是IPv6
+ 修饰符说明：
	- `p` - 显示端口号
	- `f` - 显示IPv6的flowinfo
	- `s` - 显示IPv6的scope
	- `c` - 显示未压缩的IPv6地址（显示所有的0）
- 传入的必须是 `struct sockaddr *` 类型的指针
- 字节序要注意使用网络字节序（big-endian）

#### `[Ii][4S][hnbl]`
```c
/* 定义 IPv4 地址 192.168.1.1 */
__be32 be_addr = cpu_to_be32(0xC0A80101);
u32 host_addr = 0xC0A80101;

/* 主机字节序 [h] */
printk("Host order (%%pI4h):    %pI4h\n", &host_addr);

/* 网络字节序 [n] */
printk("Network order (%%pI4n):  %pI4n\n", &be_addr);

/* 大端序 [b] */
printk("Big endian (%%pI4b):     %pI4b\n", &be_addr);

/* 小端序 [l] */
printk("Little endian (%%pI4l):   %pI4l\n", &host_addr);
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106152911104.png|Linux小技巧-printk的扩展打印/file-20241106152911104.png]]
主要区别和说明：
1. `%I4` vs `%pI4`：
    - `%I4` 接受直接的 u32/__be32 值
    - `%pI4` 接受指向 IP 地址的指针
2. 字节序修饰符（同样适用）：
    - `h` - 主机字节序
    - `n` - 网络字节序
    - `b` - 大端序
    - `l` - 小端序
#### `I[6S]c`
这个是`IPv6`的缩略写法，具体不操作示例。

### 转义字符
```c
/* 包含特殊字符的缓冲区 */
char special_chars[] = "Hello\n\t\rWorld\x01\x02\x1F";

/* 包含空格和可打印字符的缓冲区 */
char spaces_str[] = "Hello   World\t\tTest";

/* 包含null字符的缓冲区 */
char with_null[] = "Hello\0World\0Test";

/* 包含非ASCII字符的缓冲区 */
char non_ascii[] = "Hello\x80\x90\xFF测试World";

/* 二进制数据 */
unsigned char binary_data[] = {
	0x48, 0x65, 0x6c, 0x6c, 0x6f, /* "Hello" */
	0x00, 0x01, 0x02, 0x03, /* 二进制数据 */
	0x57, 0x6f, 0x72, 0x6c, 0x64  /* "World" */
};

/* ESCAPE_ANY (a) - 转义任何非ASCII字符 */
printk(KERN_INFO "ESCAPE_ANY: %*pEa\n", sizeof(special_chars), special_chars);

/* ESCAPE_SPECIAL (c) - 只转义特殊字符 */
printk(KERN_CONT "ESCAPE_SPECIAL: %*pEc\n", sizeof(special_chars), special_chars);

/* ESCAPE_HEX (h) - 使用十六进制转义 */
printk(KERN_INFO "ESCAPE_HEX: %*pEh\n", sizeof(binary_data),  binary_data);

/* ESCAPE_NULL (n) - 转义null字符 */
printk(KERN_INFO "ESCAPE_NULL: %*pEn\n", sizeof(with_null), with_null);

/* ESCAPE_OCTAL (o) - 使用八进制转义 */
printk(KERN_INFO "ESCAPE_OCTAL: %*pEo\n", sizeof(binary_data), binary_data);

/* ESCAPE_NP (p) - 转义非可打印字符 */
printk(KERN_INFO "ESCAPE_NP: %*pEp\n", sizeof(special_chars), special_chars);

/* ESCAPE_SPACE (s) - 转义空格字符 */
printk(KERN_INFO "ESCAPE_SPACE: %*pEs\n", sizeof(spaces_str), spaces_str);
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106161015069.png|Linux小技巧-printk的扩展打印/file-20241106161015069.png]]

### `U`
```c
#include <linux/uuid.h>
uuid_t test_uuid = UUID_INIT(0x12345678, 0x1234, 0x5678, 0x90, 0xab, 0xcd, 0xef, 0x12, 0x34, 0x56, 0x78);

printk(KERN_INFO "原始 UUID (big endian, lowercase): %pUb\n", &test_uuid);   // 默认大端小写
printk(KERN_INFO "大端 UPPER case: %pUB\n", &test_uuid);                   // 大端大写
printk(KERN_INFO "小端 lower case: %pUl\n", &test_uuid);                   // 小端小写
printk(KERN_INFO "小端 UPPER case: %pUL\n", &test_uuid);                   // 小端大写
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106163617812.png|Linux小技巧-printk的扩展打印/file-20241106163617812.png]]

### `V`
```c
#include <linux/stdarg.h> 
#include <linux/string.h>
void my_printk(const char *fmt, ...)
{
    struct va_format vaf;
    va_list args;
    va_start(args, fmt);
    vaf.fmt = fmt;
    vaf.va = &args;
    printk(KERN_INFO "%pV", &vaf);
    va_end(args);
}
// va_format测试
const char *test_str = "测试字符串";
my_printk("嵌套格式化字符串示例: int=%d, string=%s\n", 42, test_str);
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106164534217.png|Linux小技巧-printk的扩展打印/file-20241106164534217.png]]
这里`%pV`的作用是，打印`struct va_format`结构体中的字符串。

### `K`
> 在内核中，格式化符 `%pK` 专门用于打印内核指针，但与直接用于 `printk` 不同，这一格式符会根据系统配置和进程的权限来控制指针值的显示情况。它的主要用途是保护敏感的内核信息，仅在合适的场合下（例如 `procfs` 和 `sysfs` 中）向有权限的用户显示内核指针，避免潜在的安全风险。
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/slab.h>

static void *test_ptr;

static int proc_show(struct seq_file *m, void *v)
{
    // 使用 %pK 格式化指针
    seq_printf(m, "Kernel pointer: %pK\n", test_ptr);
    return 0;
}

static int proc_open(struct inode *inode, struct file *file)
{
    return single_open(file, proc_show, NULL);
}

static const struct file_operations proc_fops = {
    .owner = THIS_MODULE,
    .open = proc_open,
    .read = seq_read,
    .release = single_release,
};

static int __init ptr_test_init(void)
{
    test_ptr = kmalloc(128, GFP_KERNEL); // 分配一个测试内存块并作为指针使用
    if (!test_ptr) {
        printk(KERN_ERR "Failed to allocate memory\n");
        return -ENOMEM;
    }

    // 创建 /proc/pointer_test 文件用于显示指针
    proc_create("pointer_test", 0, NULL, &proc_fops);
    printk(KERN_INFO "Pointer test module loaded\n");

    return 0;
}

static void __exit ptr_test_exit(void)
{
    remove_proc_entry("pointer_test", NULL);
    kfree(test_ptr);
    printk(KERN_INFO "Pointer test module unloaded\n");
}

module_init(ptr_test_init);
module_exit(ptr_test_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Test Author");
MODULE_DESCRIPTION("Kernel pointer test with %pK format");

```
上面的代码如果是`root`可以看到正常的值，而普通用户则是`00000000`。
但我发现并没有效果。

### `NF`
> 打印`netdev_features_t`

### `4cc`
> 打印`v4l2`或者`DRM`中的`FourCC`代码


### `h[CDN]`
> 打印数据用的，一般用在数据处理
```c
unsigned char *test_buffer;
// 分配 16 字节的缓冲区并初始化
test_buffer = kmalloc(16, GFP_KERNEL);
if (!test_buffer) {
	printk(KERN_ERR "Failed to allocate memory for test buffer\n");
	return -ENOMEM;
}

// 初始化缓冲区的内容
int i;
for(i = 0; i < 16; i++) {
	test_buffer[i] = i + 1;  // 填充 1, 2, 3, ... , 16
}
// 使用 printk 打印缓冲区内容
printk(KERN_INFO "Hex dump with dash separator: %*phD\n", 16, test_buffer);
printk(KERN_INFO "Hex dump with colon separator: %*phC\n", 16, test_buffer);
printk(KERN_INFO "Hex dump with no separator: %*phN\n", 16, test_buffer);
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106172001647.png|Linux小技巧-printk的扩展打印/file-20241106172001647.png]]


### `a[pd]`
+ **P**-`phys_addr_t`
+ **D**-`dma_addr_t`
+ 默认是`P`

### `d[234]`和`D[234]`
> 处理路径用的

假设我们有如下路径：
`/home/user/Documents/projects/linux/kernel/module.c`
如果你使用 `%d2`，则打印出路径的最后两个目录组件：
`projects/linux`

如果你使用 `%d3`，则打印出最后三个目录组件：
`Documents/projects/linux`

如果你使用 `%d4`，则打印出最后四个目录组件：
`home/user/Documents/projects`

### `g`
> 用来打印块设备。
```c
struct block_device *bdev = NULL; // 块设备指针

// 获取系统的第一个块设备，通常是 /dev/sda
bdev = get_bdev("/dev/sda");

if (bdev) {
	// 打印设备名，包括 gendisk 和分区号
	printk(KERN_INFO "Block device: %g\n", bdev);
} else {
	printk(KERN_ERR "Failed to get block device\n");
}
```
未真实验证。

### `t[RT][dt][r][s]`
> 处理时间用的

+ **R**：`struct rtc_time`
+ **T**：`time64_t`

### `C`或者`Cn`
> 打印`struct clk`用。

### `G`
> 用来打印页面的标志位。
- **`%pGp`**：打印页面标志（page flags）。这个标志是一个指向 `unsigned long` 的指针，表示页面的状态信息。例如，页面是否被交换出、是否被映射等。
- **`%pGg`**：打印 GFP 标志（GFP flags）。这是一个指向 `gfp_t` 的指针，表示内存分配时使用的标志位，通常用于控制内存分配器如何处理内存分配请求。例如，是否是紧急分配、是否允许内存分配失败等。
- **`%pGv`**：打印 VMA 标志（VMA flags）。这是一个指向 `unsigned long` 的指针，表示虚拟内存区域（VMA）的标志位，例如内存区域的权限（可读、可写、可执行）等。

### `OF[fnpPcCF]`
> 打印设备树用

```c
/{
	pwm5: pwm@febd0010 {
		compatible = "rockchip,rk3588-pwm", "rockchip,rk3328-pwm";
		reg = <0x0 0xfebd0010 0x0 0x10>;
		#pwm-cells = <3>;
		pinctrl-names = "active";
		pinctrl-0 = <&pwm5m0_pins>;
		clocks = <&cru 84>, <&cru 83>;
		clock-names = "pwm", "pclk";
		status = "disabled";
	};
}

```
```c
#include <linux/of.h>
#include <linux/of_device.h>

//设备树测试
struct device_node *np;
const char *node_name;
const char *compatible;
const char *status;

// 查找设备树中的 serial@feb40000 节点
np = of_find_node_by_path("/chosen");
if (!np) {
	printk(KERN_ERR "Device node serial@feb40000 not found\n");
}
else{
	printk(KERN_INFO "Device node full_name: %pOFf\n", np);
	printk(KERN_INFO "Device node name: %pOFn\n", np);
	printk(KERN_INFO "Device node phandle: %pOFp\n", np);
	printk(KERN_INFO "Device node path: %pOFP\n", np);
	printk(KERN_INFO "Device node flags: %pOFF\n", np);
	printk(KERN_INFO "Device node compatible: %pOFc\n", np);
	printk(KERN_INFO "Full compatible string: %pOFC\n", np);
}
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106181229188.png|Linux小技巧-printk的扩展打印/file-20241106181229188.png]]

### `fw[fP]`
> 打印的是一个`fw_node`
```c
// 部分沿用上面设备树的代码

struct fwnode_handle *fw;
// 获取 fwnode_handle 指针
fw = of_fwnode_handle(np);
// 使用 %pfw 打印固件节点的信息
printk(KERN_INFO "Firmware node information: %pfw\n", fw);
printk(KERN_INFO "Firmware full name: %pfwf\n", fw);
printk(KERN_INFO "Firmware node name: %pfwP\n", fw);
```
![[笔记/01 附件/Linux小技巧-printk的扩展打印/file-20241106181758048.png|Linux小技巧-printk的扩展打印/file-20241106181758048.png]]

### `x`
> 当您确实想要打印地址时，用于打印指针。在使用 %px 打印指针之前，请考虑是否泄露了有关内核内存布局的敏感信息。`%px` 在功能上等同于 `%lx`（或 `%lu`）。`%px` 是首选，因为它更独特，更易于`grep`。如果将来我们需要修改内核处理打印指针的方式，我们将能够更好地找到调用点。


### `[ku]s`
> 应用在`bpf`中
> `k`和说明符`u`用于打印先前探测到的内存，无论是内核内存 (k) 还是用户内存 (u)。后续`s`说明符将打印字符串。如果直接在常规中使用，[`vsnprintf()`](https://www.kernel.org/doc/html/v5.10/core-api/kernel-api.html#c.vsnprintf "vsnprintf")则 (k) 和 (u) 注释会被忽略，但是，如果在 BPF 的 bpf_trace_ 之外使用，则打印（），例如，它读取它指向的内存而不会出现错误。


# 代码简单分析
```c
// kernel-source/lib/vsprintf.c
static noinline_for_stack
char *pointer(const char *fmt, char *buf, char *end, void *ptr,
	      struct printf_spec spec)
{
	//略
}
```


参考链接：
https://www.cnblogs.com/pengdonglin137/p/17868516.html
https://www.kernel.org/doc/html/v5.10/core-api/printk-formats.html?highlight=printk