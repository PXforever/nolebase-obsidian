---
share: "true"
---

# QEMU执行
以`ARM64`为例
```shell
make qemu_arm64_defconfig
make
```
执行：
```shell
qemu-system-aarch64 -machine virt -nographic -cpu cortex-a57 -bios u-boot.bin
```




# GDB调试
启动服务端：
```shell
# 执行执行
qemu-system-aarch64 -machine virt -nographic -cpu cortex-a57 -bios u-boot.bin -gdb tcp::3333

# 阻塞等待远程GDB链接
qemu-system-aarch64 -machine virt -nographic -cpu cortex-a57 -bios u-boot.bin -gdb tcp::3333 -S
```

GDB调试连接：
```shell
gdb-multiarch u-boot
# 进入gdb
> target extended-remote :3333
```

