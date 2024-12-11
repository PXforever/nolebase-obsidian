---
share: "true"
---

```shell
#!/bin/bash

# 检查参数数量
if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <input_directory> <output_directory>"
    exit 1
fi

# 指定输入目录和输出目录
INPUT_DIRECTORY="$1"
OUTPUT_DIRECTORY="$2"

# 创建输出目录（如果不存在）
mkdir -p "$OUTPUT_DIRECTORY"

# 遍历输入目录中的所有 DTB 文件
for dtb_file in "$INPUT_DIRECTORY"/*.dtb; do
    # 检查文件是否存在
    if [[ -f "$dtb_file" ]]; then
        # 提取文件名（不带扩展名）
        base_name=$(basename "$dtb_file" .dtb)
        # 转换为 DTS 格式
        dtc -I dtb -O dts -o "$OUTPUT_DIRECTORY/$base_name.dts" "$dtb_file"
        echo "Converted $dtb_file to $OUTPUT_DIRECTORY/$base_name.dts"
    fi
done
```