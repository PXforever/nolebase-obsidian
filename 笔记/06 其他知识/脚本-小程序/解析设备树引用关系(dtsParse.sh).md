---
share: "true"
---
> 这是一个用来解析设备树的引用关系的脚本，比如一个`xxx.dts`可能会引用多个`dtsi`文件，我们可以将其打印出来。
> 例如：`./dtsParse.sh arch/arm64/boot/dts/rockchip/rk3588-linux.dts arch/arm64/boot/dts/rockchip`

下载文件[dtsParse.sh](https://my-download-1333032936.cos.ap-guangzhou.myqcloud.com/dtsParse.sh)

```shell
#!/bin/bash

declare -g HEAD_FILES=()    #头文件
declare -g DT_FILES=()      #设备树文件

# 用于解析 DTS 文件及其引用的 DTSI 文件的函数
function parse_dts() {
    local current_dts_file="$1"    # 当前的 DTS 文件
    local current_dts_dir="$2"     # 当前 DTS 文件所在的目录
    local current_indent="$3"      # 缩进层次，用于打印树状结构
    local header_files=()

    echo "${current_indent}正在解析文件: $current_dts_file"
    DT_FILES+=("$current_dts_file");

    # 使用 grep 提取所有 #include 语句
    local includes=$(grep -E '^\s*#include\s+("|<)[^">]+("|>)' "$current_dts_file")

    header_files+=("xxx")

    # 先打印所有找到的 include 语句
    if [[ -n "$includes" ]]; then
        echo "${current_indent}找到以下包含文件:"
        while IFS= read -r line; do
            #echo "${current_indent}  $line"
            if [[ "$line" =~ ([a-zA-Z0-9_/.,-]+\.h) ]];then
                filename="${BASH_REMATCH[1]}"  # 提取匹配的头文件名
                HEAD_FILES+=("$filename")     # 添加到数组中
            fi
        done <<< "$includes"   # 将 includes 直接重定向到 while 循环
    else
        :
        echo "${current_indent}没有找到包含文件。"
    fi

    # 遍历所有找到的 include 语句并递归解析
    while IFS= read -r line; do
        # 从 include 语句中提取文件名
        if [[ "$line" =~ \"(.*)\" ]]; then
            local include_file="${BASH_REMATCH[1]}"
            
            # 标准化路径，避免出现双斜杠
            local include_path=$(realpath -m "$current_dts_dir/$include_file")

            if [[ -f "$include_path" ]]; then
                # 递归调用自身解析包含的文件，缩进增加
                parse_dts "$include_path" "$current_dts_dir" "    $current_indent"
            else
                echo -e "${current_indent}警告: \e[33m找不到包含文件 $include_file\e[0m"
            fi
        fi

    done <<< "$includes"
    
}

# 用于统计并打印二级节点的函数
function print_second_level_nodes() {
    local current_dts_file="$1"    # 当前的 DTS 文件
    local current_indent="$2"      # 缩进层次，用于打印树状结构

    # 初始化变量用于二级节点统计
    local level=0                  # 当前层次，初始为0表示文件最外层
    local second_level_node_count=0 # 二级节点计数
    local second_level_nodes=()     # 存储二级节点名称的数组

    # 重新遍历当前 DTS 文件以统计二级节点
    while IFS= read -r line; do
        # 删除行内注释（以 "//" 开头的注释）
        line=$(echo "$line" | sed 's/\/\/.*//')

        # 检测大括号的层次，"{" 表示进入下一层，"}" 表示回到上一层
        if [[ "$line" =~ "{" ]]; then
            level=$((level + 1))
        elif [[ "$line" =~ "}" ]]; then
            level=$((level - 1))
        fi

        # 检测二级节点（层次为2）
        if [[ $level -eq 2 && "$line" =~ ^[a-zA-Z0-9_,-]+: ]]; then
            second_level_node_count=$((second_level_node_count + 1))
            # 提取节点名称并加入到数组中
            local node_name=$(echo "$line" | cut -d ':' -f 1)
            second_level_nodes+=("$node_name")
        fi
    done < <(grep -v '/\*.*\*/' "$current_dts_file")  # 过滤掉多行注释

    # 打印二级节点数量和节点名称
    echo "${current_indent}二级节点数量: $second_level_node_count"
    if [[ $second_level_node_count -gt 0 ]]; then
        echo "${current_indent}二级节点名称:"
        for node in "${second_level_nodes[@]}"; do
            echo "${current_indent}  $node"
        done
    fi

    # 再次打印分隔符
    echo "${current_indent}======================"
}

# 检查传入的文件参数
if [[ $# -ne 2 ]]; then
    echo "功能：递归解析一个设备树中(dts,dtsi)的引用情况"
    echo "用法: $0 <dts文件路径> <dts目录>"
    exit 1
fi

# 获取传入的 DTS 文件路径和目录
input_dts_file=$(realpath $1)
input_dts_dir="$2"

# 检查文件是否存在
if [[ -f "$input_dts_file" && -d "$input_dts_dir" ]]; then
    # 调用解析函数，传入初始缩进为空字符串
    parse_dts "$input_dts_file" "$input_dts_dir" "";
else
    echo "错误: 找不到文件或目录"
    exit 1
fi

echo "*******************************************************************************"
echo ""

# 去重并排序相同前缀
unique_sorted_dt_files=($(printf "%s\n" "${DT_FILES[@]}" | sort | uniq | sort -t. -k1,1))

echo "其中的设备树文件:"
for dt_file in "${unique_sorted_dt_files[@]}"; do
    echo -e "\t$dt_file"
done

echo "*******************************************************************************"
echo ""

echo "==============================================================================="
echo ""

# 去重并排序相同前缀
unique_sorted_header_files=($(echo "${HEAD_FILES[@]}" | tr ' ' '\n' | sort | uniq | tr '\n' ' '))

echo "其中的头文件:"
for headers in "${unique_sorted_header_files[@]}"; do
    echo -e "\t$headers"
done

echo "==============================================================================="
echo ""

# 统计并打印二级节点
print_second_level_nodes "$input_dts_file" ""


```