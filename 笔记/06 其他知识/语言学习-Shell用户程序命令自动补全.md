---
share: "true"
---

> 我们在使用`ls`命令时，通常可以双击`tab`来自动补全命令，而用户自定义的脚本我们该如何使他自动补全。

我们先准备一个示例脚本程序`LinuxVersionCheck.sh`：
```shell
#!/bin/bash

# 这是一个检查Linux版本的脚本，通过源码目录
# "--src-dir": 表示Linux的根目录
# "--root-Makefile": 表示源码目录下的Makefile

# 函数：显示帮助信息
show_help() {
    echo "Usage: $0 [--src-dir <linux-source-directory>] [--root-Makefile <Makefile-path>]"
    echo ""
    echo "选项："
    echo "  --src-dir       指定 Linux 源码目录路径"
    echo "  --root-Makefile 指定 Linux 源码根目录下的 Makefile 路径"
}

# 解析命令行参数
while [[ "$1" != "" ]]; do
    case "$1" in
        --src-dir)
            shift
            KERNEL_SRC_DIR="$1"
            ;;
        --root-Makefile)
            shift
            MAKEFILE_PATH="$1"
            ;;
        -h | --help)
            show_help
            exit 0
            ;;
        *)
            echo "未知选项: $1"
            show_help
            exit 1
            ;;
    esac
    shift
done

# 优先检查是否提供了源码目录或Makefile文件
if [ -n "$KERNEL_SRC_DIR" ]; then
    # 如果提供了源码目录，则使用源码目录
    version_file="$KERNEL_SRC_DIR/Makefile"
elif [ -n "$MAKEFILE_PATH" ]; then
    # 如果提供了Makefile路径，则使用指定的Makefile
    version_file="$MAKEFILE_PATH"
else
    echo "错误：请提供 --src-dir 或 --root-Makefile 参数。"
    show_help
    exit 1
fi

# 检查 Makefile 是否存在
if [ ! -f "$version_file" ]; then
    echo "错误：无法找到 $version_file 文件，请检查路径是否正确。"
    exit 1
fi

# 解析 Makefile 中的版本信息
version=$(grep "^VERSION =" $version_file | awk '{print $3}')
patchlevel=$(grep "^PATCHLEVEL =" $version_file | awk '{print $3}')
sublevel=$(grep "^SUBLEVEL =" $version_file | awk '{print $3}')
extraversion=$(grep "^EXTRAVERSION =" $version_file | awk '{print $3}')

# 输出完整的内核版本信息
kernel_version="$version.$patchlevel.$sublevel$extraversion"
echo "Linux 内核版本：$kernel_version"

# 如果指定了源码目录，还可以尝试检查 utsrelease.h
if [ -n "$KERNEL_SRC_DIR" ]; then
    utsrelease_file="$KERNEL_SRC_DIR/include/generated/utsrelease.h"
    if [ -f "$utsrelease_file" ]; then
        uts_version=$(grep "#define UTS_RELEASE" $utsrelease_file | awk '{print $3}' | tr -d '"')
        echo "详细内核版本：$uts_version"
    else
        echo "警告：无法找到 $utsrelease_file 文件。"
    fi
fi
```
我们需要在：`~/.bashrc`或者`/etc/bash_completion.d/xxx.sh-completion`
+ `~/.bashrc`方式：
```shell
vim ~/.bashrc
# 编辑如下内容
_LinuxVersionCheck_completions()
{
    local cur prev opts
    COMPREPLY=()
    cur="${COMP_WORDS[COMP_CWORD]}"
    prev="${COMP_WORDS[COMP_CWORD-1]}"

    # 提供的选项
    opts="--src-dir --root-Makefile --help"

    # 如果当前输入的前缀是 --，提供选项补全
    if [[ ${cur} == --* ]] ; then
        COMPREPLY=( $(compgen -W "${opts}" -- ${cur}) )
        return 0
    fi

    # 对 --src-dir 或 --root-Makefile 提供路径补全
    if [[ ${prev} == "--src-dir" || ${prev} == "--root-Makefile" ]]; then
        COMPREPLY=( $(compgen -d -- ${cur}) )
        return 0
    fi
}

# 将补全函数与脚本 ./check_kernel_version.sh 关联
complete -F _LinuxVersionCheck_completions ./LinuxVersionCheck.sh
```
  ![[笔记/01 附件/语言学习-Shell用户程序命令自动补全/file-20241022143418007.png|笔记/01 附件/语言学习-Shell用户程序命令自动补全/file-20241022143418007.png]]
  
+ `/etc/bash_completion.d/xxx.sh-completion`方式：
  我们在`/etc/bash_completion.d/LinuxVersionCheck.sh-completion`:
```shell
_LinuxVersionCheck_completions()
{
    local cur prev opts
    COMPREPLY=()
    cur="${COMP_WORDS[COMP_CWORD]}"
    prev="${COMP_WORDS[COMP_CWORD-1]}"

    # 提供的选项
    opts="--src-dir --root-Makefile --help"

    # 如果当前输入的前缀是 --，提供选项补全
    if [[ ${cur} == --* ]] ; then
        COMPREPLY=( $(compgen -W "${opts}" -- ${cur}) )
        return 0
    fi

    # 对 --src-dir 或 --root-Makefile 提供路径补全
    if [[ ${prev} == "--src-dir" || ${prev} == "--root-Makefile" ]]; then
        COMPREPLY=( $(compgen -d -- ${cur}) )
        return 0
    fi
}

# 将补全函数与脚本 ./check_kernel_version.sh 关联
complete -F _LinuxVersionCheck_completions ./LinuxVersionCheck.sh
```
同样可以实现上面的原理。

**注意**：当然一般脚本要考虑可迁移性，所以最好将以上两种该注入方式编写到脚本中，且脚本可以放至`/usr/bin`这种可以全局搜索的目录中，这样可以便于发行。