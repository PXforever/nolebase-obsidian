---
share: "true"
---

```shell
#!/bin/bash

# 显示使用说明
usage() {
    echo "用法: $0 [输出目录]"
    echo
    echo "如果未指定输出目录，设备树源文件将保存在当前目录。"
    echo "输出目录必须是有效的目录路径。"
    echo
    echo "例如："
    echo "  $0 /path/to/output/directory"
    echo "  $0  # 将生成的文件保存在当前目录"
    exit 1
}

# 检查参数个数
if [[ $# -gt 1 ]]; then
    usage
fi

# 设置默认输出目录（当前目录）
OUTPUT_DIR="."
DTS_FILE="extracted_device_tree.dts"
TEMP_DIR="/tmp/device_tree"

# 如果用户提供了输出路径，则使用用户提供的路径
if [ ! -z "$1" ]; then
    OUTPUT_DIR="$1"
fi

# 检查输出目录是否存在
if [ ! -d "$OUTPUT_DIR" ]; then
    echo "错误: 输出目录 '$OUTPUT_DIR' 不存在！"
    usage
fi

# 设置输出文件的完整路径
DTS_FILE_PATH="$OUTPUT_DIR/$DTS_FILE"

# 确保脚本以 root 权限运行
if [[ $EUID -ne 0 ]]; then
    echo "请以 root 权限运行此脚本！"
    exit 1
fi

# 检查是否安装了 dtc 工具
if ! command -v dtc &> /dev/null; then
    echo "未找到 dtc (设备树编译器)。请安装后重试。"
    echo "例如，在 Ubuntu 上运行: sudo apt install device-tree-compiler"
    exit 1
fi

# 创建临时目录
echo "创建临时目录: $TEMP_DIR"
mkdir -p "$TEMP_DIR"

# 复制 /proc/device-tree 到临时目录
echo "复制 /proc/device-tree 到临时目录..."
cp -R /proc/device-tree/* "$TEMP_DIR"

# 将临时目录转换为设备树源文件 (DTS)
echo "生成设备树源文件: $DTS_FILE_PATH"
dtc -I fs -O dts -o "$DTS_FILE_PATH" "$TEMP_DIR"
if [[ $? -ne 0 ]]; then
    echo "生成 DTS 文件失败！"
    rm -rf "$TEMP_DIR"
    exit 1
fi

# 清理临时目录
echo "清理临时目录..."
rm -rf "$TEMP_DIR"

# 提示完成
echo "生成完成！"
echo "设备树源文件: $DTS_FILE_PATH"
```