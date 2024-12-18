---
share: "true"
---
> 这是一个用来自动设置`KGDB`波特率，以及寻找连接到的`USB tty`，我们将其命名为`.gdbinit`
> 然后放置在调试目录下。

```shell

python
import subprocess
import re
import sys

def get_ttyusb():
    try:
        # 使用Popen获取dmesg输出，不进行解码
        process = subprocess.Popen(['dmesg'], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        output, error = process.communicate()
        
        # 使用errors='ignore'忽略无法解码的字符
        if sys.version_info[0] >= 3:
            output = output.decode('utf-8', errors='ignore')
        else:
            output = output.decode('utf-8', errors='ignore')
        
        # 查找所有ttyUSB设备
        matches = re.findall(r'ttyUSB(\d+)', output)
        
        if matches:
            # 获取最后一个ttyUSB设备号
            last_ttyusb = matches[-1]
            # 设置GDB变量
            gdb.execute('set $ttyusb = "%s"' % last_ttyusb)
            
            # 设置串口波特率
            print("Setting baud rate: 115200")
            gdb.execute('set serial baud 115200')
            
            # 构建target remote命令并执行
            remote_cmd = 'target remote /dev/ttyUSB%s' % last_ttyusb
            print("Connecting to %s" % remote_cmd)
            try:
                gdb.execute(remote_cmd)
                print("Successfully connected to /dev/ttyUSB%s" % last_ttyusb)
            except gdb.error as e:
                print("Connection failed: %s" % str(e))
            
            return last_ttyusb
        else:
            print("No ttyUSB device found")
            return None
    except Exception as e:
        print("Error getting ttyUSB info: %s" % str(e))
        return None

# 创建一个GDB命令来获取ttyUSB信息并连接
class GetTTYUSB(gdb.Command):
    def __init__(self):
        super(GetTTYUSB, self).__init__("get-ttyusb", gdb.COMMAND_USER)
    
    def invoke(self, arg, from_tty):
        get_ttyusb()

GetTTYUSB()
end

# 定义变量 set_tty
set $set_tty = "/dev/ttyVirtual0"

# 选择手动设置还是自动搜索USB串口
# 自动连接到目标
define connect-kgdb
    if $_strlen($set_tty) == 0
        echo "set_tty is empty. Auto-detecting USB TTY devices..."\n
        # 自动检测 USB 串口设备的逻辑
        # 这里可以使用 GDB 命令来执行设备检测，或者使用其他外部命令
        python get_ttyusb()
        set $manual_tty=0
    else
        printf "使用指定tty设备: %s\n", $set_tty
    end
end

# 设置串口波特率
echo "设置波特率为: 115200"\n
set serial baud 115200

# 连接设备
#printf "Checking device: %s\n", $set_tty
connect-kgdb

python
import subprocess
import re
import sys
# 获取 set_tty 变量
tty = gdb.parse_and_eval('$set_tty')
if tty and str(tty) != "":
    # 如果 tty 变量存在且不为空，执行连接
    tty_str = str(tty).strip('"')  # 去除双引号
    remote_cmd = 'target remote %s' % tty_str
    gdb.execute(remote_cmd)
end

```