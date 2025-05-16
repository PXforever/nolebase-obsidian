---
share: "true"
---

某些时候，我们希望能够保存当前的`rootfs`，重新打包为一个`img`，在本机中操作难免会因为某些活动而发生问题，我们可以通过`rsync`来实现：

同步至本地(需要较长时间)
```shell
sudo rsync -avx root@192.168.0.93:/ ./remote_rootfs
```
创建一个`remote_rootfs.img`
```shell
# 空间设为11GB
dd if=/dev/zero of=remote_rootfs.img bs=1M count=10240

# 格式化
sudo mkfs.ext4 -F -L linuxroot remote_rootfs.img
```
创建一个挂载目录：
```shell
mkdir rootfs_mount_tmp
```
挂载，并且复制：
```shell
sudo mount remote_rootfs.img rootfs_mount_tmp
sudo cp -rfp remote_rootfs/* rootfs_mount_tmp
sudo umount rootfs_mount_tmp
```
检查`remote_rootfs.img`，并且缩小尺寸：
```shell
sudo e2fsck -p -f remote_rootfs.img
sudo resize2fs -M remote_rootfs.img
```
---
当然上面可以总结为一个脚本(`make_remote_rootfs_img.sh`)：
```shell
#!/bin/bash

set -e

usage() {
    echo "用法: $0 [-v] <remote_ip>"
    echo "  -v      显示 rsync 详细复制过程"
    exit 1
}

# 参数解析
VERBOSE=0
while getopts ":v" opt; do
    case $opt in
        v) VERBOSE=1 ;;
        *) usage ;;
    esac
done
shift $((OPTIND - 1))

if [ $# -ne 1 ]; then
    usage
fi

REMOTE_IP="$1"
REMOTE_DIR="/"
LOCAL_DIR="./remote_rootfs"
IMG_NAME="remote_rootfs.img"
MOUNT_DIR="rootfs_mount_tmp"
IMG_SIZE_MB=11000

echo
echo "===================== 🛠️ remote rootfs 镜像制作脚本 ====================="
echo

### ① 同步 rootfs
echo "① >>>>>===== 开始同步远程 rootfs: root@$REMOTE_IP:$REMOTE_DIR ====="
if [ ! -d "$LOCAL_DIR" ]; then
    echo "🔧 创建本地目录: $LOCAL_DIR"
    mkdir -p "$LOCAL_DIR"
fi

# 检查 pv 是否可用
if command -v pv >/dev/null 2>&1; then
    echo "🔍 检测到 pv，使用 tar + pv 简洁模式"

    if [ "$VERBOSE" -eq 1 ]; then
        echo "🔧 详细模式启用（tar + pv 显示文件名）"
        ssh root@$REMOTE_IP "tar -cvf - --one-file-system $REMOTE_DIR" | \
            pv | sudo tar -xvf - -C "$LOCAL_DIR"
    else
        echo "🧮 正在估算远程 rootfs 总大小..."
        TOTAL_KB=$(ssh root@$REMOTE_IP "du -sx --exclude=/proc --exclude=/sys --exclude=/dev --exclude=/run /" | awk '{print $1}')
        TOTAL_BYTES=$((TOTAL_KB * 1024))
        echo "📦 总大小: $((TOTAL_KB / 1024)) MB"

        ssh root@$REMOTE_IP "tar -cf - --one-file-system \
            --exclude=/proc --exclude=/sys --exclude=/dev --exclude=/run \
            -C / ." | \
            pv -s "$TOTAL_BYTES" | sudo tar -xf - -C "$LOCAL_DIR"
    fi

    echo -e "\n✅ 同步完成"

else
    echo "⚠️ 使用 rsync 进行同步，开始进度监控"

    # 创建一个显示进度的后台进程
    show_progress() {
        local count=0
        local spinner=('⠋' '⠙' '⠹' '⠸' '⠼' '⠴' '⠦' '⠧' '⠇' '⠏')
        
        while [ -f "$LOCAL_DIR/.sync_running" ]; do
            # 检查本地目录大小
            if [ -d "$LOCAL_DIR" ]; then
                local size=$(du -sh "$LOCAL_DIR" 2>/dev/null | cut -f1)
                echo -ne "\r⏳ 正在同步: ${spinner[$((count % 10))]} 已传输 $size "
            else
                echo -ne "\r⏳ 正在同步: ${spinner[$((count % 10))]} 准备中... "
            fi
            count=$((count+1))
            sleep 1
        done
        echo -e "\r✅ 同步完成                         "
    }

    # 创建标记文件
    touch "$LOCAL_DIR/.sync_running"
    
    # 启动进度显示（使用后台子shell而不是单独进程）
    show_progress &
    PROGRESS_PID=$!
    
    # 执行 rsync 命令
    echo "📂 开始远程同步，可能需要几分钟..."
    if [ "$VERBOSE" -eq 1 ]; then
        echo "🔧 详细模式启用，将显示所有 rsync 输出"
        sudo rsync -ahv --info=progress2 -x --exclude=/proc --exclude=/sys --exclude=/dev --exclude=/run root@$REMOTE_IP:"$REMOTE_DIR" "$LOCAL_DIR"
        RSYNC_STATUS=$?
    else
        # 重定向错误输出到临时文件以便查看可能的错误
        sudo rsync -ah --info=progress2 -x --exclude=/proc --exclude=/sys --exclude=/dev --exclude=/run root@$REMOTE_IP:"$REMOTE_DIR" "$LOCAL_DIR" > /dev/null 2>rsync_errors.log
        RSYNC_STATUS=$?
    fi

    # 移除标记文件停止进度显示
    rm -f "$LOCAL_DIR/.sync_running"
    
    # 等待进度显示进程结束（最多等待5秒）
    WAIT_COUNT=0
    while kill -0 $PROGRESS_PID 2>/dev/null && [ $WAIT_COUNT -lt 5 ]; do
        sleep 1
        WAIT_COUNT=$((WAIT_COUNT+1))
    done
    
    # 如果进程仍在运行，强制终止
    if kill -0 $PROGRESS_PID 2>/dev/null; then
        kill $PROGRESS_PID 2>/dev/null
    fi

    # 检查 rsync 退出状态
    if [ $RSYNC_STATUS -ne 0 ]; then
        echo "❌ rsync 同步失败，退出码: $RSYNC_STATUS"
        if [ -f rsync_errors.log ]; then
            echo "错误日志:"
            cat rsync_errors.log
        fi
        exit 1
    fi

    # 检查同步是否成功 - 如果目录为空则失败
    if [ ! "$(ls -A "$LOCAL_DIR" | grep -v ".sync_running")" ]; then
        echo "❌ 同步失败: $LOCAL_DIR 目录是空的，请检查远程连接和权限"
        exit 1
    fi
    
    echo "📊 同步统计: $(du -sh "$LOCAL_DIR" | cut -f1) 数据已传输"
fi

### ② 创建空镜像
echo
echo "② >>>>>===== 创建 $IMG_SIZE_MB MB 的空 ext4 镜像：$IMG_NAME ====="
dd if=/dev/zero of=$IMG_NAME bs=1M count=$IMG_SIZE_MB status=progress
sudo mkfs.ext4 -F -L linuxroot $IMG_NAME
echo "✅ 镜像创建并格式化完成"

### ③ 挂载镜像并复制 rootfs
echo
echo "③ >>>>>===== 挂载镜像并复制 rootfs 内容 ====="
mkdir -p $MOUNT_DIR
sudo mount $IMG_NAME $MOUNT_DIR
echo "⏳ 正在复制 rootfs 内容到镜像（可能较慢）..."

# 使用 rsync 显示进度，保留权限和符号链接，-a归档，-h人类可读，--info=progress2 显示整体进度
sudo rsync -ah --info=progress2 --delete "$LOCAL_DIR"/ "$MOUNT_DIR"/

sync
sudo umount $MOUNT_DIR
echo "✅ 镜像填充完成"

### ④ 检查并缩小镜像
echo
echo "④ >>>>>===== 检查并缩小 ext4 镜像大小 ====="
sudo e2fsck -p -f $IMG_NAME
sudo resize2fs -M $IMG_NAME
echo "✅ 镜像压缩完成"

### 🧹 清理
echo
echo "⑤ >>>>>===== 清理临时挂载目录 ====="
rm -rf $MOUNT_DIR
echo "✅ 清理完成"

### ✅ 完成
echo
echo "🎉 所有步骤完成，最终生成镜像：$IMG_NAME"
echo "======================================================================"
```