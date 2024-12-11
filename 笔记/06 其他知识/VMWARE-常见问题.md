---
share: "true"
---
1. 手动挂载共享目录
```shell
sudo mount -t fuse.vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other
```
2. 没有网络图标
```shell
systemctl stop NetworkManager &&
sudo rm /var/lib/NetworkManager/NetworkManager.state &&
systemctl start NetworkManager
```
上面两项可以合并为一个脚本`vm_my_tools.sh`:
```shell
#!/bin/bash

if [ "$1"x == ""x ];then
	echo Usage:
	echo 	$0 net
	echo 	$0 share
	exit 1
fi

case $1 in
	[nN][eE][Tt])
		systemctl stop NetworkManager &&
		sudo rm /var/lib/NetworkManager/NetworkManager.state &&
		systemctl start NetworkManager
		;;
	share)
		sudo mount -t fuse.vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other
		;;
	resize)
		mount -o remount -rw /
		mount -o remount /var/snap/firefox/common/host-hunspell
esac
```