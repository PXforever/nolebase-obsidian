---
share: "true"
---


> 很多嵌入式设备实际上并没有办法能够正常的使用`apt`下载到合适的`bpftool`，所以这里我们使用内核源码来编译该工具
> 这里是：`RK3588,kernel-6.6`

# 方法1(交叉编译)
我们进入源码目录，然后先确定好自己的交叉编译工具链的位置：
```shell
# 比如我这里交叉编译工具链位置是,注意是带上编译工具链的前缀
CROSS_COMPILE="/home/<user-name>/<rk3588 source>/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-"

make ARCH=arm64 CROSS_COMPILE=${CROSS_COMPILE} -j25 tools/bpf/bpftool
```

执行后，我们发现错误：
![[笔记/01 附件/Linux调试与分析-嵌入式交叉编译bpftool工具/file-20241115151159048.png|笔记/01 附件/Linux调试与分析-嵌入式交叉编译bpftool工具/file-20241115151159048.png]]
查找了一下，发现`编译工具链目录`没有该头文件，表示缺少库，我们使用源码进行编译`elfutils`：
```shell
# 下载源码
wget https://sourceware.org/elfutils/ftp/0.189/elfutils-0.189.tar.bz2

# 解压
tar -xvf elfutils-0.189.tar.bz2

cd elfutils-0.189

# 配置
./condfig
```

# 方法2(本机编译)
我们可以使用`make srcdeb-pkg`获得源码包，然后上传到`RK3588`，然后在上面进行编译。
```shell
# 安装工具
sudo apt install dpkg-dev
```
```shell
# 进入内核代码
cd kernel_source

# 设置交叉编译工具位置
CROSS_COMPILE="/home/<user name>/rk3588/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-"

# 编译
make ARCH=arm64 CROSS_COMPILE=${CROSS_COMPILE} -j20 srcdeb-pkg
```
![[笔记/01 附件/Linux调试与分析-嵌入式交叉编译bpftool工具/file-20241116155021111.png|笔记/01 附件/Linux调试与分析-嵌入式交叉编译bpftool工具/file-20241116155021111.png]]
接着，我们上传源码包到机器上，然后解压：
```shell
sudo tar -xvf linux.tar.gz -C /usr/src/
cd /usr/src/
mv linux kernel_src-6.6.29
```
接着我们来编译`bpftool`:
```shell
# 安装如下软件
sudo apt update
sudo apt install -y gcc flex bison libelf-dev
sudo apt install -y clang llvm libcap-dev binutils-dev     #接下来开发会使用到
```
```shell
# 进入源码
cd kernel_src-6.6.29

# 使用原系统的config
zcat /proc/config.gz > .config

# 编译
make tools/bpf/bpftool

# 安装
cd kernel_src-6.6.29/tools/bpf/bpftool
make install
```

好了，我们来测试下：
```shell
bpftool prog list
```
![[笔记/01 附件/Linux调试与分析-嵌入式交叉编译bpftool工具/file-20241116163837028.png|笔记/01 附件/Linux调试与分析-嵌入式交叉编译bpftool工具/file-20241116163837028.png]]
