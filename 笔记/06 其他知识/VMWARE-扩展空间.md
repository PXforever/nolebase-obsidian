---
share: "true"
---
> 我们常常会遇到使用虚拟机时经常空间不足，那么我们可以这样去做。
# 分配空间方法
## 方法1：使用Gparted
1. 先在虚拟机的配置中扩展空间，`vmware`与`vitural box`不一样。
2. 在`ubuntu`系统中使用`Gparted`设置：
![[笔记/01 附件/VMWARE-扩展空间/image-20240713195816387.png|Vmware扩展空间/image-20240713195816387.png]]

## 方法2：使用`parted`
> 对于很多系统，可能不会安装`Gparted`，我们可以用这个方法。

1. 查看该在的设备名：
```shell
df -h
```
![[笔记/01 附件/VMWARE-扩展空间/image-20240713200040078.png|Vmware扩展空间/image-20240713200040078.png]]
2. `/dev/sda3`是属于`/dev/sda`下的逻辑分区，我们操作`/dev/sda`，使用`parted`
```shell
sudo parted /dev/sda
```
![[笔记/01 附件/VMWARE-扩展空间/image-20240713200138488.png|Vmware扩展空间/image-20240713200138488.png]]
3. 查看到是第一个分区，在`parted`使用该命令：
```shell
(parted) resizepart 1 100%
(parted) quit
```
4. 退出后，使用命令：
```shell
sudo resize2fs /dev/sda3
# 注意这里是sda3
```
5. 再次查看系统：
```shell
df -h
# 或者
lsblk
```

# 无法分配(只读系统)
> 当我们遇到这种问题

![[笔记/01 附件/VMWARE-扩展空间/file-20241025213501929.png|笔记/01 附件/VMWARE-扩展空间/file-20241025213501929.png]]
我们则需要重新挂载系统，先记下挂载地址：
![[笔记/01 附件/VMWARE-扩展空间/file-20241025213559531.png|笔记/01 附件/VMWARE-扩展空间/file-20241025213559531.png]]
挂载于：
```shell
/var/snap/firefox/common/host-hunspell
```
我们执行：
```shell
sudo -i  
mount -o remount -rw /  
mount -o remount -rw /var/snap/firefox/common/host-hunspell
```
点击：
![[笔记/01 附件/VMWARE-扩展空间/file-20241025213740647.png|笔记/01 附件/VMWARE-扩展空间/file-20241025213740647.png]]
![[笔记/01 附件/VMWARE-扩展空间/file-20241025213800798.png|笔记/01 附件/VMWARE-扩展空间/file-20241025213800798.png]]