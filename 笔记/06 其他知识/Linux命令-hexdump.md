---
share: "true"
---
## 1. 基本用法

```bash
hexdump [options] <file>
```

最简单的用法：
```bash
hexdump file.bin          # 默认格式输出
hexdump -C file.bin       # 最常用：十六进制 + ASCII 显示
```

---

## 2. 常用选项详解

### 2.1 显示格式选项

| 选项 | 说明 | 示例输出 |
|------|------|----------|
| `-b` | 单字节八进制 | `0000000 101 102 103 104` |
| `-c` | 单字节字符 | `0000000   A   B   C   D` |
| `-C` | 经典格式（十六进制+ASCII） | `00000000  41 42 43 44 \|ABCD\|` |
| `-d` | 双字节十进制 | `0000000 16706 17220` |
| `-o` | 双字节八进制 | `0000000 041101 042103` |
| `-x` | 双字节十六进制 | `0000000 4241 4443` |

### 2.2 控制选项
---

| 选项 | 说明 | 示例 |
|------|------|------|
| `-n <length>` | 只读取指定字节数 | `hexdump -C -n 64 file` |
| `-s <offset>` | 跳过开头指定字节 | `hexdump -C -s 0x100 file` |
| `-v` | 显示所有行（不压缩重复行） | `hexdump -Cv file` |
| `-e <format>` | 自定义格式字符串 | 见下文详解 |
| `-f <file>` | 从文件读取格式字符串 | `hexdump -f fmt.txt file` |

### 2.3 `-n` 和 `-s` 支持的单位后缀

```bash
hexdump -n 1K file      # 1024 字节
hexdump -n 1M file      # 1MB
hexdump -s 4K file      # 跳过 4KB
```

---

## 3. `-e` 格式字符串详解（重点）

### 3.1 基本语法

```
-e '重复次数/字节数 "格式字符串"'
```

格式：`'迭代次数/字节数 "格式"'`

- **迭代次数**：每行重复多少次（可省略，默认尽可能多）
- **字节数**：每次读取的字节数
- **格式**：printf 风格的格式字符串

### 3.2 格式说明符

| 说明符 | 含义 |
|--------|------|
| `%d` | 有符号十进制 |
| `%u` | 无符号十进制 |
| `%x` | 小写十六进制 |
| `%X` | 大写十六进制 |
| `%o` | 八进制 |
| `%c` | 字符 |
| `%_a` | 当前偏移地址（十六进制）|
| `%_A` | 当前偏移地址（带换行）|
| `%_p` | 可打印字符，不可打印显示 `.` |

### 3.3 字节数与数据类型对应

| 字节数 | 数据类型 |
|--------|----------|
| 1 | uint8_t / char |
| 2 | uint16_t / short |
| 4 | uint32_t / int |
| 8 | uint64_t / long long |

---

## 4. 常用格式示例

### 4.1 基本十六进制显示

```bash
# 每行 16 字节，空格分隔
hexdump -e '16/1 "%02x " "\n"' file

# 带地址显示
hexdump -e '"%08_ax: " 16/1 "%02x " "\n"' file
```

### 4.2 查看 8-bit 数据

```bash
# 无符号十进制
hexdump -e '8/1 "%3u " "\n"' file

# 十六进制
hexdump -e '8/1 "%02x " "\n"' file

# 二进制文件逐字节
hexdump -e '1/1 "0x%02x\n"' file
```

### 4.3 查看 16-bit 数据（寄存器/半字）

```bash
# 小端序 16-bit 十六进制
hexdump -e '8/2 "%04x " "\n"' file

# 带地址
hexdump -e '"%08_ax: " 4/2 "0x%04x " "\n"' file
```

### 4.4 查看 32-bit 数据（寄存器/字）

```bash
# 32-bit 十六进制，每行 4 个
hexdump -e '4/4 "%08x " "\n"' file

# 带地址，常用于查看寄存器
hexdump -e '"%08_ax: " 4/4 "0x%08x " "\n"' file

# 单个 32-bit 值
hexdump -e '1/4 "0x%08x\n"' file
```

### 4.5 查看 64-bit 数据（地址/长整型）

```bash
# 64-bit 十六进制
hexdump -e '2/8 "%016x " "\n"' file

# 带地址
hexdump -e '"%08_ax: " 2/8 "0x%016x " "\n"' file

# 单个 64-bit 值
hexdump -e '1/8 "0x%016x\n"' file
```

---

## 5. 设备树 (Device Tree) 常用示例

### 5.1 查看 memory 节点的 reg 属性

设备树的 `reg` 属性通常是 `<address size>` 对，使用大端序。

```bash
# 查看原始数据
hexdump -C /proc/device-tree/memory/reg

# 32-bit cells（常见于 32 位系统）
hexdump -e '2/4 "addr: 0x%08x  size: 0x%08x\n"' /proc/device-tree/memory/reg

# 64-bit cells（常见于 64 位系统，#address-cells=2, #size-cells=2）
hexdump -v -e '8/1 "%02x" " " 8/1 "%02x" "\n"' /proc/device-tree/memory/reg | \
  awk '$2 != "0000000000000000" {printf "addr: 0x%s  size: 0x%s\n", $1, $2}'
```

### 5.2 处理大端序（Device Tree 使用大端）

由于 hexdump 按小端读取，查看大端数据需要特殊处理：

```bash
# 方法1：逐字节显示，手动组合
hexdump -e '4/1 "%02x"' /proc/device-tree/memory/reg

# 方法2：使用 xxd（推荐处理大端）
xxd -e -g 4 /proc/device-tree/memory/reg

# 方法3：结合 od 命令
od -A x -t x4 -N 32 /proc/device-tree/memory/reg
```

### 5.3 查看其他设备树属性

```bash
# 查看 compatible 字符串
cat /proc/device-tree/compatible | tr '\0' '\n'

# 查看中断号（通常是 32-bit）
hexdump -e '1/4 "%u\n"' /proc/device-tree/xxx/interrupts

# 查看时钟频率
hexdump -e '1/4 "%u\n"' /proc/device-tree/cpus/cpu@0/clock-frequency
```

---

## 6. 实用技巧

### 6.1 组合多个格式

```bash
# 地址 + 十六进制 + ASCII（类似 -C）
hexdump -e '"%08_ax  " 8/1 "%02x " "  " 8/1 "%02x "' \
        -e '"  |" 16/1 "%_p" "|\n"' file
```

### 6.2 提取特定偏移的值

```bash
# 提取偏移 0x10 处的 32-bit 值
hexdump -s 0x10 -n 4 -e '1/4 "0x%08x\n"' file

# 提取偏移 0x100 处的 64-bit 值
hexdump -s 0x100 -n 8 -e '1/8 "0x%016x\n"' file
```

### 6.3 批量转换为 C 数组

```bash
# 生成 C 语言数组格式
hexdump -e '8/1 "0x%02x, ""\n"' file

# 更完整的格式
echo "uint8_t data[] = {"
hexdump -e '12/1 "0x%02x, " "\n"' file
echo "};"
```

### 6.4 比较两个二进制文件

```bash
diff <(hexdump -C file1) <(hexdump -C file2)
```

### 6.5 查看特定范围

```bash
# 查看 0x100 到 0x1FF（256字节）
hexdump -C -s 0x100 -n 256 file
```

---

## 7. 快速参考卡

| 用途 | 命令 |
|------|------|
| 标准查看 | `hexdump -C file` |
| 查看前 N 字节 | `hexdump -C -n N file` |
| 从偏移开始 | `hexdump -C -s OFFSET file` |
| 8-bit 十六进制 | `hexdump -e '16/1 "%02x " "\n"' file` |
| 16-bit 十六进制 | `hexdump -e '8/2 "%04x " "\n"' file` |
| 32-bit 十六进制 | `hexdump -e '4/4 "%08x " "\n"' file` |
| 64-bit 十六进制 | `hexdump -e '2/8 "%016x " "\n"' file` |
| 带地址 32-bit | `hexdump -e '"%08_ax: " 4/4 "0x%08x " "\n"' file` |
| 设备树 reg | `hexdump -e '4/1 "%02x"' /proc/device-tree/xxx/reg` |
| 单值提取 | `hexdump -s OFF -n 4 -e '1/4 "0x%08x\n"' file` |

---

## 8. 替代工具

| 工具 | 优势 | 安装 |
|------|------|------|
| `xxd` | 支持反向转换、更简洁 | `vim` 自带 |
| `od` | 更好的大端支持 | 系统自带 |
| `hd` | hexdump -C 的别名 | `bsdextrautils` |

```bash
# xxd 常用
xxd file                    # 标准格式
xxd -g 4 file               # 4字节分组
xxd -e -g 4 file            # 4字节大端

# od 常用
od -A x -t x4 file          # 32-bit 十六进制
od -A x -t x1 file          # 8-bit 十六进制
```