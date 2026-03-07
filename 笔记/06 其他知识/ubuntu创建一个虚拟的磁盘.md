---
share: "true"
---

**如何创建一个虚拟的sd卡，然后我在ubuntu上进行格式化，分区等操作**

在 Ubuntu 上创建虚拟 SD 卡非常简单，主要使用 `dd` 创建镜像文件，然后用 `losetup` 将其挂载为 loop 设备。下面我来演示完整流程：

## 1. 创建虚拟 SD 卡镜像文件

```bash
# 创建一个 1GB 的虚拟 SD 卡镜像
dd if=/dev/zero of=sdcard.img bs=1M count=1024
```

## 2. 将镜像挂载为 loop 设备

```bash
# 挂载为 loop 设备
sudo losetup -fP sdcard.img

# 查看挂载到哪个设备
sudo losetup -a
```

## 3. 分区操作

```bash
# 假设挂载到了 /dev/loop0
# 使用 fdisk 或 parted 进行分区
sudo fdisk /dev/loop0
```

在 fdisk 中：
- `n` - 创建新分区
- `p` - 主分区
- `w` - 写入并退出

## 4. 格式化分区

```bash
# 格式化为 FAT32（SD 卡常用格式）
sudo mkfs.vfat -F 32 /dev/loop0p1

# 或格式化为 ext4
sudo mkfs.ext4 /dev/loop0p1
```

## 5. 挂载使用

```bash
sudo mkdir -p /mnt/sdcard
sudo mount /dev/loop0p1 /mnt/sdcard
```

## 6. 用完后卸载

```bash
sudo umount /mnt/sdcard
sudo losetup -d /dev/loop0
```

---

需要我在当前环境中实际演示这些操作吗？
我：lsblk -d -o NAME,SIZE,TYPE,MODEL
需要这个能够显示
claude:明白了！你希望虚拟 SD 卡能像真实磁盘一样在 `lsblk` 中显示，包括 MODEL 信息。

普通的 loop 设备虽然能显示，但 MODEL 会是空的。要实现完整的效果，可以使用 **scsi_debug** 内核模块，它能创建一个带 MODEL 信息的虚拟 SCSI 设备。

让我在当前环境演示一下：
[Artifacts: undefined]
当前容器环境没有完整的 `/sys` 文件系统，无法演示。不过我可以给你完整的方法，在你的真实 Ubuntu 系统上使用：

---

## 方法一：使用 scsi_debug 模块（推荐，有 MODEL 显示）

```bash
# 加载 scsi_debug 模块，创建一个 1GB 的虚拟 SD 卡
sudo modprobe scsi_debug dev_size_mb=1024 sector_size=512 \
    vendor="Generic" product="SD Card" rev="1.0"

# 查看设备
lsblk -d -o NAME,SIZE,TYPE,MODEL
```

输出效果类似：
```
NAME   SIZE TYPE MODEL
sda    500G disk VBOX HARDDISK
sdb      1G disk SD Card         <-- 这就是虚拟 SD 卡
```

然后可以对 `/dev/sdb` 进行分区和格式化：

```bash
# 分区
sudo fdisk /dev/sdb

# 格式化为 FAT32
sudo mkfs.vfat -F 32 /dev/sdb1

# 挂载
sudo mount /dev/sdb1 /mnt
```

卸载模块：
```bash
sudo modprobe -r scsi_debug
```

---

## 方法二：普通 loop 设备（简单，但 MODEL 为空）

```bash
# 创建镜像
dd if=/dev/zero of=~/sdcard.img bs=1M count=1024

# 挂载为 loop 设备
sudo losetup -fP ~/sdcard.img

# 查看
lsblk -d -o NAME,SIZE,TYPE,MODEL
```

输出：
```
NAME   SIZE TYPE MODEL
loop0    1G loop          <-- MODEL 为空，TYPE 是 loop
```

---

## 总结

| 方法 | TYPE 显示 | MODEL 显示 | 真实度 |
|------|-----------|------------|--------|
| scsi_debug | disk | ✅ 有 | 高（像真实设备）|
| losetup | loop | ❌ 无 | 中等 |

如果你需要模拟真实 SD 卡的效果（比如测试脚本），推荐使用 **scsi_debug** 方法。
