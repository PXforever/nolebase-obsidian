---
share: "true"
---
> 这是一个用来测试DRM显示框架的脚本，其基本是需要`modetest`

```shell
#!/bin/bash

# 设置固定的 CONNECTOR_ID 和 CRTC_ID
CONNECTOR_ID=222
CRTC_ID=71
PANEL_ID=55

# 设备名称
DEV_NAME='rockchip'

# 颜色定义，使输出更加清晰
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
MAGENTA='\033[0;35m'
CYAN='\033[0;36m'
NC='\033[0m'

# 日志函数
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# 执行测试并检查结果
execute_test() {
    local test_name="$1"
    local cmd="$2"
    local timeout_duration="${3:-5}"  # 默认超时时间为 5 秒
    
    log_info "执行测试: $test_name"
    log_info "命令: $cmd"
    
    # 使用 timeout 限制执行时间
    timeout $timeout_duration $cmd
    local result=$?
    
    # 确保 result 不为空，防止进行比较时出错
    if [ -z "$result" ]; then
        result=1  # 如果 result 为空，手动设置为失败
    fi
    
    if [ $result -eq 0 ]; then
        log_info "${test_name} 测试通过"
    else
        log_error "${test_name} 测试失败"
    fi
    
    return $result
}

# 颜色图案测试函数
color_pattern_tests() {
    log_info "开始颜色和图案测试..."
    
    # 测试基本颜色填充模式
    local color_patterns=(
        "0xff0000"    # 红色
        "0x00ff00"    # 绿色
        "0x0000ff"    # 蓝色
        "0xffff00"    # 黄色
        "0xff00ff"    # 紫色
        "0x00ffff"    # 青色
    )
	
	# 可用图案列表
	local inside_pattern=(
		"tiles"
		"smpte"
		"plain"
		"gradient"
	)
    
	# 测试内置图案
    for pattern in "${inside_pattern[@]}"; do
		execute_test "内置图案测试 $pattern" \
			"modetest -M $DEV_NAME -D 0 -a -s $CONNECTOR_ID@$CRTC_ID:1920x1080 -P $PANEL_ID@$CRTC_ID:1920x1080 -F$pattern"
    done
}

# 高级颜色和图案测试
advanced_color_tests() {
    log_info "开始高级颜色测试..."
    
    # 测试不同像素格式的颜色填充
    local pixel_formats=(
        "XRGB8888"
        "ARGB8888"
        "RGB565"
    )
    
    for format in "${pixel_formats[@]}"; do
        execute_test "像素格式 $format 颜色测试" \
            "modetest -s $CONNECTOR_ID@$CRTC_ID:1920x1080@$format -F 0xff0000"
    done
    
    # 测试不同大小的颜色块
    local resolutions=(
        "640x480"
        "1280x720"
        "1920x1080"
        "3840x2160"
    )
    
    for res in "${resolutions[@]}"; do
        execute_test "分辨率 $res 颜色测试" \
            "modetest -s $CONNECTOR_ID@$CRTC_ID:$res -F 0x00ff00"
    done
	
	# 测试vsync
	execute_test "vsync测试" \
		"modetest -M $DEV_NAME -s $CONNECTOR_ID@$CRTC_ID:1920x1080 -v"
		
	# 测试hw cursor
	execute_test "hw cursor测试" \
		"modetest -M $DEV_NAME -s $CONNECTOR_ID@$CRTC_ID:1920x1080 -C"
}

# 主测试流程
main() {
    log_info "开始 DRM Modetest 颜色和图案测试..."
    
    color_pattern_tests
    advanced_color_tests
    
    log_info "DRM Modetest 颜色和图案测试完成"
	modetest -d
}

# 执行主测试
main

# 参考链接：https://www.cnblogs.com/yaongtime/p/12940386.html
```