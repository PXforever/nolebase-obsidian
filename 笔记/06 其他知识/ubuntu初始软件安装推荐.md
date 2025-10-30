---
share: "true"
---

# 嵌入式Linux开发环境软件清单

## 1. 虚拟机工具

| 软件包 | 作用说明 |
|--------|----------|
| open-vm-tools | VMware虚拟机的开源工具集，提供与宿主机的集成功能 |
| open-vm-tools-desktop | VMware桌面环境增强工具，支持复制粘贴、文件拖放、分辨率自适应等 |

```bash
open-vm-tools 
open-vm-tools-desktop
```

---

## 2. 基础编译工具链

| 软件包 | 作用说明 |
|--------|----------|
| build-essential | 包含gcc、g++、make等基础编译工具的元包 |
| gcc | GNU C编译器，编译C语言程序 |
| g++ | GNU C++编译器，编译C++语言程序 |
| make | 自动化编译工具，根据Makefile构建项目 |
| cmake | 跨平台的构建系统生成器，生成Makefile或其他构建文件 |

```bash
build-essential 
gcc 
g++ 
make 
cmake
```

---

## 3. 压缩/解压工具

| 软件包 | 作用说明 |
|--------|----------|
| tar | 打包和解包.tar格式文件 |
| gzip | 压缩和解压.gz格式文件 |
| bzip2 | 压缩和解压.bz2格式文件，压缩率比gzip高 |
| lbzip2 | bzip2的多线程并行版本，压缩解压更快 |
| xz-utils | 压缩和解压.xz格式文件，内核源码常用此格式 |
| p7zip-full | 7-Zip压缩工具的完整版，支持.7z、.zip等多种格式 |
| unzip | 解压.zip格式文件 |

```bash
tar 
gzip 
bzip2 
lbzip2 
xz-utils 
p7zip-full 
unzip
```

---

## 4. 调试工具

| 软件包 | 作用说明 |
|--------|----------|
| gdb | GNU调试器，调试C/C++程序，设置断点、查看变量等 |
| gdb-multiarch | 支持多架构的GDB，可调试ARM、MIPS等交叉编译程序 |
| valgrind | 内存泄漏检测工具，分析程序内存使用和性能 |
| strace | 系统调用跟踪工具，查看程序执行的系统调用 |

```bash
gdb 
gdb-multiarch 
valgrind 
strace
```

---

## 5. 编程语言

| 软件包 | 作用说明 |
|--------|----------|
| python2 | Python 2.x解释器，部分旧工具和脚本需要 |
| python3 | Python 3.x解释器，现代Python开发标准版本 |
| python3-pip | Python包管理工具，安装第三方Python库 |

```bash
python2 
python3 
python3-pip
```

---

## 6. 内核编译依赖

| 软件包 | 作用说明 |
|--------|----------|
| bc | 任意精度计算器，内核编译时用于计算 |
| bison | 语法分析器生成器，编译内核配置工具需要 |
| flex | 词法分析器生成器，与bison配合使用 |
| libelf-dev | ELF文件处理库开发文件，编译内核模块需要 |
| libncurses-dev | 终端UI库开发文件，make menuconfig需要 |
| libssl-dev | OpenSSL开发库，内核签名和加密模块需要 |
| dwarves | 包含pahole工具，生成内核BTF调试信息 |
| kmod | 内核模块加载工具集，管理.ko模块 |
| cpio | 创建和解压cpio归档，制作initramfs需要 |
| rsync | 文件同步工具，内核编译安装时使用 |

```bash
bc 
bison 
flex 
libelf-dev 
libncurses-dev 
libssl-dev 
dwarves 
kmod 
cpio 
rsync
```

---

## 7. 网络传输工具

| 软件包 | 作用说明 |
|--------|----------|
| ssh | SSH客户端，远程登录开发板或服务器 |
| curl | 命令行URL传输工具，支持HTTP、FTP等多种协议 |
| wget | 文件下载工具，支持断点续传 |
| net-tools | 网络配置工具集，包含ifconfig、route等命令 |

```bash
ssh 
curl 
wget 
net-tools
```

---

## 8. 网络管理与配置工具

| 软件包 | 作用说明 |
|--------|----------|
| network-manager | 网络连接管理器守护进程，自动管理网络连接 |
| nmcli | NetworkManager命令行工具，配置和管理网络连接 |
| nmtui | NetworkManager文本用户界面，交互式配置网络 |
| iproute2 | 现代网络配置工具集，包含ip、ss、bridge等命令，替代net-tools |
| iputils-ping | ping命令,测试网络连通性和延迟 |
| traceroute | 路由跟踪工具，显示数据包到达目的地的路径 |
| tcpdump | 网络数据包抓取和分析工具，调试网络协议 |
| wireshark | 图形化网络协议分析器，深度分析网络流量 |
| tshark | Wireshark命令行版本，适合脚本和远程分析 |
| nmap | 网络扫描和安全审计工具，扫描端口和服务 |
| netcat-openbsd | 网络调试瑞士军刀，TCP/UDP数据传输和端口监听 |
| iperf3 | 网络带宽性能测试工具，测试TCP/UDP吞吐量 |
| ethtool | 以太网卡配置和诊断工具，查看网卡状态和参数 |
| bridge-utils | Linux网桥管理工具，创建和配置虚拟网桥 |
| vlan | VLAN配置工具，创建和管理虚拟局域网 |
| dnsutils | DNS工具包，包含dig、nslookup、host等DNS查询工具 |
| telnet | Telnet客户端，远程登录和端口测试 |
| mtr | 结合ping和traceroute功能的网络诊断工具 |
| iftop | 实时网络流量监控工具，显示各连接的带宽占用 |
| iptables | Linux防火墙管理工具，配置网络过滤规则 |

```bash
network-manager 
nmcli 
nmtui 
iproute2 
iputils-ping 
traceroute 
tcpdump 
wireshark 
tshark 
nmap 
netcat-openbsd 
iperf3 
ethtool 
bridge-utils 
vlan 
dnsutils 
telnet 
mtr 
iftop 
iptables
```

---

## 9. 交叉编译工具链

| 软件包 | 作用说明 |
|--------|----------|
| gcc-arm-linux-gnueabi | ARM 32位硬浮点交叉编译器（旧ABI） |
| gcc-arm-linux-gnueabihf | ARM 32位硬浮点交叉编译器（新ABI），树莓派等使用 |
| gcc-aarch64-linux-gnu | ARM 64位交叉编译器，用于ARMv8架构 |

```bash
gcc-arm-linux-gnueabi 
gcc-arm-linux-gnueabihf 
gcc-aarch64-linux-gnu
```

---

## 10. 版本控制

| 软件包 | 作用说明 |
|--------|----------|
| git | 分布式版本控制系统，管理源代码 |
| git-lfs | Git大文件存储扩展，管理二进制大文件 |

```bash
git 
git-lfs
```

---

## 11. 嵌入式开发工具

| 软件包 | 作用说明 |
|--------|----------|
| device-tree-compiler | 设备树编译器（dtc），编译.dts为.dtb |
| u-boot-tools | U-Boot工具集，包含mkimage制作uImage内核 |
| qemu-system-arm | ARM架构QEMU虚拟机，模拟ARM开发板调试 |
| minicom | 串口终端工具，连接开发板串口调试 |
| screen | 终端复用器，也可用于串口通信 |
| picocom | 轻量级串口通信工具 |
| nfs-kernel-server | NFS服务器，通过网络挂载根文件系统 |
| tftp-hpa | TFTP客户端，通过网络下载内核和文件系统 |
| tftpd-hpa | TFTP服务器，为开发板提供网络启动文件 |

```bash
device-tree-compiler 
u-boot-tools 
qemu-system-arm 
minicom 
screen 
picocom 
nfs-kernel-server 
tftp-hpa 
tftpd-hpa
```

---

## 12. 文本编辑器

| 软件包 | 作用说明 |
|--------|----------|
| vim | 强大的命令行文本编辑器 |
| nano | 简单易用的命令行文本编辑器 |

```bash
vim 
nano
```

---

## 13. 其他实用工具

| 软件包 | 作用说明 |
|--------|----------|
| tree | 树状显示目录结构 |
| htop | 交互式进程查看器，比top更友好 |
| tmux | 终端复用器，支持分屏和会话保持 |
| dos2unix | 转换DOS/Windows文本文件为Unix格式 |
| cscope | C代码浏览工具，查找函数定义和调用 |
| ctags | 代码标签生成器，vim代码跳转需要 |
| gpart | 磁盘分区扫描和恢复工具，恢复丢失的分区表 |
| gparted | 图形化分区编辑器，可视化管理磁盘分区，VMware虚拟机强烈推荐 |
| samba | SMB/CIFS文件共享服务，与Windows系统共享文件 |

```bash
tree 
htop 
tmux 
dos2unix 
cscope 
ctags
gpart
gparted
samba
```

---

## 完整安装命令

### 更新软件源
```bash
sudo apt update
```

### 一键安装所有软件包
```bash
sudo apt install -y open-vm-tools open-vm-tools-desktop build-essential gcc g++ make cmake tar gzip bzip2 lbzip2 xz-utils p7zip-full unzip gdb gdb-multiarch valgrind strace python2 python3 python3-pip bc bison flex libelf-dev libncurses-dev libssl-dev dwarves kmod cpio rsync ssh curl wget net-tools network-manager nmcli nmtui iproute2 iputils-ping traceroute tcpdump wireshark tshark nmap netcat-openbsd iperf3 ethtool bridge-utils vlan dnsutils telnet mtr iftop iptables gcc-arm-linux-gnueabi gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu git git-lfs device-tree-compiler u-boot-tools qemu-system-arm minicom screen picocom nfs-kernel-server tftp-hpa tftpd-hpa vim nano tree htop tmux dos2unix cscope ctags gpart gparted samba
```

### 软件列表

```ini
open-vm-tools
open-vm-tools-desktop
build-essential
gcc
g++
make
cmake
tar
gzip
bzip2
lbzip2
xz-utils
p7zip-full
unzip
gdb
gdb-multiarch
valgrind
strace
python2
python3
python3-pip
bc
bison
flex
libelf-dev
libncurses-dev
libssl-dev
dwarves
kmod
cpio
rsync
ssh
curl
wget
net-tools
network-manager
nmcli
nmtui
iproute2
iputils-ping
traceroute
tcpdump
wireshark
tshark
nmap
netcat-openbsd
iperf3
ethtool
bridge-utils
vlan
dnsutils
telnet
mtr
iftop
iptables
gcc-arm-linux-gnueabi
gcc-arm-linux-gnueabihf
gcc-aarch64-linux-gnu
git
git-lfs
device-tree-compiler
u-boot-tools
qemu-system-arm
minicom
screen
picocom
nfs-kernel-server
tftp-hpa
tftpd-hpa
vim
nano
tree
htop
tmux
dos2unix
cscope
ctags
gpart
gparted
samba
```

可以使用如下脚本(`preinstall_app.sh`)进行读取文件列表进行安装：
```shell
#!/bin/bash

if [ "$1"x == ""x ];then
	echo "\$1 is empty"
	exit 1
fi
app_list=`cat $1`
#echo "app lsit :${app_list[@]}"


while read line
do
#	echo $line
	sudo apt -y install $line &&
	echo "try to install $line"
done < $1
```

---

## 推荐精简安装

如果只是快速搭建基础嵌入式Linux开发环境，可以只安装以下核心软件包：

### 方案一：最小化安装（仅编译环境）
```bash
sudo apt install -y build-essential gcc g++ make cmake
```

### 方案二：基础开发环境（推荐）
```bash
sudo apt install -y \
    build-essential gcc g++ make cmake \
    git vim \
    tar gzip bzip2 xz-utils unzip \
    python3 python3-pip \
    ssh curl wget \
    gdb valgrind
```

### 方案三：完整嵌入式开发（包含交叉编译和调试）
```bash
sudo apt install -y \
    build-essential gcc g++ make cmake \
    git vim \
    tar gzip bzip2 xz-utils unzip \
    python3 python3-pip \
    ssh curl wget \
    gdb gdb-multiarch valgrind strace \
    bc bison flex libelf-dev libncurses-dev libssl-dev \
    gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu \
    device-tree-compiler u-boot-tools \
    minicom screen \
    nfs-kernel-server tftp-hpa tftpd-hpa
```

**说明：**
- **方案一**：适合只需要编译本地代码的场景
- **方案二**：适合日常开发，包含版本控制、调试和基本工具
- **方案三**：适合完整的嵌入式Linux开发，包含交叉编译、内核编译、设备调试等全套工具

## 使用说明

1. **首次安装建议先更新软件源**
   ```bash
   sudo apt update
   ```

2. **可以选择性安装**
   - 如果不使用VMware虚拟机，可跳过第1类
   - 如果已有特定架构的交叉编译器，可跳过第9类中不需要的架构
   - 如果不需要图形化网络分析工具，可跳过第8类中的wireshark

3. **安装后配置建议**
   - 配置NFS服务器：编辑`/etc/exports`
   - 配置TFTP服务器：编辑`/etc/default/tftpd-hpa`
   - 配置Git用户信息：`git config --global user.name/user.email`
   - NetworkManager管理网络：使用`nmcli`或`nmtui`配置网络连接
   - 查看网络接口状态：`ip addr`或`nmcli device status`

4. **验证安装**
   ```bash
   gcc --version
   arm-linux-gnueabihf-gcc --version
   make --version
   nmcli --version
   ip -V
   ```