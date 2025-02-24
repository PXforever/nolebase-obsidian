---
share: "true"
---

> `initramfs`是....

在这里我们将会分为如下几个部分来探究该内容：
+ `Makefile`是如何将`initramfs`打包进`Imnage`的
+ 使用`busybox`制作`initramfs`
+ `initramfs`如何被载入到系统中，并且挂载
# initramfs 与 Makefile

# busybox制作initramfs
+ initramfs启动过程(格式，解压，内核传递)，cmdline设置，console无法启动

# 加载过程分析

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