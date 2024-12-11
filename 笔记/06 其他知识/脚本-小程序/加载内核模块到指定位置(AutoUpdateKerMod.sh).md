---
share: "true"
---

```shell
#!/bin/bash

# 该脚本主要是用来更新内核模块
# 他会检查当前要更新的模块内核版本以及当前已经加载的对应模块的路径地址，然后判断是否更新

# 检查是否提供了参数
if [ $# -lt 1 ]; then
    # 如果没有给定参数，则寻找当前目录下的所有 .ko 文件
    KO_FILES=(*.ko)
    
    if [ ${#KO_FILES[@]} -eq 0 ]; then
        echo "当前目录没有找到任何 .ko 文件，请提供模块路径作为参数。"
        exit 1
    else
        echo "未提供参数，自动找到以下 .ko 文件进行处理："
        for ko_file in "${KO_FILES[@]}"; do
            echo "$ko_file"
        done
    fi
else
    KO_FILES=("$1")
fi

for i in "${!KO_FILES[@]}"; do
    MODULE_PATH=`realpath ${KO_FILES[$i]}`
    NEW_MODULE_DIR=`dirname $MODULE_PATH`
    NEW_MODULE_BASENAME=$(basename $MODULE_PATH)
    MODULE_NAME=`modinfo -F name $(basename $MODULE_PATH)`
    #echo "新模块:$MODULE_NAME,位于:$NEW_MODULE_DIR,全路径:$MODULE_PATH"

    # 获取当前系统的内核版本
    CURRENT_KERNEL=$(uname -r)
    #echo "Current kernel version: $CURRENT_KERNEL"

    # 查找当前加载的模块路径
    CURRENT_MODULE_PATH=$(modinfo -n "$MODULE_NAME" 2>/dev/null)
    #echo "Current loaded module path: $CURRENT_MODULE_PATH"

    # 检查模块是否已加载
    if [ -z "$CURRENT_MODULE_PATH" ]; then
        #echo "Module $MODULE_NAME is not loaded or not found."
        continue
    fi

    # 找到新的 .ko 文件
    NEW_MODULE_PATH="$NEW_MODULE_DIR/$NEW_MODULE_BASENAME"
    if [ ! -f "$NEW_MODULE_PATH" ]; then
        #echo "New module file $NEW_MODULE_PATH does not exist."
        continue
    fi

    # 获取新模块的内核版本 (vermagic)
    NEW_MODULE_KERNEL=$(modinfo -F vermagic "$NEW_MODULE_PATH" | awk '{print $1}')
    #echo "New module kernel version: $NEW_MODULE_KERNEL"

    # 查找新模块对应的内核模块路径
    NEW_MODULE_DEST=`dirname $(echo "$CURRENT_MODULE_PATH" | sed "s/$CURRENT_KERNEL/$NEW_MODULE_KERNEL/")`
    if [ ! -d "$NEW_MODULE_DEST" ]; then
        #echo "Creating directory: $NEW_MODULE_DEST"
        mkdir -p "$NEW_MODULE_DEST"
    fi

    #echo "New module load path: $NEW_MODULE_DEST"

    # 检查新模块的内核版本是否与当前系统内核匹配
    if [ "$NEW_MODULE_KERNEL" != "$CURRENT_KERNEL" ]; then
        #echo "Kernel version mismatch! Moving the new module to $NEW_MODULE_DEST"
        cp "$NEW_MODULE_PATH" "$NEW_MODULE_DEST"
        if [ $? -eq 0 ]; then
            #echo "New module $NEW_MODULE_BASENAME successfully copied to $NEW_MODULE_DEST"
			echo -e "\033[33m[$((i+1))]:\033[0m\033[32m[$MODULE_PATH]\033[0m >->-> [$NEW_MODULE_DEST] \033[33m[当前:$CURRENT_KERNEL|模块:$NEW_MODULE_KERNEL]\033[0m"
        else
            echo "Failed to copy the new module."
            continue
        fi
    else
        # 替换当前的模块
        #echo "Replacing the current module with the new one..."
        cp "$NEW_MODULE_PATH" "$CURRENT_MODULE_PATH"
        if [ $? -eq 0 ]; then
            #echo "Module $NEW_MODULE_BASENAME successfully updated."
			echo -e "[$((i+1))]:\033[32m[$MODULE_PATH]\033[0m >->-> [$NEW_MODULE_DEST]"
        else
            echo "Failed to update the module."
            continue
        fi
    fi
	continue

    # 更新模块依赖
    echo "Updating module dependencies..."
    depmod -a

    # 卸载旧模块并重新加载新模块
    echo "Reloading module $MODULE_NAME..."
    modprobe -r "$MODULE_NAME"
    if modprobe "$MODULE_NAME"; then
        echo "Module $MODULE_NAME reloaded successfully."
    else
        echo "Failed to reload module $MODULE_NAME."
    fi
done

exit 0

```