---
share: "true"
---

# V4L2概述

> 。。。。

## 章节安排

1. 先使用`dt3155驱动进行基本入门`
2. 开始编写简单的`v4l2驱动`
3. 实现`v4l2`的查询与设置参数功能
4. 实现`buffer`接口，从而显示图片
5. 使用`DMA`传输
6. 其他....
# v4l2实例描述

> 在此，我们会通过编写一个简单的`v4l2`驱动设备来熟悉`v4l2`框架，该设备包含如下参数：
> 支持`YUV422,RGB888`，分辨率为`640*480 30fps`。
> 我们将上面的驱动编写分为：
> 1. 设备注册
> 2. 实现`ioctl`中的：`vidioc_querycap,vidioc_g_fmt_vid_cap,vidioc_s_fmt_vid_cap`
> 3. 实现`ioctl`中的：`vidioc_reqbufs,vidioc_querybuf,vidioc_qbuf,vidioc_dqbuf,vidioc_streamon,vidioc_streamoff`
> 4. 实现图像内容填充(通过虚拟的向`v4l2`框架循环写入`4`张连续图，实现设备的图像采集)。
> 5. 使用`DMA`方式实现。
> 6. 实现`Export-buffer`接口
> 因为内核只提供机制，而不编写策略，所以在开发驱动的同时我们还需要编写一个简单的上层应用来进行验证，所以该应用程序会穿插实现，当编写了新的驱动，则改动相应应用程序进行测试。

==注：为了脱离硬件开发环境，这里选用目标机器是`ubuntu`虚拟机，这样减少外部硬件依赖，导致工作不连续的情况。==

# 构建环境
> 安装`gcc,make,g++等编译工具`
>
> 安装`linux-herders，linux-source`等内核源码构建工具

## ubuntu的环境
> 使用`ubuntu`开发时，可能未开启`v4l2`，所以在`insmod`时会出现：
```shell
[12764.176366] v4l2_register: Unknown symbol v4l2_fh_open (err -2)
[12764.176373] v4l2_register: Unknown symbol video_unregister_device (err -2)
[12764.176376] v4l2_register: Unknown symbol v4l2_device_register (err -2)
[12764.176381] v4l2_register: Unknown symbol __video_register_device (err -2)
[12764.176384] v4l2_register: Unknown symbol v4l2_device_unregister (err -2)
[12764.176386] v4l2_register: Unknown symbol video_device_release_empty (err -2)
```
查看符号表：
```shell
cat /proc/kallsyms |grep v4l2
# 没有任何结果
```
再使用：
```shell
lsmod|grep v4l2
modprobe v4l2
# 没有查找到任何相关信息
```
**那么可以判断该系统内核没有开启`v4l2`。**
我们在`root`下使用如下命令：
```shell
# 安装相应工具

# 配置V4L2
make menuconfig

# 进入源码进行编译
cd /usr/src/linux-source-`uname -r`
make -j$(nproc)
make modules

# 安装
make modules_install
make install

# 更新Grub
update-grub

# 重启后，加载v4l2模块

```
接着安装`v4l2`工具
```shell
#
v4l2-utilit
```

# v4l2设备的简单注册(`v4l2_reg`)
> `v4l2`的注册只需要使用
> + `v4l2_device_register`
> + `video_register_device`
>
> 具体代码为：`driver/v4l2_reg/v4l2_register.c`

这样在上层的目录`/dev`下可以看到`videox`设备，具体编号看自己的设备。
在这里，我们再编写一个`v4l2`的上层应用来测试它，具体代码在：`app/v4l2_demo.c`。
编译成功后，如下操作：
```shell
sudo insmod v4l2_register.ko
./v4l2_demo /dev/video4
```
执行结果为：
```shell
[   63.899540] running my_v4l2_fh_open
[   63.899567] runing my_v4l2_fop_release
```

# v4l2的参数查询与设置(`v4l2_ioctl`)
> 在这里，我们将实现`struct v4l2_ioctl_ops`中的接口函数：
> + `vidioc_querycap`：查询视频设备的能力和功能。(驱动名，总线信息，驱动版本，支持的功能等)
> + `vidioc_enum_fmt_vid_cap`：遍历当前设备所支持的图像格式。（枚举设置所支持的格式）
> + `vidioc_g_fmt_vid_cap`：获取当前设备设置的图像格式。
> + `vidioc_s_fmt_vid_cap`：设置当前的图像格式。

在上一节代码中增加：
```c
static const struct v4l2_ioctl_ops my_v4l2_ioctl_ops = {
    ....
}
static const struct v4l2_file_operations my_v4l2_fops = {
	.owner = THIS_MODULE,
	.open = my_v4l2_fh_open,
	.release = my_v4l2_fop_release,
	.unlocked_ioctl = video_ioctl2,//这里换成这个。删除原来的函数，当然也可以自己实现，但是利用本来已经有的接口方便学习，可自行源码该部分跳转代码
	.read = my_v4l2_fop_read,
	.mmap = my_v4l2_fop_mmap,
	.poll = my_v4l2_fop_poll
};
static const struct video_device my_vdev = {
    .ioctl_ops = &my_v4l2_ioctl_ops,
};
```

## vidioc_querycap
> 该接口主要是由`VIDIOC_QUERYCAP`命令查询，具体命令列表可看：`kernel/include/uapi/linux/videodev2.h`
>
> `VIDIOC_QUERYCAP`主要是查询设备的基本信息。

接着实现`my_v4l2_ioctl_ops`中的函数：
```c
static const struct v4l2_ioctl_ops my_v4l2_ioctl_ops = {
    .vidioc_querycap = my_v4l2_querycap,
};
```
我们来看看`my_v4l2_querycap`的第三个参数`struct v4l2_capability`
```c
struct v4l2_capability {
	__u8	driver[16]; //驱动名称
	__u8	card[32];	//设备名称
	__u8	bus_info[32];	//设备连接到的总线信息
	__u32   version;		//驱动版本，一共三个字段，每个字段8字节，如：1.0.110，那么在参数编写为：65646=0x1006E
	__u32	capabilities;	//设备支持的功能，各个位意义可以参看头文件
	__u32	device_caps;	//设备特定的功能
	__u32	reserved[3];	//保留字段
};
```
这意味着，我们在查询设备时，需要返回上面结构体信息给应用层，这里我们实现：
```c
driver = "MY V4L2 Drv"
card = "my virtual card"
version = 1010
capabilities = V4L2_CAP_VIDEO_CAPTURE|V4L2_CAP_STREAMING  
```
使用载入模块后，出现崩溃：
```shell
[   44.191038] Call trace:
[   44.191256]  do_raw_spin_lock+0x20/0x10c
[   44.191603]  _raw_spin_lock_irqsave+0x54/0x64
[   44.191984]  ida_alloc_range+0x8c/0x400
[   44.192322]  media_device_register_entity+0x70/0x1ec
[   44.192757]  __video_register_device+0xe50/0x17a0
[   44.193171]  v4l2_reg_init+0x120/0x1000 [v4l2_ioctl]
```
检查发现需要实例化一个`互斥锁`：
```c
v4l2_demo->vdev.lock = &v4l2_demo->mux;//具体看实际的代码
```
继续载入模块，这次仍然出错：
```shell
[  476.869010] ------------[ cut here ]------------
[  476.869023] WARNING: CPU: 2 PID: 4400 at drivers/media/v4l2-core/v4l2-ioctl.c:1113 v4l_querycap+0xc8/0xd4
```
我们通过使用`cat call_trace.log |./decode_stacktrace.sh /usr/src/linux-source-5.10/vmlinux`(具体参考`call trace分析`)：
```shell
[  476.869010] ------------[ cut here ]------------
[  476.869023] WARNING: CPU: 2 PID: 4400 at drivers/media/v4l2-core/v4l2-ioctl.c:1113 v4l_querycap (/usr/src/linux-source-5.10/drivers/media/v4l2-core/v4l2-ioctl.c:1102 (discriminator 1)) 
[  476.869027] Modules linked in: v4l2_ioctl(O) xt_conntrack nft_chain_nat xt_MASQUERADE nf_nat nf_conntrack_netlink nf_conntrack nf_defrag_ipv6 nf_defrag_ipv4 nft_counter xt_addrtype nft_compat nf_tables nfnetlink bcmdhd dhd_static_buf ahci_platform ProCapture(O) libahci_platform libahci libata scsi_mod ip_tables x_tables
......................................
[  476.869215] Call trace:
[  476.869221] v4l_querycap (/usr/src/linux-source-5.10/drivers/media/v4l2-core/v4l2-ioctl.c:1102 (discriminator 1)) 
[  476.869227] __video_do_ioctl (/usr/src/linux-source-5.10/drivers/media/v4l2-core/v4l2-ioctl.c:3013) 
[  476.869233] video_usercopy (/usr/src/linux-source-5.10/drivers/media/v4l2-core/v4l2-ioctl.c:3324) 
[  476.869239] video_ioctl2 (/usr/src/linux-source-5.10/drivers/media/v4l2-core/v4l2-ioctl.c:3362) 
[  476.869244] v4l2_ioctl (/usr/src/linux-source-5.10/drivers/media/v4l2-core/v4l2-dev.c:364) 
[  476.869251] __arm64_sys_ioctl (/usr/src/linux-source-5.10/fs/ioctl.c:49 /usr/src/linux-source-5.10/fs/ioctl.c:753 /usr/src/linux-source-5.10/fs/ioctl.c:739 /usr/src/linux-source-5.10/fs/ioctl.c:739) 
[  476.869259] el0_svc_common.constprop.0 (/usr/src/linux-source-5.10/./include/asm-generic/bitops/non-atomic.h:106 /usr/src/linux-source-5.10/./include/linux/thread_info.h:97 /usr/src/linux-source-5.10/./arch/arm64/include/asm/compat.h:188 /usr/src/linux-source-5.10/./arch/arm64/include/asm/syscall.h:58 /usr/src/linux-source-5.10/arch/arm64/kernel/syscall.c:53 /usr/src/linux-source-5.10/arch/arm64/kernel/syscall.c:155) 
[  476.869264] do_el0_svc (/usr/src/linux-source-5.10/arch/arm64/kernel/syscall.c:195) 
[  476.869271] el0_svc (/usr/src/linux-source-5.10/arch/arm64/kernel/entry-common.c:366) 
[  476.869277] el0_sync_handler (/usr/src/linux-source-5.10/arch/arm64/kernel/entry-common.c:382) 
[  476.869282] el0_sync (/usr/src/linux-source-5.10/arch/arm64/kernel/entry.S:690) 
```
可以看到`PC`指针停在了`v4l2-ioctl.c:1102`位置，我们看看源码：
```c
1080 static int v4l_querycap(const struct v4l2_ioctl_ops *ops,
1081                                 struct file *file, void *fh, void *arg)
1082 {
..............
1102         WARN_ON((cap->capabilities &
1103                  (vfd->device_caps | V4L2_CAP_DEVICE_CAPS)) !=
1104                 (vfd->device_caps | V4L2_CAP_DEVICE_CAPS));
1105         cap->capabilities |= V4L2_CAP_EXT_PIX_FORMAT;
1106         cap->device_caps |= V4L2_CAP_EXT_PIX_FORMAT;
..............
1109 }
```
这里为了检查`cap->capabilities`是`vfd->device_caps | V4L2_CAP_DEVICE_CAPS`的超集，做如下修改：
```c
//原来device_caps 有3个属性，而my_v4l2_querycap只有2个，只需要
static const struct video_device my_vdev = {
        ........
    .device_caps = V4L2_CAP_VIDEO_CAPTURE | V4L2_CAP_STREAMING
        ........        
}
static int my_v4l2_querycap {
    .........
#if (LINUX_VERSION_CODE < KERNEL_VERSION(3,17,0))
    cap->capabilities = V4L2_CAP_VIDEO_CAPTURE | V4L2_CAP_STREAMING;
#else
    /* V4L2_CAP_DEVICE_CAPS must be set since linux kernle 3.17 */
    cap->device_caps = V4L2_CAP_VIDEO_CAPTURE | V4L2_CAP_STREAMING;
    cap->capabilities = cap->device_caps | V4L2_CAP_DEVICE_CAPS;
#endif
}
```
再次执行加载模块，正常运行。
在命令行键入：
```shell
# 使用 VIDIOC_QUERYCAP
v4l2-ctl  -d /dev/video4 -D

# 结果如下：
Driver Info:
        Driver name      : MY V4L2 Drv
        Card type        : my virtual card
        Bus info         : 
        Driver version   : 0.3.242
        Capabilities     : 0x84200001
                Video Capture
                Streaming
                Extended Pix Format
                Device Capabilities
        Device Caps      : 0x04200001
                Video Capture
                Streaming
                Extended Pix Format
```

## vidioc_enum_fmt_vid_cap
> 该接口主要是实现一个图像格式的实例

我们使用自己创建的结构体：
```c
struct my_fmt {
    char  *name;
    u32    fourcc;          /* v4l2 format id 这是关键参数 */
    int    depth;
};
```
分别加入`RGB888`与`YUV422`两个参数，再添加相应代码（查看源码）。
我们执行代码`v4l2-ctl --list-formats -d /dev/video4`发现并没有跑到自定义的`vidioc_enum_fmt_vid_cap`，查找原因发现停留在：
```c
//*kernel/driver/media/v4l2-core/v4l2_ioctl.c -> static int v4l_enum_fmt
int ret = check_fmt(file, p->type);
if (ret)
    return ret;
ret = -EINVAL;

static int check_fmt(struct file *file, enum v4l2_buf_type type)
{
    。。。。
	case V4L2_BUF_TYPE_VIDEO_CAPTURE:
		if ((is_vid || is_tch) && is_rx &&
		    (ops->vidioc_g_fmt_vid_cap || ops->vidioc_g_fmt_vid_cap_mplane)) //发现vidioc_g_fmt_vid_cap没实现导致的
			return 0;
		break;
}        
```
嗯...，先暂时写一个空壳`vidioc_g_fmt_vid_cap`。
`insmod`后崩溃，我们将崩溃代码导入至驱动程序中，我们复制下崩溃部分写到`call_trace.log`，并且找到`vmlinux`(可以参考call trace文档)：
```shell
/usr/src/linux-headers-5.10.xxx/scripts/decode_stacktrace.sh ~/vmlinux ~/v4l2_test/driver/v4l2_ioctl < call_trace.log 
# ./decode_stacktrace.sh vmlinux_path/  module_path/ < call_tarce(崩溃记录)
```
得到：
```shell
.....
[ 2014.673900] Call trace:
[ 2014.673902] my_v4l2_enum_fmt_vid_cap (/home/firefly/v4l2_test/driver/v4l2_ioctl/v4l2_ioctl.c:119) v4l2_ioctl
[ 2014.673906] v4l2_open (/home/px/rk3588/kernel/drivers/media/v4l2-core/v4l2-dev.c:428) 
[ 2014.673910] chrdev_open (/home/px/rk3588/kernel/fs/char_dev.c:414) 
[ 2014.673912] do_dentry_open (/home/px/rk3588/kernel/fs/open.c:820) 
[ 2014.673914] vfs_open (/home/px/rk3588/kernel/fs/open.c:943) 
[ 2014.673916] path_openat (/home/px/rk3588/kernel/fs/namei.c:3397 /home/px/rk3588/kernel/fs/namei.c:3515) 
[ 2014.673918] do_filp_open (/home/px/rk3588/kernel/fs/namei.c:3542) 
[ 2014.673919] do_sys_openat2 (/home/px/rk3588/kernel/fs/open.c:1217) 
[ 2014.673920] __arm64_sys_openat (/home/px/rk3588/kernel/fs/open.c:1244) 
[ 2014.673923] el0_svc_common.constprop.0 (/home/px/rk3588/kernel/./arch/arm64/include/asm/current.h:19 /home/px/rk3588/kernel/arch/arm64/kernel/syscall.c:53 /home/px/rk3588/kernel/arch/arm64/kernel/syscall.c:155) 
[ 2014.673925] do_el0_svc (/home/px/rk3588/kernel/arch/arm64/kernel/syscall.c:195) 
[ 2014.673928] el0_svc (/home/px/rk3588/kernel/arch/arm64/kernel/entry-common.c:358) 
[ 2014.673930] el0_sync_handler (/home/px/rk3588/kernel/arch/arm64/kernel/entry-common.c:374) 
[ 2014.673931] el0_sync (/home/px/rk3588/kernel/arch/arm64/kernel/entry.S:789) 
```
问题在`119`行，
```c
static int my_v4l2_enum_fmt_vid_cap(struct file *file, void  *priv,
        struct v4l2_fmtdesc *f)
{
    struct my_fmt *fmt;
    struct my_v4l2_demo *vd = video_drvdata(file);
    fmt = vd->fmts;
    printk("runing %s\n", __func__);

    if (f->index >= ARRAY_SIZE(fmts)) //这里
        return -EINVAL;

    fmt += f->index;

    strscpy(f->description, fmt->name, sizeof(f->description));
    f->pixelformat = fmt->fourcc;
    return 0;
}
```
但是检查并没有问题，重启再次执行，它又正常运行，故暂时不理会。
在命令行输入：
```shell
# v4l2-ctl --help-all |grep VIDIOC_ENUM_FMT 可以看下用哪条命令测试
v4l2-ctl --list-formats -d /dev/video4

# 结果如下：
ioctl: VIDIOC_ENUM_FMT
        Type: Video Capture

        [0]: 'RGB4' (32-bit A/XRGB 8-8-8-8)
        [1]: 'YUYV' (YUYV 4:2:2)
```

## vidioc_g_fmt_vid_cap
> 该接口主要是获取当前设置的图像定义，也就是当前流将会以什么图像格式传输数据。
>
> 该函数主要做的工作是：填充上层的结构体`struct v4l2_format`

```c
struct v4l2_format {
	__u32	 type;
	union {
		struct v4l2_pix_format		pix;     /* V4L2_BUF_TYPE_VIDEO_CAPTURE */
		struct v4l2_pix_format_mplane	pix_mp;  /* V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE */
		struct v4l2_window		win;     /* V4L2_BUF_TYPE_VIDEO_OVERLAY */
		struct v4l2_vbi_format		vbi;     /* V4L2_BUF_TYPE_VBI_CAPTURE */
		struct v4l2_sliced_vbi_format	sliced;  /* V4L2_BUF_TYPE_SLICED_VBI_CAPTURE */
		struct v4l2_sdr_format		sdr;     /* V4L2_BUF_TYPE_SDR_CAPTURE */
		struct v4l2_meta_format		meta;    /* V4L2_BUF_TYPE_META_CAPTURE */
		__u8	raw_data[200];                   /* user-defined */
	} fmt;
};
```
我们的设备设置的属性是`V4L2_BUF_TYPE_VIDEO_CAPTURE`，所以填充时使用`struct v4l2_pix_format pix`。
首先，我们需要在`struct my_v4l2_demo`中加入自己的结构体：
```c
struct my_stream_fmt{
    unsigned width, height;
    unsigned pix_format;
    unsigned field;
    unsigned bytesperline;
    unsigned colorspace;
}
struct my_v4l2_demo{ 
    struct v4l2_device v4l2; 
    struct video_device vdev; 
    struct mutex mux; 
    struct my_fmt *fmts; 
    struct my_stream_fmt my_s_fmt;
};
```
再初始化这些参数，具体查看源码。
我们使用命令来查看:
```shell
# v4l2-ctl --help-all |grep VIDIOC_G_FMT 可以看下用哪条命令测试
v4l2-ctl --get-fmt-video -d /dev/video4
v4l2-ctl -V -d /dev/video4

# 结果
Format Video Capture:
        Width/Height      : 640/480
        Pixel Format      : 'RGB4' (32-bit A/XRGB 8-8-8-8)
        Field             : None
        Bytes per Line    : 640
        Size Image        : 1228800
        Colorspace        : sRGB
        Transfer Function : Default (maps to sRGB)
        YCbCr/HSV Encoding: Default (maps to ITU-R 601)
        Quantization      : Default (maps to Full Range)
        Flags             : 
```

## vidioc_s_fmt_vid_cap
> 这里是设置参数，我们只有两个图像参数可选：`RGB`，`YUV`

所以这部分代码实际很简单，具体看源码。
我们使用命令来验证：
```shell
# v4l2-ctl --help-all |grep VIDIOC_S_FMT 可以看下用哪条命令测试
firefly@firefly:~/v4l2_test/driver$ v4l2-ctl --set-fmt-video width=648,height=480,pixelformat=YUYV -d /dev/video4

# 结果
Format Video Capture:
        Width/Height      : 640/480
        Pixel Format      : 'YUYV' (YUYV 4:2:2)
        Field             : None
        Bytes per Line    : 640
        Size Image        : 614400
        Colorspace        : sRGB
        Transfer Function : Default (maps to sRGB)
        YCbCr/HSV Encoding: Default (maps to ITU-R 601)
        Quantization      : Default (maps to Limited Range)
        Flags             : 
```
## 问题
在`insmod`时总是偶发性的崩溃：
```shell
[  209.578739] Unable to handle kernel paging request at virtual address 004d003200e000d1
。。。。
[  209.757291] Call trace:
[  209.759738]  do_raw_spin_lock+0x20/0x12c
[  209.763661]  _raw_spin_lock_irqsave+0x44/0x60
[  209.768017]  ida_alloc_range+0x8c/0x400
[  209.771849]  media_device_register_entity+0x70/0x1ec
[  209.776808]  __video_register_device+0x658/0x740
[  209.781422]  v4l2_reg_init+0x17c/0x1000 [v4l2_ioctl]
[  209.786382]  do_one_initcall+0x48/0x270
[  209.790214]  do_init_module+0x4c/0x254
[  209.793963]  load_module+0x2370/0x2b30
```
检查发现是在分配空间的时候没有清除内存，导致在函数：
```c
 video_register_device
	__video_register_device
		video_register_media_controller

static int video_register_media_controller(struct video_device *vdev)
{
		if (!vdev->v4l2_dev->mdev || vdev->vfl_dir == VFL_DIR_M2M) //这里
		return 0;
}		
```
因为程序并没有实现`struct media_device *mdev`，但程序走到这里并没有返回，所以我们在驱动程序：
```c
    v4l2_demo = kmalloc( sizeof(struct my_v4l2_demo), GFP_KERNEL);
//换为
	v4l2_demo = kzalloc( sizeof(struct my_v4l2_demo), GFP_KERNEL);
```

# v4l2的数据流接口(`v4l2_stream`)
> 对于数据流这块，实际上`v4l2`有现成的接口可以使用：`videobuffer2`，简称`vb2`，但这里为了方便理解，我们不使用该接口，自己单独编写代码.

`v4l2`的内存映射方式有如下几种：
```c
//videodev2.h
enum v4l2_memory {
	V4L2_MEMORY_MMAP             = 1, //内存映射，内存在内核中申请，然后将用户地址映射到内核中
	V4L2_MEMORY_USERPTR          = 2, //用户指针，内存由用户申请，将其传递给内核做填充
	V4L2_MEMORY_OVERLAY          = 3, //用于视频覆盖操作，允许视频直接显示在帧缓冲区上，而不需要用户空间应用的进行处理
	V4L2_MEMORY_DMABUF           = 4, //DMA_BUF，这是一种使用DMA的传输。
};
```
对于初学，应该是越简单越好，循序渐进，所以我们选择用户指针来进行编写，然后是内核映射。
再编写驱动前，我们需要知道应用层是如何申请`buffer`和获取图像的，这里有一个简单的例程：
```c
//v4l2_demo_usrptr.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <linux/videodev2.h>
#include <errno.h>

int main(int argc, char **argv)
{
    int fd;
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <device>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    char *dev_name = argv[1];

    // 打开视频设备
    fd = open(dev_name, O_RDWR);
    if (fd == -1) {
        perror("Opening video device");
        exit(EXIT_FAILURE);
    }
	
	//设置图像格式
	struct v4l2_format fmt;
	memset( &fmt, 0, sizeof(fmt));
	fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_RGB32;//只这只这个参数，其它参数因为都是唯一的，就不设置
	if( ioctl( fd, VIDIOC_S_FMT, &fmt) < 0)
	{
		perror("Failed to set video format\n");
		close( fd);
		return -1;
	}
	
	//申请buffer
	struct v4l2_requestbuffers req;
	memset( &req, 0, sizeof(req));
	req.count = 1;
	req.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
	req.memory = V4L2_MEMORY_USERPTR;
	if( ioctl( fd, VIDIOC_REQBUFS, &req) < 0)
	{
		perror("Failed to request buffer\n");
		close( fd);
		return -1;
	}
	
	//用户层分配内存
	void *buffer = malloc( 648*480 * 3); //长*宽*字节数(RGB32)
	if( !buffer)
	{
		perror("Failed to alloc user buffer\n");
		close( fd);
		return -1;
	}
	
	//入队
	struct v4l2_buffer buf;
	buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
	buf.memory = V4L2_MEMORY_USERPTR;
	buf.index = 0;
	buf.m.userptr = (unsigned long)buffer;
	buf.length = 648*480*3;
	if( ioctl( fd, VIDIOC_QBUF, &buf) < 0)
	{
		perror("Failed to queue buffer\n");
		free(buffer);
		close( fd);
		return -1;
	}
	
	//开启数据流
	enum v4l2_buf_type type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
	if( ioctl( fd, VIDIOC_STREAMON, &type) < 0)
	{
		perror("Failed to queue buffer\n");
		free(buffer);
		close( fd);
		return -1;
	}
	
	//出队
	if( ioctl( fd, VIDIOC_DQBUF, &buf) < 0)
	{
		perror("Failed to dequeue buffer\n");
		free(buffer);
		close( fd);
		return -1;
	}
	
	//关闭数据流
	if( ioctl( fd, VIDIOC_STREAMOFF, &type) < 0)
	{
		perror("Failed to queue buffer\n");
		free(buffer);
		close( fd);
		return -1;
	}
	
	free(buffer);
    close(fd);
    return 0;
}
```
从上面我们知道，需要实现如下基本接口：
```markdown
+ stream on
+ stream off
+ queue buffer	 入队
+ dequeeu buffer 出队
+ request buffer 申请buffer
```
因为接下来的目标是实现虚拟的向设备循环连续写入`4`张图，那么我们需要来简单设计一下程序的操作逻辑：
```markdown
1. 首先，我们需要申请`n*buffer`。
2. 接着我们需要实现一个生产-消费者的模型。
3. 生产**图像数据**的是设备`IRQ`，这里使用一个内核线程代码，在该线程中填充buffer
4. 消费**图像数据**的是用户层。
5. 而queue buffer、dequeeu buffer接口是消费者控制生产者的接口
6. 出入队的速率则控制生产者的生产速率
```
那么我们需要设计如下数据结构：
```markdown
1. 内核的buffer结构
2. 入队与线程的同步关系
3. 出队与线程的同步关系
```
这里使用**紫色**来进行填充图案。具体代码查看`v4l2_stream`目录，可以根据该代码自己仿写。
代码思路：我们需要先理解应用层的调用逻辑，它的操作逻辑是：
+ 设置颜色格式
+ 申请用户内存
+ 开启数据流
+ 入队
+ 出队
+ 保存图像到文件
+ 关闭数据流
+ 结束
那么我们需要在驱动层对应的实现以上接口，但除了以上的接口，我们还需要保证帧率问题，我们需要通过一个`生产-消费者`模型来控制`fps`，所以申请了一个内核线程，它主要的作用就是，将已经`queue buffer`的缓冲记录下来，然后通知睡眠的`vidioc_dqbuf`函数继续进行下去。
这次因为源码过多，就不赘述，具体可以查看源码进行消化。
测试时可以使用`app/v4l2_demo_usrprt`抓取图片，然后使用`7yuv`查看，也可以使用`qv4l2`在本机查看数据。
![[笔记/01 附件/Linux驱动学习-V4L2实操以及框架分析/image-20240806165946886.png|7yuv查看]]

![[笔记/01 附件/Linux驱动学习-V4L2实操以及框架分析/image-20240806165727121.png|qv4l2查看]]

# V4L2的数据填充(`v4l2_virtualOut`)
我们先寻找一张简单的`gif`图，在网络上搜索：`gif图拆分在线`，将其拆分为多张图片，然后使用`Img2Lcd`工具转换出对应的图像数组：
![[笔记/01 附件/Linux驱动学习-V4L2实操以及框架分析/image-20240807111509911.png|image-20240807111509911]]
这里使用的颜色顺序为`BGRX`，`X`表示忽略或者`alpha`通道。然后点击保存，获得一个`1228808`字节的数组，重复制作三张不一样的图，然后放入`driver/v4l2_virtualOut/image_temp.h`中。
我们在`dqbuf`中循环填充上面的三个图的数组，这样则会产生视频效果：
![[笔记/01 附件/Linux驱动学习-V4L2实操以及框架分析/image-20240807111944692.png|image-20240807111944692]]

# 使用DMA传输数据
> 暂时不考虑

# export buffer
> 暂时不考虑

# 总结
> 在这里会对`v4l2`框架以及源码进行全面的总结，该总结大都以结构图为主，阅读下面的结构图需要从一个头开始阅读，所以我们需要一步步来阅读。
>
> 同时也会对其数据结构(`struct v4l2_device ......`)进行较为清晰的讲解。
## 文件总览
我们先来看看`v4l2`核心代码的目录：
![[笔记/01 附件/Linux驱动学习-V4L2实操以及框架分析/image-20240809164202452.png|笔记/01 附件/Linux驱动学习-V4L2实操以及框架分析/image-20240809164202452.png]]



### video_ioctl2函数流程





接下来的章节：

+ `v4l2_subdev`
+ `media_entity`
