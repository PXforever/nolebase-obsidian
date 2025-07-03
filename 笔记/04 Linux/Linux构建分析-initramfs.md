---
share: "true"
---

> `initramfs`是是 **Initial RAM Filesystem** 的缩写，中文通常称为“初始内存文件系统”。它是在 Linux 系统启动初期，挂载根文件系统之前加载并运行的临时根文件系统。以下是关于 `initramfs` 的完整解释：
- **引导阶段的最小文件系统**：  
    `initramfs` 提供了一个包含基本工具和脚本的最小化环境，允许内核执行早期的初始化任务。
- **挂载真正的根文件系统**：  
    在系统启动初期，内核尚无法直接访问磁盘上的根文件系统（例如 ext4、xfs、NFS 等），因此通过 `initramfs` 加载所需驱动和脚本来找到并挂载真正的根文件系统。
- **设备驱动初始化**：  
    对于某些需要额外驱动才能识别的设备（如 SATA 控制器、RAID、LVM、加密分区），`initramfs` 可提供相应模块进行初始化。
- **支持多种启动方案**：  
    包括网络引导（NFS root）、挂载 USB、使用 squashfs 压缩根文件系统、挂载 A/B 分区等。
 `initramfs` 与 `initrd` 的区别：

|特性|initramfs|initrd|
|---|---|---|
|类型|CPIO 格式存档（内存中展开）|使用块设备的映像（挂载为loop）|
|挂载方式|被内核解压至内存中（tmpfs）|被挂载为临时块设备|
|内核支持|2.6 以后内核默认使用 initramfs|旧内核仍可使用 initrd|
在这里我们将会分为如下几个部分来探究该内容：
+ `Makefile`是如何将`initramfs`打包进`Imnage`的
+ 使用`busybox`制作`initramfs`
+ `initramfs`如何被载入到系统中，并且挂载
# initramfs 与 Makefile
`initramfs`构建目录在`kernel_source/usr`下，我们可以阅读该目录下的`Makefile`:
```makefile
# SPDX-License-Identifier: GPL-2.0
#
# kbuild file for usr/ - including initramfs image
# 用于usr/目录的kbuild文件 - 包含initramfs镜像的构建

# 定义压缩方法的映射关系
# 根据内核配置选项选择相应的压缩算法
compress-y					:= copy    # 默认使用copy（无压缩）
compress-$(CONFIG_INITRAMFS_COMPRESSION_GZIP)	:= gzip    # 如果启用GZIP压缩配置
compress-$(CONFIG_INITRAMFS_COMPRESSION_BZIP2)	:= bzip2   # 如果启用BZIP2压缩配置
compress-$(CONFIG_INITRAMFS_COMPRESSION_LZMA)	:= lzma    # 如果启用LZMA压缩配置
compress-$(CONFIG_INITRAMFS_COMPRESSION_XZ)	:= xzmisc  # 如果启用XZ压缩配置
compress-$(CONFIG_INITRAMFS_COMPRESSION_LZO)	:= lzo     # 如果启用LZO压缩配置
compress-$(CONFIG_INITRAMFS_COMPRESSION_LZ4)	:= lz4     # 如果启用LZ4压缩配置
compress-$(CONFIG_INITRAMFS_COMPRESSION_ZSTD)	:= zstd    # 如果启用ZSTD压缩配置

# 定义构建目标
# 只有在启用块设备initrd支持时才构建initramfs_data.o
obj-$(CONFIG_BLK_DEV_INITRD) := initramfs_data.o

# 定义依赖关系
# initramfs_data.o依赖于initramfs_inc_data（压缩后的数据）
$(obj)/initramfs_data.o: $(obj)/initramfs_inc_data

# 获取initramfs源文件路径配置
ramfs-input := $(CONFIG_INITRAMFS_SOURCE)
cpio-data :=  # 初始化cpio数据变量

# 如果CONFIG_INITRAMFS_SOURCE为空，使用默认的initramfs内容
# 这会生成一个包含基本内容的小型initramfs
ifeq ($(ramfs-input),)
ramfs-input := $(srctree)/$(src)/default_cpio_list
endif

# 检查是否只指定了一个输入文件
ifeq ($(words $(ramfs-input)),1)

# 如果CONFIG_INITRAMFS_SOURCE指定的单个文件以.cpio结尾
# 直接将其用作initramfs，无需重新生成
ifneq ($(filter %.cpio,$(ramfs-input)),)
cpio-data := $(ramfs-input)
endif

# 如果CONFIG_INITRAMFS_SOURCE指定的单个文件以.cpio.*结尾
# （如.cpio.gz、.cpio.bz2等），直接使用它作为initramfs
# 并避免重复压缩（设置compress-y为copy）
ifeq ($(words $(subst .cpio.,$(space),$(ramfs-input))),2)
cpio-data := $(ramfs-input)
compress-y := copy  # 避免重复压缩已压缩的文件
endif

endif

# 对于其他情况，需要根据CONFIG_INITRAMFS_SOURCE指定的内容
# 生成initramfs cpio归档文件
ifeq ($(cpio-data),)

# 定义生成的cpio数据文件路径
cpio-data := $(obj)/initramfs_data.cpio

# 声明需要构建的主机程序（在主机系统上运行的工具）
hostprogs := gen_init_cpio

# .initramfs_data.cpio.d文件用于：
# 1. 识别initramfs中包含的所有文件
# 2. 检测是否有文件被添加或删除
# 3. 通过目录时间戳更新来识别已删除的文件
# 4. 依赖列表由gen_initramfs.sh -l生成
-include $(obj)/.initramfs_data.cpio.d

# 不要尝试更新initramfs中包含的文件
# 这些文件作为依赖项，但不需要重新构建
$(deps_initramfs): ;

# 定义构建initramfs的命令
# quiet_cmd_initfs: 安静模式下显示的消息
# cmd_initfs: 实际执行的命令
quiet_cmd_initfs = GEN     $@
      cmd_initfs = \
	$(CONFIG_SHELL) $< -o $@ -l $(obj)/.initramfs_data.cpio.d \
	$(if $(CONFIG_INITRAMFS_ROOT_UID), -u $(CONFIG_INITRAMFS_ROOT_UID)) \
	$(if $(CONFIG_INITRAMFS_ROOT_GID), -g $(CONFIG_INITRAMFS_ROOT_GID)) \
	$(if $(KBUILD_BUILD_TIMESTAMP), -d "$(KBUILD_BUILD_TIMESTAMP)") \
	$(ramfs-input)

# 命令参数解释：
# $(CONFIG_SHELL): 使用配置的shell（通常是bash）
# $<: 第一个依赖项（gen_initramfs.sh脚本）
# -o $@: 输出文件为目标文件
# -l $(obj)/.initramfs_data.cpio.d: 生成依赖列表文件
# -u $(CONFIG_INITRAMFS_ROOT_UID): 设置root用户ID（如果配置了）
# -g $(CONFIG_INITRAMFS_ROOT_GID): 设置root组ID（如果配置了）
# -d "$(KBUILD_BUILD_TIMESTAMP)": 设置构建时间戳（如果配置了）
# $(ramfs-input): 输入源文件或目录

# 重新构建initramfs_data.cpio的条件：
# 1) 任何包含的文件比initramfs_data.cpio更新
# 2) 包含的文件列表发生变化（添加或删除文件）
# 3) gen_init_cpio工具比initramfs_data.cpio更新
# 4) gen_initramfs.sh的参数发生变化
$(obj)/initramfs_data.cpio: $(src)/gen_initramfs.sh $(obj)/gen_init_cpio $(deps_initramfs) FORCE
	$(call if_changed,initfs)

endif

# 生成最终的压缩数据文件
# 使用前面定义的压缩方法（compress-y变量）对cpio数据进行压缩
$(obj)/initramfs_inc_data: $(cpio-data) FORCE
	$(call if_changed,$(compress-y))

# 声明构建目标，确保它们被正确清理
targets += initramfs_data.cpio initramfs_inc_data

# 如果启用了UAPI头文件测试，包含include子目录
subdir-$(CONFIG_UAPI_HEADER_TEST) += include
```
**核心功能：**
1. **压缩算法选择** - 根据内核配置选择合适的压缩方法
2. **源文件处理** - 支持多种initramfs源格式（目录、cpio文件等）
3. **智能构建** - 避免不必要的重复压缩和构建
4. **依赖管理** - 跟踪文件变化，只在必要时重新构建
**关键特性：**
- 支持7种不同的压缩算法（gzip、bzip2、lzma、xz、lzo、lz4、zstd）
- 自动检测已压缩的cpio文件，避免重复压缩
- 通过依赖文件(.d)跟踪所有包含的文件
- 支持自定义root **用户/组ID**和构建时间戳
一般设置`deconfig`会这样设置：
```ini
# For Initramfs 
CONFIG_INITRAMFS_SOURCE="/home/px/busybox_for_os_update/image_out/initramfs.cpio"
CONFIG_INITRAMFS_COMPRESSION_NONE=y
CONFIG_INITRAMFS_ROOT_GID=1000
CONFIG_INITRAMFS_ROOT_UID=1000
#CONFIG_BLK_DEV_RAM=y
#CONFIG_INITRAMFS_FORCE=y
CONFIG_CMDLINE_FORCE=y
CONFIG_BLK_DEV_INITRD=y
CONFIG_INITRAMFS_COMPRESSION_XZ=y
```
注意这里的`CONFIG_INITRAMFS_SOURCE`需要加入双引号，否则可能会`initramfs`无法编入`kernel`文件中。

# busybox制作initramfs
制作一个`initramfs`，首先的是一个小型`rootfs`，因为`kernel size`通常是有限制的，制作过程实际上很简单，只需要按照标准的`busybox`构建即可。需要注意的是，`console`是必须正确配置的，否则会出现错误。
最终`busy_rootfs`目录，可以使用如下命令进行打包为`cpio`格式：
```shell
cd busy_rootfs
find . -print0 | cpio --null -ov --format=newc > ../initramfs.cpio
# 当然也可以压缩initramfs.cpio.gz
gzip -9 initramfs.cpio
```
# 加载过程分析
`initramfs`是一个`rootfs`，所以它最终需要进行挂载到根目录。它的加载过程可以如下的代码流程执行：
![[笔记/01 附件/Linux构建分析-initramfs/initramfs与文件系统加载过程.drawio.png|笔记/01 附件/Linux构建分析-initramfs/initramfs与文件系统加载过程.drawio.png]]
根据上图，我们可以清晰看到，`rootfs`文件系统的组织结构会在一开始就构建好。接着通过`rdinit=/sbin/init`这个异步的`__init_call`启动加载`initramfs`，并解压到前面的文件系统中。
最终在`if (init_eaccess(ramdisk_execute_command) != 0)`中判断是否使用`ramdisk`中的初始化脚本。如果没有则使用`prepare_namespace`，启动外部的`rootfs(由root=/dev/xxxx)`设置的。
这里面有几个环境变量需要理解：
```shell
rdinit=/sbin/init，指示ramdisk中的(initramfs)init进程位置，一旦有这个环境变量，那么会优先启动initramfs。
root=/dev/mmcblk0p1, 指示外部rootfs的设备位置。
init=/sbin/init, 外部rootfs的init脚本位置，当然也是可以不设置的，系统会自动搜索几个固定位置。
```

为了更好的理解`initramfs`加载过程，这里会定向的详细分析`initramfs`从flash中读取到RAM的过程。
在`kernel_source/init/initramfs.c`中，`rootfs_initcall(populate_rootfs);`
是一个`init_call`，所以在启动时，会自动运行。
```c
/**
 * populate_rootfs - 填充根文件系统
 * 
 * 这个函数负责初始化和填充initramfs作为初始根文件系统
 * 使用异步机制来提高启动性能
 */
static int __init populate_rootfs(void)
{
	/* 
	 * 异步调度do_populate_rootfs函数来处理initramfs
	 * - do_populate_rootfs: 实际执行initramfs解压和填充的工作函数
	 * - NULL: 传递给工作函数的参数(这里不需要参数)
	 * - &initramfs_domain: 指定异步执行的域，用于管理相关的异步任务
	 * - initramfs_cookie: 保存异步任务的句柄，用于后续等待或管理
	 */
	initramfs_cookie = async_schedule_domain(do_populate_rootfs, NULL,
						 &initramfs_domain);
	
	/*
	 * 启用用户模式帮助程序机制
	 * 允许内核调用用户空间程序来完成某些任务
	 * 这在initramfs环境中是必需的
	 */
	usermodehelper_enable();
	
	/*
	 * 检查是否需要同步等待initramfs处理完成
	 * initramfs_async为false时表示需要同步等待
	 * 这通常发生在某些特殊配置或调试模式下
	 */
	if (!initramfs_async)
		wait_for_initramfs();  /* 阻塞等待initramfs异步任务完成 */
	
	return 0;  /* 返回成功状态 */
}

/**
 * do_populate_rootfs - 实际执行根文件系统填充工作的函数
 * @unused: 未使用的参数
 * @cookie: 异步任务的标识符
 * 
 * 这是异步执行的工作函数，负责解压和设置initramfs作为初始根文件系统
 */
 /*
 现代系统主要使用initramfs，因为它更高效、更灵活
 传统initrd主要用于兼容老系统或特殊需求
 CONFIG_BLK_DEV_RAM是一个兼容性配置选项，主要作用是：
	支持传统的initrd格式
	提供RAM磁盘功能
	在initramfs解压失败时提供回退机制
	维持与老版本bootloader的兼容性
*/
static void __init do_populate_rootfs(void *unused, async_cookie_t cookie)
{
	/* 
	 * 加载内置的initramfs
	 * __initramfs_start: 内嵌在内核中的initramfs数据起始地址
	 * __initramfs_size: 内嵌initramfs的大小
	 * 这些符号在链接时由链接脚本定义
	 */
	char *err = unpack_to_rootfs(__initramfs_start, __initramfs_size);
	if (err)
		panic_show_mem("%s", err); /* 内置initramfs解压失败，系统无法继续启动 */
	
	/*
	 * 检查是否需要处理外部initrd：
	 * - 如果没有外部initrd (initrd_start == 0)
	 * - 或者强制只使用内置initramfs (CONFIG_INITRAMFS_FORCE=y)
	 * 则跳过外部initrd处理，直接到清理阶段
	 */
	if (!initrd_start || IS_ENABLED(CONFIG_INITRAMFS_FORCE))
		goto done;
	
	/* 根据配置打印不同的提示信息 */
	if (IS_ENABLED(CONFIG_BLK_DEV_RAM))
		printk(KERN_INFO "Trying to unpack rootfs image as initramfs...\n");
	else
		printk(KERN_INFO "Unpacking initramfs...\n");
	
	/*
	 * 尝试解压外部initrd作为initramfs
	 * initrd_start: 外部initrd在内存中的起始地址
	 * initrd_end - initrd_start: 外部initrd的大小
	 */
	err = unpack_to_rootfs((char *)initrd_start, initrd_end - initrd_start);
	if (err) {
		/* 外部initrd解压失败的处理 */
#ifdef CONFIG_BLK_DEV_RAM
		/* 
		 * 如果支持RAM块设备，尝试将initrd作为传统的块设备镜像处理
		 * 而不是作为initramfs格式
		 */
		populate_initrd_image(err);
#else
		/* 如果不支持RAM块设备，打印错误信息 */
		printk(KERN_EMERG "Initramfs unpacking failed: %s\n", err);
#endif
	}

done:
	/*
	 * 清理initrd占用的内存
	 * 需要考虑crashkernel预留区域的重叠情况：
	 * - 如果initrd区域与crashkernel预留区域重叠
	 * - 只释放不属于crashkernel区域的内存部分
	 * - 避免影响内核崩溃转储功能
	 */
	if (!do_retain_initrd && initrd_start && !kexec_free_initrd())
		free_initrd_mem(initrd_start, initrd_end);
	
	/* 清除initrd的起始和结束地址，标记已处理完成 */
	initrd_start = 0;
	initrd_end = 0;
	
	/*
	 * 执行延迟的文件关闭操作
	 * 在initramfs设置过程中可能有一些文件操作被延迟
	 */
	flush_delayed_fput();
	
	/*
	 * 运行当前任务的工作队列
	 * 确保所有相关的任务都被完成
	 */
	task_work_run();
}

/*
 * 函数作用：将压缩的initramfs数据解压到根文件系统
 * 这个函数是Linux内核启动过程中的关键函数，负责解压initramfs（初始RAM文件系统）
 * 并将其内容写入到根文件系统中，为系统启动提供初始的文件系统环境
 * 
 * 参数：
 * - buf: 指向压缩数据的缓冲区
 * - len: 压缩数据的长度
 * 
 * 返回值：
 * - 成功时返回NULL，失败时返回错误信息字符串
 */
static char * __init unpack_to_rootfs(char *buf, unsigned long len)
{
	long written;                    // 已写入的字节数
	decompress_fn decompress;        // 解压函数指针
	const char *compress_name;       // 压缩方法名称
	static __initdata char msg_buf[64];  // 静态错误信息缓冲区
	
	// 分配必要的缓冲区内存
	header_buf = kmalloc(110, GFP_KERNEL);                              // 文件头缓冲区
	symlink_buf = kmalloc(PATH_MAX + N_ALIGN(PATH_MAX) + 1, GFP_KERNEL); // 符号链接缓冲区
	name_buf = kmalloc(N_ALIGN(PATH_MAX), GFP_KERNEL);                   // 文件名缓冲区
	
	// 检查内存分配是否成功，失败则触发panic
	if (!header_buf || !symlink_buf || !name_buf)
		panic_show_mem("can't allocate buffers");
	
	// 初始化状态变量
	state = Start;          // 设置初始状态为Start
	this_header = 0;        // 当前文件头位置
	message = NULL;         // 错误信息指针
	
	// 主循环：处理压缩数据直到完成或遇到错误
	while (!message && len) {
		loff_t saved_offset = this_header;  // 保存当前文件头位置
		
		// 检查是否遇到填充字节（'0'）且位置对齐
		if (*buf == '0' && !(this_header & 3)) {
			state = Start;                   // 重置状态
			written = write_buffer(buf, len); // 写入缓冲区数据
			buf += written;                  // 更新缓冲区指针
			len -= written;                  // 更新剩余长度
			continue;
		}
		
		// 跳过空字节
		if (!*buf) {
			buf++;           // 移动到下一个字节
			len--;           // 减少剩余长度
			this_header++;   // 更新头部位置
			continue;
		}
		
		// 重置头部位置
		this_header = 0;
		
		// 检测压缩方法并获取对应的解压函数
		decompress = decompress_method(buf, len, &compress_name);
		pr_debug("Detected %s compressed data\n", compress_name);
		
		if (decompress) {
			// 如果找到了解压函数，执行解压操作
			int res = decompress(buf, len, NULL, flush_buffer, NULL,
				   &my_inptr, error);
			if (res)
				error("decompressor failed");  // 解压失败
		} else if (compress_name) {
			// 如果识别了压缩方法但没有对应的解压函数
			if (!message) {
				snprintf(msg_buf, sizeof msg_buf,
					 "compression method %s not configured",
					 compress_name);
				message = msg_buf;  // 设置错误信息
			}
		} else
			// 无效的压缩格式
			error("invalid magic at start of compressed archive");
		
		// 检查解压后的状态
		if (state != Reset)
			error("junk at the end of compressed archive");
		
		// 更新位置和缓冲区指针
		this_header = saved_offset + my_inptr;
		buf += my_inptr;
		len -= my_inptr;
	}
	
	// 设置目录时间戳
	dir_utime();
	
	// 释放分配的内存
	kfree(name_buf);
	kfree(symlink_buf);
	kfree(header_buf);
	
	return message;  // 返回错误信息（成功时为NULL）
}

/*
 * 写入缓冲区函数 - initramfs数据处理的核心函数
 * 
 * 函数作用：
 * 这个函数是initramfs解压过程中的状态机驱动函数
 * 它通过状态机模式来解析和处理initramfs数据流中的各种内容
 * 包括文件头、文件数据、目录创建、符号链接等操作
 * 
 * 参数：
 * - buf: 指向待处理数据的缓冲区
 * - len: 缓冲区中数据的长度
 * 
 * 返回值：
 * - 返回实际处理的字节数（len - byte_count）
 * 
 * 工作原理：
 * 使用状态机模式，根据当前状态调用相应的处理函数
 * 每个状态对应initramfs格式中的不同阶段（如读取头部、处理文件数据等）
 */
static long __init write_buffer(char *buf, unsigned long len)
{
	byte_count = len;           // 设置剩余待处理的字节数
	victim = buf;               // 设置当前处理的数据指针（victim意为"待处理的数据"）
	
	// 状态机循环：持续执行当前状态对应的处理函数
	// actions[state]()返回0表示需要继续处理，返回非0表示状态处理完成
	while (!actions[state]())
		;
	
	// 返回已处理的字节数 = 总长度 - 剩余未处理的字节数
	return len - byte_count;
}


/*
 * 刷新缓冲区函数 - 解压器的输出处理回调函数
 * 
 * 函数作用：
 * 这个函数作为解压器的flush回调函数，负责处理解压后的数据
 * 它将解压后的数据写入到initramfs文件系统中，同时处理数据流中的特殊字符
 * 
 * 参数：
 * - bufv: 包含解压后数据的缓冲区（void*类型，需要转换为char*）
 * - len: 缓冲区中数据的长度
 * 
 * 返回值：
 * - 成功时返回原始长度（origLen）
 * - 失败时返回-1
 */
static long __init flush_buffer(void *bufv, unsigned long len)
{
	char *buf = bufv;           // 将void*转换为char*指针
	long written;               // 单次写入的字节数
	long origLen = len;         // 保存原始长度，用于返回值
	
	// 如果已经有错误信息，直接返回失败
	if (message)
		return -1;
	
	// 循环写入数据，直到全部写完或遇到错误
	while ((written = write_buffer(buf, len)) < len && !message) {
		char c = buf[written];  // 获取当前未能写入的字符
		
		if (c == '0') {
			// 遇到字符'0'：这是填充字符，表示一个归档的结束
			buf += written;     // 移动缓冲区指针，跳过已写入的数据
			len -= written;     // 减少剩余长度
			state = Start;      // 设置状态为Start，准备处理下一个归档
		} else if (c == 0) {
			// 遇到null字符(0)：表示当前归档处理完毕
			buf += written;     // 移动缓冲区指针
			len -= written;     // 减少剩余长度  
			state = Reset;      // 设置状态为Reset，表示需要重置解压器状态
		} else
			// 遇到其他字符：这是无效数据，压缩归档中不应该出现
			error("junk within compressed archive");
	}
	
	// 返回原始长度，表示所有数据都已处理
	return origLen;
}

```
以上是解压的部分。而将解压后的数据写入到原先的`rootfs`模型中是在解压函数中实现的：
```c
获取：decompress = decompress_method(buf, len, &compress_name);

假设为xz解压(linux-6.6.29\lib\decompress_unxz.c)：
STATIC int INIT unxz(unsigned char *in, long in_size,
		     long (*fill)(void *dest, unsigned long size),
		     long (*flush)(void *src, unsigned long size),
		     unsigned char *out, long *in_used,
			     void (*error)(char *x))
中会调用flush进行构建。

在flush_buffer(传入到解压函数的flush)对相应的节点进行相应的action:
static __initdata int (*actions[])(void) = {
	[Start]		= do_start,
	[Collect]	= do_collect,
	[GotHeader]	= do_header,
	[SkipIt]	= do_skip,
	[GotName]	= do_name,
	[CopyFile]	= do_copy,
	[GotSymlink]	= do_symlink,
	[Reset]		= do_reset,
};
```
最终这些`action`会一点点构建出一个完整的文件系统。

```c
// 在内核启动早期（在unpack_to_rootfs之前）
void __init init_rootfs(void)
{
    // 注册rootfs文件系统类型
    register_filesystem(&rootfs_fs_type);
    
    // 挂载rootfs作为根文件系统
    // 这会设置current->fs->root和current->fs->pwd
    init_mount_tree();
}
```
关键的全局数据：
```c
// 内核中的全局变量，指向当前进程的文件系统信息
struct task_struct *current;
current->fs->root;  // 根目录dentry
current->fs->pwd;   // 当前工作目录

// 全局的根文件系统挂载点
struct vfsmount *rootfs_mount;
```
数据流向和集成过程:
```txt
unpack_to_rootfs()
    ↓
actions[state]() 状态机函数
    ↓
sys_mkdir("/some/path") 等系统调用
    ↓
VFS层 (do_mkdir, vfs_mkdir等)
    ↓
rootfs文件系统操作函数
    ↓
更新rootfs的内存数据结构:
  - dentry (目录项)
  - inode (索引节点) 
  - file (文件对象)
```


# 探究压缩
这里测试了集中压缩方式下，`Image`的大小：

|                压缩配置                |  使用命令  | Image大小(MB) |
| :--------------------------------: | :----: | :---------: |
| CONFIG_INITRAMFS_COMPRESSION_NONE  |   cp   |     82      |
|            无`initramfs`            |   /    |     41      |
| CONFIG_INITRAMFS_COMPRESSION_GZIP  |  gzip  |     54      |
| CONFIG_INITRAMFS_COMPRESSION_BZIP2 | bzip2  |     53      |
| CONFIG_INITRAMFS_COMPRESSION_LZMA  |  lzma  |     45      |
|  CONFIG_INITRAMFS_COMPRESSION_XZ   | xzmisc |     50      |
|  CONFIG_INITRAMFS_COMPRESSION_LZO  |  lzo   |     55      |
|  CONFIG_INITRAMFS_COMPRESSION_LZ4  |  lz4   |     56      |
| CONFIG_INITRAMFS_COMPRESSION_ZSTD  |  zstd  |     49      |
上面的仅仅只是作为参考，实际情况与压缩内容相关，有些内容的形式可能使用某种压缩的压缩率更高，而换一种就低了，这也是有可能的。