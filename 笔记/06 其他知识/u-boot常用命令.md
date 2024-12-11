---
share: "true"
---
# U-boot命令使用

## mmc

```shell
mmc list
mmc dev 0                           #设置当前设备为mmc 0
mmcinfo
mmc rescan 扫描 MMC 设备。
mmc part 列出 MMC 设备的分区。
mmc erase blk# cnt

mmc erase 0x600 0x4000   			#可以不加0x,这里按照block来计算的，一般一个block=512B
mmc write 0x10100000 0x600 0x4000
mmc read 0x10100000 0x600 0x4000

如果要将 EMMC 的分区 2 设置为当前 MMC 设备，可以使用如下命令：
mmc dev 1 2
```

## 网络

```shell
setenv ipaddr 192.168.0.4
setenv ethaddr b8:ae:1d:01:00:00
setenv gatewayip 192.168.0.1
setenv netmask 255.255.255.0
setenv serverip 192.168.1.253
```

## 环境变量

```shell
 printenv 		 #打印所有环境变量
 setenv test 100 #设置某个环境变量
 saveenv
```

