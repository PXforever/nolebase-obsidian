---
share: "true"
---

> `boot.img`是内核的镜像，不过该镜像并不只有内核的可执行文件，还有其他的信息，比如设备树等。但是有些嵌入式系统它的内核段分区也其他名称的，这里就不讨论定制的内核镜像格式。
> 内核镜像它生成后，最终会被`uboot`使用，所以我们从标准的`uboot`开始着手去探讨内核镜像如何的生成与制作。
> 这里的讲解以`RK3588, kernel-6.6`为参考进行分析。

# `U-Boot`制作`kernel`镜像的工具
我们可以在[官网](https://ftp.denx.de/pub/u-boot/)下载最新的`uboot`源码：
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241111140547280.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241111140547280.png]]
接着解压，我们可以在目录`u-boot-2024.10/tools`中看到文件：
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241111140643422.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241111140643422.png]]
这里有`mkimage`工具源码。
所以说，**内核的镜像格式，或者组织结构，是由`uboot`决定的。**
## FIT格式
我们可以在[UBOOT DOC-Package](https://docs.u-boot.org/en/latest/develop/package/binman.html#relationship-to-mkimage)中找到如下信息：
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112114109458.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112114109458.png]]

## `mkimage`简单分析
> 略(未来有空完善)


# 示例1-RK3588的内核镜像

> `RK3588`的内核镜像(`boot.img`)支持两种方式：
> + `Android`模式
> + `FIT`模式

## Android模式
在你使用`./build.sh kernel(实际上是make ARCH=arm64 kernel_source)`时，在[arch/arm64/Makefile](https://gitlab.com/firefly-linux/kernel/-/blob/firefly/arch/arm64/Makefile?ref_type=heads)中有如下代码:
```makefile
%.img: rockchip/%.dtb kernel.img $(LOGO) $(LOGO_KERNEL)
	$(Q)$(srctree)/scripts/mkimg --dtb $*.dtb
```

[RK3588代码](https://github.com/PXforever/linux-rockchip)中有一个制作`Android 启动内核`的工具：`scripts/mkimg`：
```shell
#!/bin/bash
# SPDX-License-Identifier: GPL-2.0
# Copyright (c) 2019 Fuzhou Rockchip Electronics Co., Ltd.

......

OUT=out
ITB=${BOOT_IMG}
ITS=${OUT}/boot.its
MKIMAGE=${MKIMAGE-"mkimage"}
MKIMAGE_ARG="-E -p 0x800"

make_boot_img()
{
	RAMDISK_IMG_PATH=${objtree}/ramdisk.img
	echo "RAMDISK_IMG_PATH=${RAMDISK_IMG_PATH}, PWD=$(pwd)"
	[ -f ${RAMDISK_IMG_PATH} ] && RAMDISK_IMG=ramdisk.img && RAMDISK_ARG="--ramdisk ${RAMDISK_IMG_PATH}"

	${srctree}/scripts/mkbootimg \
		${KERNEL_IMAGE_ARG} \
		${RAMDISK_ARG} \
		--second resource.img \
		-o boot.img && \
	echo "  Image:  boot.img (with Image ${RAMDISK_IMG} resource.img) is ready";
	${srctree}/scripts/mkbootimg \
		${KERNEL_ZIMAGE_ARG} \
		${RAMDISK_ARG} \
		--second resource.img \
		-o zboot.img && \
	echo "  Image:  zboot.img (with ${ZIMAGE} ${RAMDISK_IMG} resource.img) is ready"
}

......

scripts/resource_tool ${DTB_PATH} ${LOGO} ${LOGO_KERNEL} >/dev/null
echo "  Image:  resource.img (with ${DTB} ${LOGO} ${LOGO_KERNEL}) is ready"

echo "BOOT_IMG=${BOOT_IMG}"
echo "BOOT_ITS=${BOOT_ITS}"
if [ -f "${BOOT_IMG}" ]; then
	if file -L -p -b ${BOOT_IMG} | grep -q 'Device Tree Blob' ; then
		repack_itb;
	elif [ -x ${srctree}/scripts/repack-bootimg ]; then
		repack_boot_img;
	fi
elif [ -f "${BOOT_ITS}" ]; then
	make_fit_boot_img;
elif [ -x ${srctree}/scripts/mkbootimg ]; then
	echo "running make_boot_img"
	make_boot_img;
fi
```
这里有两个指令：
```shell
${srctree}/scripts/mkbootimg \
	${KERNEL_IMAGE_ARG} \
	${RAMDISK_ARG} \
	--second resource.img \
	-o boot.img

${srctree}/scripts/mkbootimg \
	${KERNEL_ZIMAGE_ARG} \
	${RAMDISK_ARG} \
	--second resource.img \
	-o zboot.img
```
`mkbootimg`是一个`Android`的构建工具，我们可以在[Android mkbootimg.py源码](https://android.googlesource.com/platform/system/tools/mkbootimg/+/9c46d9eee9a1b469ccf46db917ac78fc9f0fdaf1/mkbootimg.py)这里获取。接着我们来看看该工具生成的`boot.img`是怎么样的结构。
我们在[bootimg.h](https://android.googlesource.com/platform/system/tools/mkbootimg/+/refs/heads/main/include/bootimg/bootimg.h)中可以看到其结构注释如下：
```c
/* When a boot header is of version 0, the structure of boot image is as
 * follows:
 *
 * +-----------------+
 * | boot header     | 1 page
 * +-----------------+
 * | kernel          | n pages
 * +-----------------+
 * | ramdisk         | m pages
 * +-----------------+
 * | second stage    | o pages
 * +-----------------+
 *
 * n = (kernel_size + page_size - 1) / page_size
 * m = (ramdisk_size + page_size - 1) / page_size
 * o = (second_size + page_size - 1) / page_size
 *
 * 0. all entities are page_size aligned in flash
 * 1. kernel and ramdisk are required (size != 0)
 * 2. second is optional (second_size == 0 -> no second)
 * 3. load each element (kernel, ramdisk, second) at
 *    the specified physical address (kernel_addr, etc)
 * 4. prepare tags at tag_addr.  kernel_args[] is
 *    appended to the kernel commandline in the tags.
 * 5. r0 = 0, r1 = MACHINE_TYPE, r2 = tags_addr
 * 6. if second_size != 0: jump to second_addr
 *    else: jump to kernel_addr
 */
```
为了验证是否正确，我们可以进行如下操作，先在`build.sh`(你的编译脚本可能不是这个)中阻止`mk-fitimage.sh`覆盖`boot.img`，这里在`mkimage`打包后，立马停止打包。这样我们就可以得到一个`Android`格式的`boot.img`。
```shell
build_kernel()
{
......
	echo "============Start building kernel============"
	echo "TARGET_KERNEL_ARCH   =$RK_KERNEL_ARCH"
	echo "TARGET_KERNEL_CONFIG =$RK_KERNEL_DEFCONFIG"
	echo "TARGET_KERNEL_DTS    =$RK_KERNEL_DTS"
	echo "TARGET_KERNEL_CONFIG_FRAGMENT =$RK_KERNEL_DEFCONFIG_FRAGMENT"
	echo "=========================================="
......
	$KMAKE $RK_KERNEL_DTS.img -j1
	exit 0 #增加这句

	ITS="$CHIP_DIR/$RK_KERNEL_FIT_ITS"
	if [ -f "$ITS" ]; then
		$COMMON_DIR/mk-fitimage.sh kernel/$RK_BOOT_IMG \
			"$ITS" $RK_KERNEL_IMG
	fi
}
```
我们将内核目录下生成的`boot.img`进行解压，在此前，我们需要安装工具`unpackbootimg`:
```shell
git clone https://github.com/osm0sis/mkbootimg
cd mkbootimg
make
sudo make install
```
解压：
```shell
mkdir bootimg_out
unpackbootimg -i ./boot.img -o ./bootimg_out
```
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241111173858483.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241111173858483.png]]
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241111174130661.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241111174130661.png]]
上面得知，`magic`为`ANDROID!`，这个是`boot header`，他有不同的版本，我们可以参考[Android boot镜像Header](https://source.android.com/docs/core/architecture/bootloader/boot-image-header#header-v4)来分析，这里我实际的应该是`v1`或者`v2`。同时我们使用`Hex Editor`查看该`boot.img`:
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241111174532411.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241111174532411.png]]
`kernel size(小端模式)`与`arch/arm64/boot/Image`的大小比较，基本一致。
接着我们偏移`1个page_size`，查看内核`Image`的内容：
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241111175237858.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241111175237858.png]]
内核可执行文件`Image`被放置到`1 page_size`位置，和`bootimg.h`中指示的一样。如果我们直接烧录该`boot.img`，启动时会有如下`log`:
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112143634418.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112143634418.png]]
这个证明`Android`镜像可以成功被加载。

## FIT模式
我们在[build.sh](https://gitlab.com/firefly-linux/device/rockchip/-/blob/firefly/common/build.sh?ref_type=heads)中找到如下代码(和上面的`build.sh`有些差异,但基本意思不变)：
```shell
cd kernel
make ARCH=$RK_ARCH $RK_KERNEL_DEFCONFIG $RK_KERNEL_DEFCONFIG_FRAGMENT
make ARCH=$RK_ARCH $RK_KERNEL_DTS.img -j$RK_JOBS
if [ -f "$TOP_DIR/device/rockchip/$RK_TARGET_PRODUCT/$RK_KERNEL_FIT_ITS" ]; then
	$COMMON_DIR/mk-fitimage.sh $TOP_DIR/kernel/$RK_BOOT_IMG \
		$TOP_DIR/device/rockchip/$RK_TARGET_PRODUCT/$RK_KERNEL_FIT_ITS \
		$TOP_DIR/kernel/ramdisk.img
fi
```
这里会检查`$TOP_DIR/device/rockchip/$RK_TARGET_PRODUCT/$RK_KERNEL_FIT_ITS`是否存在，而预设的变量`RK_KERNEL_FIT_ITS`，`RK_BOOT_IMG`等会放在软链接指示的地方：
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112144145183.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112144145183.png]]
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112144224842.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112144224842.png]]
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112144604935.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112144604935.png]]
所以上面的命令是判断是否在`RK3588_SOURCE/device/rockchip/.target_product`下是否有`boot.its`文件，如果有则使用`FIT`：
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112144726192.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112144726192.png]]
这里的`boot.its`只是提供一个模板，实际上会在`mk-fitimages.sh`中使用`sed`进行替换，比如某次替换结果如下：
```shell
/*
 * Copyright (C) 2021 Rockchip Electronics Co., Ltd
 *
 * SPDX-License-Identifier: GPL-2.0
 */

/dts-v1/;
/ {
    description = "U-Boot FIT source file for arm";

    images {
        fdt {
            data = /incbin/("/home/xxx/RK3588/kernel-6.6.29-git/arch/arm64/boot/dts/rockchip/rk3588-evb7-lp4-v10-linux.dtb");
            type = "flat_dt";
            arch = "arm64";
            compression = "none";
            load = <0xffffff00>;

            hash {
                algo = "sha256";
            };
        };

        kernel {
            data = /incbin/("/home/xxx/RK3588/kernel-6.6.29-git/arch/arm64/boot/Image");
            type = "kernel";
            arch = "arm64";
            os = "linux";
            compression = "none";
            entry = <0xffffff01>;
            load = <0xffffff01>;

            hash {
                algo = "sha256";
            };
        };

        resource {
            data = /incbin/("/home/xxx/RK3588/kernel-6.6.29-git/resource.img");
            type = "multi";
            arch = "arm64";
            compression = "none";

            hash {
                algo = "sha256";
            };
        };
    };

    configurations {
        default = "conf";

        conf {
            rollback-index = <0x00>;
            fdt = "fdt";
            kernel = "kernel";
            multi = "resource";

            signature {
                algo = "sha256,rsa2048";
                padding = "pss";
                key-name-hint = "dev";
                sign-images = "fdt", "kernel", "multi";
            };
        };
    };
};
```
打包成功后有如下打印：
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112112751672.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112112751672.png]]
我们烧录后，启动`uboot`时：
![[笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112145036275.png|笔记/01 附件/Linux构建分析-boot.img生成与制作/file-20241112145036275.png]]


## 示例2-A40i的加载方式






# 参考链接
[Android boot镜像头](https://source.android.com/docs/core/architecture/bootloader/boot-image-header#header-v4)
[Android boot分区](https://source.android.com/docs/core/architecture/partitions/vendor-boot-partitions?hl=en)
[mkimage用法](https://blog.csdn.net/zhengqijun_/article/details/71809411)
[Android源码](https://android.googlesource.com/?format=HTML)
[Android mkbootimg.py源码](https://android.googlesource.com/platform/system/tools/mkbootimg/+/9c46d9eee9a1b469ccf46db917ac78fc9f0fdaf1/mkbootimg.py)
[安卓boot镜像解析-博客园](https://www.cnblogs.com/viiv/p/15674864.html)
[安卓boot类镜像解析-CSDN](https://blog.csdn.net/qq_41720266/article/details/129028253)
http://nerveware.org/device-and-image-trees/1.html
https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18841676/U-Boot+Flattened+Device+Tree
http://www.wowotech.net/u-boot/fit_image_overview.html
https://docs.u-boot.org/en/stable/usage/fit/index.html