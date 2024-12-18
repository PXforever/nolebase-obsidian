---
share: "true"
---

```shell
#!/bin/bash
# -*- coding: utf-8 -*-

# 设置 Kernel_Dir，如果没有指定则默认为当前目录
Kernel_Dir=${1:-$(pwd)}

# 临时存储文件目录
Temp_Dir=comit2patch

# 指定开始位置，这个设置最早从Start_Commit位置开始，直到当前HEAD位置的commit（注意该commit不会被包含）
# 如果不指定的话则到最初始的提交
if [ -n $1 ]; then
    Start_Commit=2b0e5c2b20346e2c175f04cace7b64af2e876f1a
else
    Start_Commit=$1
fi

# 检查指定目录是否存在 Makefile
if [ ! -f "$Kernel_Dir/Makefile" ]; then
    echo "错误: 在目录 $Kernel_Dir 中未找到 Makefile。"
    exit 1
fi

# 检查 Makefile 中的版本信息
version_major=$(grep -E '^VERSION\s*=' "$Kernel_Dir/Makefile" | awk -F '=' '{print $2}' | xargs)
version_patchlevel=$(grep -E '^PATCHLEVEL\s*=' "$Kernel_Dir/Makefile" | awk -F '=' '{print $2}' | xargs)
version_sublevel=$(grep -E '^SUBLEVEL\s*=' "$Kernel_Dir/Makefile" | awk -F '=' '{print $2}' | xargs)

# 检查是否都成功提取
if [ -z "$version_major" ] || [ -z "$version_patchlevel" ] || [ -z "$version_sublevel" ]; then
    echo "错误: 在 Makefile 中未找到完整的版本信息。"
    exit 1
fi

# 组合完整的版本号
full_version="${version_major}.${version_patchlevel}.${version_sublevel}"

echo ">>>> 找到的内核版本: $full_version <<<<"

# 当前脚本的文件名
current_script=$(basename "$0")

# 创建一个临时目录来存放未跟踪的文件
Temp_Dir=${Temp_Dir:=comit2patch}
mkdir -p $Temp_Dir

echo ">>>> 生成Patch中...."
if [ -d $Temp_Dir/kernel_patch/${full_version} ]; then
    echo -e "\t 删除目录$Temp_Dir/kernel_patch/${full_version}下的文件"
    rm $Temp_Dir/kernel_patch/${full_version}/*
fi
if [ -n "$Start_Commit" ]; then
    git format-patch --binary --root $Start_Commit..HEAD -o $Temp_Dir/kernel_patch/${full_version}
else
    git format-patch --binary --root -o $Temp_Dir/kernel_patch/${full_version}
fi

echo ">>> 打包全部..."
tar -czf "ker_${full_version}_Patchs.tar.gz" -C $Temp_Dir/ . #打包补丁文件

echo -e "\t完成的补丁为：ker_${full_version}_Patchs.tar.gz"
rm -r $Temp_Dir

# 使用方法
# 1.解压
# 2.进入内核目录
# 3.如果是git仓库：git am --ignore-whitespace /path/to/patch/directory/*.patch;git apply /path/to/patch/directory/*.patch
# 4.非git仓库：find ../kernel_patch/6.6.29/ -name "*.patch" | sort | xargs -I {} patch -p1 -N --batch -i {}  #部署补丁
```