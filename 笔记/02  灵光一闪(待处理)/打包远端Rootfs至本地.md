---
share: "true"
---

某些时候，我们希望能够保存当前的`rootfs`，重新打包为一个`img`，在本机中操作难免会因为某些活动而发生问题，我们可以通过`rsync`来实现：

同步至本地(需要较长时间)
```shell
sudo rsync -avx root@192.168.0.93:/ ./remote_rootfs
```
创建一个`remote_rootfs.img`
```shell
# 空间设为11GB
dd if=/dev/zero of=remote_rootfs.img bs=1M count=10240

# 格式化
sudo mkfs.ext4 -F -L linuxroot remote_rootfs.img
```
创建一个挂载目录：
```shell
mkdir rootfs_mount_tmp
```
挂载，并且复制：
```shell
sudo mount remote_rootfs.img rootfs_mount_tmp
sudo cp -rfp remote_rootfs/* rootfs_mount_tmp
sudo umount rootfs_mount_tmp
```
检查`remote_rootfs.img`，并且缩小尺寸：
```shell
sudo e2fsck -p -f remote_rootfs.img
sudo resize2fs -M remote_rootfs.img
```