---
share: "true"
---


> 需要`RK3588-uboot`源码支持，需要重新编写完善：
> + uboot部分
> + EFI加载过程解读
> + 加载内核时会关闭MMU，中断等操作的清晰讲解
> + uboot下的image.h，efi.h，efi_iamge_loade.c


> 这里会简单分析`uboot`启动末期，以及内核启动初期的代码。

# Image
## 具体的构建指令
在我们进行分析前我们需要对`Image`的组成进行分析，该文件的生成是在：
```makefile
// arch/arm64/boot/Makefile
targets := Image Image.bz2 Image.gz Image.lz4 Image.lzma Image.lzo Image.zst

$(obj)/Image: vmlinux FORCE
	$(call if_changed,objcopy)


// scripts/Makefile.lib
# Objcopy
# ---------------------------------------------------------------------------

quiet_cmd_objcopy = OBJCOPY $@
cmd_objcopy = $(OBJCOPY) $(OBJCOPYFLAGS) $(OBJCOPYFLAGS_$(@F)) $< $@

// Makefile
OBJCOPY     = $(CROSS_COMPILE)objcopy
```
展开后是这样的：
```shell
aarch64-none-linux-gnu-objcopy $(OBJCOPYFLAGS) $(OBJCOPYFLAGS_$(@F)) vmlinux Image
```
可以看到实际上是使用`objcopy`复制`vmlinux`到`Image`。
## `vmlinux`与`Image`的区别
`vmlinux`是一个 **未压缩的内核映像**，通常是通过编译内核源代码生成的。这是一个 **可执行文件**，包含了内核的所有代码、数据和符号，它通常是`ELF`格式（`Executable and Linkable Format`）。`vmlinux` 具有完整的、包含调试符号和其他调试信息的内核映像文件，适用于调试和开发阶段。
它的特点是：
- **格式**：通常是 ELF 格式。
- **内容**：包含内核代码、数据、符号表、调试信息等。
- **生成**：通过 `make` 编译内核时生成。具体来说，执行 `make vmlinux` 或 `make` 会生成 `vmlinux` 文件。
- **用途**：
    - 主要用于 **调试**，因为它包含了符号表和调试信息。
    - 在某些情况下也可以用作生成 `Image` 的基础。
-----
`Image` 是 Linux 内核的 **压缩映像文件**，通常用于引导。它是一个 **压缩过的内核映像**，大小比 `vmlinux` 小，通常用于直接加载到系统中启动内核。`Image` 也可能是 `zImage` 或 `bzImage` 的一个具体实例，取决于配置和架构。
它的特点是：
- **格式**：通常是压缩格式（如 gzip 或 bzip2），因此它的文件名可能是 `zImage`、`bzImage` 或 `Image.gz` 等。
- **内容**：包含了内核代码、压缩后的数据、以及一些引导程序所需的启动信息。
- **生成**：通过 `make` 生成，通常在执行 `make Image` 或 `make bzImage` 时生成。
    - **`zImage`**：通常用于小型设备或嵌入式设备，它是一个较小的压缩内核映像。
    - **`bzImage`**：通常用于支持更大内存的设备，包含更复杂的引导代码。
通过上面可以知道，`Image`实际上是一个`RAW`的可执行文件，仅仅包含少量信息和`内核可执行机器码`。
那么为什么需要分析它，因为在`uboot`跳转到内核时，不单单时简单的跳转到`Image`的第一个指令开始执行，而是会简单的解析`Image`的头部信息，根据此进行跳转。
## Image的结构
在`Image`文件的头部，是一串`PE/COFF`的头，他是`Windows`上`exe`常用的格式，也是`EFI`的一种实现。
![[笔记/01 附件/Linux源码分析-uboot切换到内核过程/file-20241113111358869.png|笔记/01 附件/Linux源码分析-uboot切换到内核过程/file-20241113111358869.png]]
![[笔记/01 附件/Linux源码分析-uboot切换到内核过程/file-20241113111428873.png|笔记/01 附件/Linux源码分析-uboot切换到内核过程/file-20241113111428873.png]]

# UBOOT的启动
## 启动流程
## 解析Image



# 内核的启动
## `head.S`
该文件位于`arch/arm64/kernel`中，内核的第一条指令在其中。同时它在编译后，会产生`PE/COFF`头。我们来看看该部分代码：
```assembly
/*
 * Kernel startup entry point.
 * ---------------------------
 *
 * The requirements are:
 *   MMU = off, D-cache = off, I-cache = on or off,
 *   x0 = physical address to the FDT blob.
 *
 * Note that the callee-saved registers are used for storing variables
 * that are useful before the MMU is enabled. The allocations are described
 * in the entry routines.
 */
	__HEAD
	/*
	 * DO NOT MODIFY. Image header expected by Linux boot-loaders.
	 */
	efi_signature_nop			// special NOP to identity as PE/COFF executable
	b	primary_entry			// branch to kernel start, magic
	.quad	0				// Image load offset from start of RAM, little-endian
	le64sym	_kernel_size_le			// Effective size of kernel image, little-endian
	le64sym	_kernel_flags_le		// Informative flags, little-endian
	.quad	0				// reserved
	.quad	0				// reserved
	.quad	0				// reserved
	.ascii	ARM64_IMAGE_MAGIC		// Magic number
	.long	.Lpe_header_offset		// Offset to the PE header.

	__EFI_PE_HEADER

	__INIT
```
这里需要结合`arch/arm64/kernel/vmlinux.lds`来分析：
```assembly
OUTPUT_ARCH(aarch64)
ENTRY(_text)
jiffies = jiffies_64;
PECOFF_FILE_ALIGNMENT = 0x200;
SECTIONS
{
 /DISCARD/ : { *(.exitcall.exit) *(.discard) *(.discard.*) *(.modinfo) *(.gnu.version*) }
 /DISCARD/ : {
  *(.interp .dynamic)
  *(.dynsym .dynstr .hash .gnu.hash)
 }
 . = ((((-(((1)) << ((((39))) - 1)))) + (0x08000000)));
 .head.text : {
  _text = .;
  KEEP(*(.head.text))
 }
```
接着我们在`include/linux/init.h`中找到对`__HEAD`的宏定义：
```c
#define __HEAD		.section	".head.text","ax"
```
结合链接脚本来看，也就是说`__HEAD`接下来的代码会放在最开始的位置。这里的`.`表示当前位置，而`_text`和`KEEP(*(.head.text))`都意味着`section:.head.text`的数据会从首部线性排列。
我们可以验证，在`Image`中的首部是`PE的标志(MZ)`，接着是`code1`，`text_offset`，`image_size`。使用`HEX`查看：
![[笔记/01 附件/Linux源码分析-uboot切换到内核过程/file-20241113134034737.png|笔记/01 附件/Linux源码分析-uboot切换到内核过程/file-20241113134034737.png]]
上面解析的信息是（注释是对应`head.S`中的指令）：
```c
4D 5A 40 FA = "MZ"    // 对应efi_signature_nop(ccmp x18, #0, #0xd, pl)
19 b4 4f 14 = 指令码   // b primary_entry
00 00 00 00 00 00 00 00 = 内核加载偏移量 // .quad 0
00 00 de 01 00 00 00 00 = 内核大小      // le64sym	_kernel_size_le
......
```
上面的`ccmp x18, #0, #0xd, pl`指令通过比较巧妙的操作，使得编译生成`MZ`的头。所以`PE/COFF`头实际上是由汇编代码配合`链接脚本`生成的。
在解析完成`EFI`的头部后，就会执行`code0`,`code1`，我们根据文档[内核(ARM64)booting](https://www.kernel.org/doc/Documentation/arm64/booting.txt)可以知道：
![[笔记/01 附件/Linux源码分析-uboot切换到内核过程/file-20241113134243237.png|笔记/01 附件/Linux源码分析-uboot切换到内核过程/file-20241113134243237.png]]
所以在`b primary_entry`后，我们可以按正常的代码调用逻辑得到以下的调用栈：
```c
primary_entry
	__primary_switch
		__primary_switched
			start_kernel
```
这里就跳入了`start_kernel`，它是位于`init/main.c`中的一个函数，含有启动内核各部分的主流程函数。
到这里意味着，`uboot`已经切换到内核，并且进入主初始化程序。
# 参考
https://krinkinmu.github.io/2023/08/21/how-u-boot-loads-linux-kernel.html
https://www.kernel.org/doc/Documentation/arm64/booting.txt
https://www.cnblogs.com/Iflyinsky/p/18149167
https://zhuanlan.zhihu.com/p/343176381
https://docs.kernel.org/admin-guide/efi-stub.html
https://blog.csdn.net/jasonactions/article/details/111223097
https://community.nxp.com/pwmxy87654/attachments/pwmxy87654/Layerscape%40tkb/153/1/ARM64%20Kernel%20Booting%20Process.pdf