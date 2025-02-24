---
share: "true"
---


> `KVM`....
> 这里使用的是`RK3588`来测试：
> + kernel-6.6

# 内核开启KVM
我们需要在内核开启如下功能：
```shell
# For Kvm
CONFIG_VIRTUALIZATION=y
CONFIG_KVM=y                # 启用 KVM 支持
CONFIG_KVM_ARM=y            # 启用 ARM 平台的 KVM 支持
CONFIG_KVM_ARM_HOST=y       # 启用 ARM 主机支持
```

# 编程
我们会用到`kvm`的`ioctl`调用，具体的目录在：`arm/arm64/kvm`和`virt/kvm`，当遇到需要使用的寄存器，我们可以查看该部分代码。
在这里，会介绍三种使用示例：
+ 纯使用`kvm 相关系统调用`启动一个操作系统
+ 介绍使用`arm64-kvm-hello-world`项目
+ 介绍使用`kvm-tools`项目
## kvm 启动kernel
> 该部分主要是通过借鉴`uboot`进入内核跳转的过程，进而在`host`中启动虚拟内核。

略....
寄存器如何知道`ID`，我们可以参考`kernel/tools/testing/selftests/kvm/get-reg-list.c`中的代码。

## `arm64-kvm-hello-world`
该项目地址为：https://github.com/Lenz-K/arm64-kvm-hello-world
我们使用如下命令：
```shell
git clone https://github.com/Lenz-K/arm64-kvm-hello-world
make
```
该项目是利用`kvm`加载一段应用程序(`bare-metal-aarch64/hello_world.elf`)。
而目录`bare-metal-aarch64-qemu`是可以使用`qemu`进行测试的版本：
```shell
qemu-system-aarch64 -M virt -cpu host -enable-kvm -nographic -kernel hello_world.elf
```
我们测试纯代码的话直接执行：
```shell
./kvm_test
```
如果我们希望改变程序中的打印：
```c
// arm64-kvm-hello-world/bare-metal-aarch64/hello_world.c
void main() {
    print_uart0("Hello World, This is Px!\n");
}
```
这里增加了字符，还需要修改输出：
```c
#define MAX_VM_RUNS 20
//修改为更大的数字
#define MAX_VM_RUNS 64
```

## kvm-tools
这是一款虚拟机启动以及管理工具，他可以不借助`qemu`启动内核。
+ 下载/编译/安装
```shell
# 多个网址
git clone https://github.com/kvmtool/kvmtool
# https://git.kernel.org/pub/scm/linux/kernel/git/will/kvmtool.git也可以下载

sudo apt update 
sudo apt install -y build-essential flex bison pkg-config libssl-dev libglib2.0-dev libpixman-1-dev libfdt-dev

cd kvmtool

make

cp ./lkvm /usr/bin
cp ./vm /usr/bin
```
+ 使用，我们这里主要介绍子命令`run`的用法
```shell
# 执行Image内核，以及加载rootfs_minimal.img
lkvm run -c 2 -m 512M --kernel Image --disk rootfs_minimal.img --console serial -p "root=/dev/vda console=ttyS0 initcall_call loglevel=8"

# 执行Image内核，以及加载rootfs.img，并且输出自动生成的设备树到output.dtb
lkvm run -c 2 -m 512M --kernel Image --disk rootfs.img --console serial -p "root=/dev/vda1 console=ttyS0" --dump-dtb output.dtb

# 如果使用busybox,然后将系统(initramfs)打包进Image(kernel)
lkvm run -c 2 -m 512M --kernel Image --console serial -p "rdinit=/sbin/init console=ttyS0"
```
**`-c 2`**：
- 指定虚拟机使用的 CPU 核心数。这里设置为 2 核心。

**`-m 512M`**：
- 为虚拟机分配 512MB 的内存。

**`--kernel Image`**：
- `Image` 是内核映像的文件名。这里指定了启动时加载的内核。

**`--disk format=raw,file=rootfs_minimal.img`**：
- `--disk` 参数用于指定虚拟机的磁盘映像。
- `format=raw` 表示输入的磁盘映像是原始格式（raw format）。这意味着 `rootfs_minimal.img` 并不是一个压缩或特定格式的镜像文件（如 `.iso` 或 `.qcow2`），而是一个直接的原始磁盘镜像。
- `file=rootfs_minimal.img` 指定了用于虚拟机的根文件系统映像。

**`--console serial`**：
- 通过 `serial` 控制台连接虚拟机，意味着虚拟机的输出会通过串口输出。这通常用于调试或者无图形界面的操作。

**`-p "root=/dev/vda console=ttyS0 initcall_call loglevel=8"`**：
- `-p` 参数用于传递内核启动参数（命令行）。
- `root=/dev/vda`：指定根文件系统的位置为 `/dev/vda`，这是虚拟磁盘设备的默认名称。这样，内核会尝试从该磁盘设备加载根文件系统。
- `console=ttyS0`：设置串口控制台的输出为 `ttyS0`。这与 `--console serial` 配合使用，使虚拟机的输出通过串口进行显示。
- `initcall_call`：这是内核的启动参数，用来调试内核的初始化过程，通常用于开发和调试。
- `loglevel=8`：设置日志级别为 8，表示内核将输出更多的日志信息。更高的级别通常意味着更详细的调试信息。
这里需要注意的是：
+ `rootfs`使用的格式是：`raw`或者`qcow2`。我们可以下载[标准测试用例linux-0.2.img](http://lassauge.free.fr/qemu/release/)
+ 使用外部虚拟硬盘需要开启内核以下功能：
```shell
# 在README有说明
	CONFIG_VIRTIO=y
	CONFIG_VIRTIO_RING=y
	CONFIG_VIRTIO_PCI=y

 - For virtio-blk devices (--disk, -d):
	CONFIG_VIRTIO_BLK=y

 - For virtio-net devices ([--network, -n] virtio):
	CONFIG_VIRTIO_NET=y

 - For virtio-9p devices (--virtio-9p):
	CONFIG_NET_9P=y
	CONFIG_NET_9P_VIRTIO=y
	CONFIG_9P_FS=y

 - For virtio-balloon device (--balloon):
	CONFIG_VIRTIO_BALLOON=y

 - For virtio-console device (--console virtio):
	CONFIG_VIRTIO_CONSOLE=y

 - For virtio-rng device (--rng):
	CONFIG_HW_RANDOM_VIRTIO=y

 - For vesa device (--sdl or --vnc):
	CONFIG_FB_VESA=y
```
这里我尝试了多种办法都没有能够从外部加载`rootfs`，包括标准测试用例，最终它停留在无法识别到任何硬盘的情况下：
![[笔记/01 附件/image.png|KVM编程使用/image.png]]
这输入内核驱动问题，这里不再深入探究。
我们使用第三种办法，使用`initramfs`来进入`rootfs`，这里需要制作一个简单的文件系统，具体可以查看[[Linux小技巧-利用busybox制作最小系统|Linux小技巧-利用busybox制作最小系统]]，我们注意启动参数中的`rdinit=/sbin/init`而非`/init`。使用该方式可以进入`initramfs`，期间遇到`warning unable to open an initial console`，该问题主要是由于在构建`rootfs`未创建`console`节点导致的。