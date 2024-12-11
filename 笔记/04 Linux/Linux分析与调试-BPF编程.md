---
share: "true"
---
# 前言

> 我们在`BPF原理`一文中理解了BPF的使用，现在开始实践。
> 这里使用的是`RK3588，kernel-6.6`

# 配置
编写`bpf`程序前，我们需要开启内核以下支持：
```shell
CONFIG_BPF=y             # 启用 BPF 支持
CONFIG_BPF_SYSCALL=y     # 允许通过 syscall 加载 BPF 程序
CONFIG_HAVE_EBPF_JIT=y   # 启用 eBPF JIT 编译器（提高性能）
CONFIG_BPF_JIT=y         # 启用 eBPF 程序的 JIT 编译支持
```

接着在机器上安装：
```shell
sudo apt install llvm libbpf-dev

# 安装工具bpftrace bpftool（在linux-tools-$(uname -r)中）
sudo apt install bpftrace linux-tools-$(uname -r)
```
如果没有对应的`bpftool`，可以在[[Linux调试与分析-嵌入式交叉编译bpftool工具|内核源码交叉编译]]构建。
# 编程
> `bpf`程序分两部分：
> + 载入内核的代码
> + 用户层代码

我们可以参靠内核源码下的`kernel/sample/bpf/*`
![[笔记/01 附件/Linux分析与调试-BPF编程/image-20240726114140248.png|image-20240726114140248]]

可以看到，里面的代码大部分都是成对出现的，比如：
```shell
test_current_task_under_cgroup_kern.c
test_current_task_under_cgroup_user.c
```
我们也来编写一个这样的程序来测试。
接下来我们会在：
+ `RK3588`
+ `kernel-5.10.160`

# 编程示例
## `i2c_write`
我们先看看内核支持的事件，有这几种方式：

1. 通过`bpftool`
```shell
sudo bpftrace -l
```

2. 通过`debugfs`
```shell
# cd /sys/kernel/debug/tarcing
这里面相应的目录就是对应的事件
```

接着编写如下代码：
```c
//trace_i2c_write.c
#include <linux/bpf.h>
#include <linux/ptrace.h>
#include <bpf/bpf_helpers.h>

SEC("tracepoint/i2c/i2c_write") //通过上面的查看获得的
int trace_i2c_write(struct pt_regs *ctx) {
    bpf_printk("i2c_write called");
    return 0;
}

char _license[] SEC("license") = "GPL";
```

```c
//loader.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <errno.h>

int main() {
    struct bpf_object *obj;
    struct bpf_link *link;
    int prog_fd;

    // Load BPF program
    obj = bpf_object__open_file("trace_i2c_write.o", NULL);
    if (libbpf_get_error(obj)) {
        fprintf(stderr, "Failed to open BPF object file\n");
        return 1;
    }
    fprintf( stdout, "open file OK!\n");

    if (bpf_object__load(obj)) {
        fprintf(stderr, "Failed to load BPF object file\n");
        return 1;
    }
    fprintf( stdout, "load file OK!\n");

    // Get the file descriptor for the BPF program
    prog_fd = bpf_program__fd(bpf_object__find_program_by_name(obj, "trace_i2c_write"));
    if (prog_fd < 0) {
        fprintf(stderr, "Failed to get file descriptor for BPF program\n");
        return 1;
    }
    fprintf( stdout, "program fd OK!\n");

    // Attach to tracepoint
    link = bpf_program__attach_tracepoint(bpf_object__find_program_by_name(obj, "trace_i2c_write"), "i2c", "i2c_write");
    if ( !link) {
        fprintf(stderr, "Failed to attach BPF program to tracepoint: %s\n", strerror(errno));
        return 1;
    }

    printf("BPF program successfully loaded and attached to tracepoint\n");
    while (1) {
        sleep(1);
    }
    
    bpf_link__destroy(link);
    bpf_object__close(obj);

    return 0;
}
```

`Makefile如下：`
```makefile
KER_BPF = trace_i2c_write.c
APP_BPF = loader.c

KER_OBJ = ${patsubst %.c,%.o, ${KER_BPF}} 
APP_OBJ = ${patsubst %.c, %, ${APP_BPF}}

all: ${KER_OBJ} ${APP_OBJ}

${KER_OBJ}: ${KER_BPF}
	clang -O2 -target bpf -c $< -o $@ -I/usr/include/aarch64-linux-gnu

${APP_OBJ}: ${APP_BPF}
	gcc -o $@ $< -lbpf

.PHONY:clean
clean:
	echo "rm ${KER_OBJ} ${APP_OBJ}"
	rm -rf ${KER_OBJ} ${APP_OBJ}
```

在`RK3588`上编译后，执行：
```shell
# 编译
make

# 执行加载到内核
sudo ./loader
# 其中trace_i2c_write.o不需要操作，因为它是被loader加载
```

我们开启另外一个`ssh`页面，使用命令来写`i2c`：
```shell
# 安装i2c工具
apt install i2c-tools

# 测试
i2cset 1  0x11 0x99 44 b
```

**发现没有任何打印信息，猜想可能输出可能不会打印到标准输出**，我们找到辅助函数：`trace_helpers.h`，`trace_helpers.c`。位置在：`SDK_SOURCE/kernel/tools/testing/selftests/bpf`下：
![[笔记/01 附件/Linux分析与调试-BPF编程/image-20240726134805165.png|image-20240726134805165]]

我们复制它俩到工程中，然后修改`Makefile`：
```makefile
APP_TARGET = loader

KER_BPF = trace_i2c_write.c
APP_BPF = loader.c trace_helpers.c 

KER_OBJ = ${patsubst %.c,%.o, ${KER_BPF}} 
APP_OBJ =  ${patsubst %.c, %.o, ${subst ${APP_TARGET}.c, , ${APP_BPF}}}

all: ${KER_OBJ} ${APP_TARGET}

${KER_OBJ}: ${KER_BPF}
	clang -O2 -target bpf -c $< -o $@ -I/usr/include/aarch64-linux-gnu

${APP_TARGET}: ${APP_TARGET}.c ${APP_OBJ}
	gcc -o $@ $? -lbpf -lelf

${APP_OBJ}: %.o: %.c
	gcc -c $? -o $@  -lbpf

.PHONY:clean
clean:
	echo "rm ${KER_OBJ} ${APP_TARGET}"
	rm -rf ${KER_OBJ} ${APP_TARGET} *.o
```

代码修改如下：
```c
//trace_i2c_write.c
#include <linux/bpf.h>
#include <linux/ptrace.h>
#include <bpf/bpf_helpers.h>

SEC("tracepoint/i2c/i2c_write")
int trace_i2c_write(struct pt_regs *ctx) {
    bpf_printk("[bpf_demo]: i2c_write trigger\n");
    return 0;
}

char _license[] SEC("license") = "GPL";

```

```c
//loader.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <errno.h>
#include "trace_helpers.h"

int main() {
    struct bpf_object *obj;
    struct bpf_link *link;
    int prog_fd;

    // Load BPF program
    obj = bpf_object__open_file("trace_i2c_write.o", NULL);
    if (libbpf_get_error(obj)) {
        fprintf(stderr, "Failed to open BPF object file\n");
        return 1;
    }
    fprintf( stdout, "open file OK!\n");

    if (bpf_object__load(obj)) {
        fprintf(stderr, "Failed to load BPF object file\n");
        return 1;
    }
    fprintf( stdout, "load file OK!\n");

    // Get the file descriptor for the BPF program
    prog_fd = bpf_program__fd(bpf_object__find_program_by_name(obj, "trace_i2c_write"));
    if (prog_fd < 0) {
        fprintf(stderr, "Failed to get file descriptor for BPF program\n");
        return 1;
    }
    fprintf( stdout, "program fd OK!\n");

    // Attach to tracepoint
    link = bpf_program__attach_tracepoint(bpf_object__find_program_by_name(obj, "trace_i2c_write"), "i2c", "i2c_write");
    if ( !link) {
        fprintf(stderr, "Failed to attach BPF program to tracepoint: %s\n", strerror(errno));
        return 1;
    }

    printf("BPF program successfully loaded and attached to tracepoint\n");
    
    read_trace_pipe();

    bpf_link__destroy(link);
    bpf_object__close(obj);

    return 0;
}
```
执行后发现打印了很多东西，也很快，我们不需要其他的数据，可以在`read_trace_pipe`中加入字符串比较进行过滤。
![[笔记/01 附件/Linux分析与调试-BPF编程/file-20241116170110317.png|笔记/01 附件/Linux分析与调试-BPF编程/file-20241116170110317.png]]
我们可以通过：
```shell
sudo bpftool prog list 
# 或者
sudo bpftool prog show
```
来查看事件是否插入到系统内核。
## v4l2
```makefile
APP_TARGET = loader

KER_BPF = trace_v4l2_qbuf.c
APP_BPF = loader.c trace_helpers.c 

KER_OBJ = ${patsubst %.c,%.o, ${KER_BPF}} 
APP_OBJ =  ${patsubst %.c, %.o, ${subst ${APP_TARGET}.c, , ${APP_BPF}}}

all: ${KER_OBJ} ${APP_TARGET}

${KER_OBJ}: ${KER_BPF}
	clang -O2 -target bpf -c $< -o $@ -I/usr/include/aarch64-linux-gnu

${APP_TARGET}: ${APP_TARGET}.c ${APP_OBJ}
	gcc -o $@ $? -lbpf -lelf

${APP_OBJ}: %.o: %.c
	gcc -c $? -o $@  -lbpf

.PHONY:clean
clean:
	echo "rm ${KER_OBJ} ${APP_TARGET}"
	rm -rf ${KER_OBJ} ${APP_TARGET} *.o
```

```c
//trace_v4l2_qbuf.c
#include <linux/bpf.h>
#include <linux/ptrace.h>
#include <bpf/bpf_helpers.h>

#define USE_TRACE_PRINT 1

#ifdef USE_TRACE_PRINT
#include <bpf/bpf_tracing.h>
#endif

SEC("tracepoint/v4l2/v4l2_qbuf")
int trace_v4l2_qbuf(struct pt_regs *ctx) {
    unsigned long long ts = bpf_ktime_get_ns();

    unsigned long long seconds = ts / 1000000000;
    unsigned long long nanoseconds = ts % 1000000000;
#ifdef USE_TRACE_PRINT //直接打印出来
    char fmt[] = "[bpf_demo_t]: v4l2_qbuf trigger-%llus.%llu\n";
    bpf_trace_printk( fmt, sizeof(fmt), seconds, nanoseconds);
#else //传输到pipe中
    bpf_printk("[bpf_demo]: v4l2_qbuf trigger - %llu.%09llu\n", seconds, nanoseconds);
#endif
    return 0;
}

char _license[] SEC("license") = "GPL";
```

```c
//loader.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <errno.h>
#include "trace_helpers.h"

int main() {
    struct bpf_object *obj;
    struct bpf_link *link;
    int prog_fd;

    // Load BPF program
    obj = bpf_object__open_file("trace_v4l2_qbuf.o", NULL);
    if (libbpf_get_error(obj)) {
        fprintf(stderr, "Failed to open BPF object file\n");
        return 1;
    }
    fprintf( stdout, "open file OK!\n");

    if (bpf_object__load(obj)) {
        fprintf(stderr, "Failed to load BPF object file\n");
        return 1;
    }
    fprintf( stdout, "load file OK!\n");

    // Get the file descriptor for the BPF program
    prog_fd = bpf_program__fd(bpf_object__find_program_by_name(obj, "trace_v4l2_qbuf"));
    if (prog_fd < 0) {
        fprintf(stderr, "Failed to get file descriptor for BPF program\n");
        return 1;
    }
    fprintf( stdout, "program fd OK!\n");

    // Attach to tracepoint
    link = bpf_program__attach_tracepoint(bpf_object__find_program_by_name(obj, "trace_v4l2_qbuf"), "v4l2", "v4l2_qbuf");
    if ( !link) {
        fprintf(stderr, "Failed to attach BPF program to tracepoint: %s\n", strerror(errno));
        return 1;
    }

    printf("BPF program successfully loaded and attached to tracepoint\n");
 
    while(1)
        sleep(1);
    //read_trace_pipe();

    bpf_link__destroy(link);
    bpf_object__close(obj);

    return 0;
}
```
因为在`i2c`实验中会打印大量无关的内容，我们通过查看节点来过滤：
```shell
sudo cat /sys/kernel/debug/tracing/trace_pipe |grep "bpf_demo"
```

## 输入设备事件捕获(鼠标、键盘)
这里使用的是`kprobe`事件，我们还需要打开：
```shell
# 开启kprobe
CONFIG_KPROBES=y
CONFIG_KPROBE_EVENTS=y
```
代码如下：
```c
//trace_usb_hid_input.c
#include <linux/bpf.h>
#include <linux/ptrace.h>
#include <bpf/bpf_helpers.h>

#define USE_TRACE_PRINT 1

#ifdef USE_TRACE_PRINT
#include <bpf/bpf_tracing.h>
#endif

SEC("kprobe/hid_input_report")
int trace_usb_hid_input(struct pt_regs *ctx) {
    static unsigned long long last_ts = 0;
    unsigned long long ts = bpf_ktime_get_ns();

#ifdef USE_TRACE_PRINT
    unsigned long long delta_ms = 0;
    unsigned long long delta_us = 0;
    char fmt[] = "[bpf_demo_t]: hid_input_report trigger - del=%llums.%lluus.%lluns\n";
    unsigned long long delta_ns = ts - last_ts;
    if( delta_ns > 1000000)
    {
        delta_ms = delta_ns/1000000;
        delta_ns %= 1000000;
    }
    if( delta_ns > 1000)
    {
        delta_us = delta_ns/1000;
        delta_ns %= 1000;
    }
    bpf_trace_printk( fmt, sizeof(fmt), delta_ms, delta_us, delta_ns);
#else
    unsigned long long seconds = ts / 1000000000;
    unsigned long long nanoseconds = ts % 1000000000; //ns
    bpf_printk("[bpf_demo]: hid_input_report trigger - %llu.%09llu\n", seconds, nanoseconds);
#endif
    last_ts = ts;
    return 0;
}

char _license[] SEC("license") = "GPL";

```

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <errno.h>
#include "trace_helpers.h"

int main() {
    struct bpf_object *obj;
    struct bpf_link *link;
    int prog_fd;

    // Load BPF program
    obj = bpf_object__open_file("trace_usb_hid_input.o", NULL);
    if (libbpf_get_error(obj)) {
        fprintf(stderr, "Failed to open BPF object file\n");
        return 1;
    }
    fprintf( stdout, "open file OK!\n");

    if (bpf_object__load(obj)) {
        fprintf(stderr, "Failed to load BPF object file\n");
        return 1;
    }
    fprintf( stdout, "load file OK!\n");

    // Get the file descriptor for the BPF program
    prog_fd = bpf_program__fd(bpf_object__find_program_by_name(obj, "trace_usb_hid_input"));
    if (prog_fd < 0) {
        fprintf(stderr, "Failed to get file descriptor for BPF program\n");
        return 1;
    }
    fprintf( stdout, "program fd OK!\n");

    // Attach to tracepoint
    link = bpf_program__attach_kprobe(bpf_object__find_program_by_name(obj, "trace_usb_hid_input"), false, "hid_input_report");
    if ( !link) {
        fprintf(stderr, "Failed to attach BPF program to tracepoint: %s\n", strerror(errno));
        return 1;
    }

    printf("BPF program successfully loaded and attached to tracepoint\n");
    
    
    while(1)
        sleep(1);
    
    //read_trace_pipe();

    bpf_link__destroy(link);
    bpf_object__close(obj);

    return 0;
}
```

接着编译，启动，然后读取`trace`:
```shell
 cat /sys/kernel/debug/tracing/trace_pipe |grep "hid_input_report"
```
![[笔记/01 附件/Linux分析与调试-BPF编程/image-20240726172411805.png|image-20240726172411805]]
## 获取函数参数
在使用过程中，我们有时候并不只是想简单的获取函数触发情况，还希望能够查看到函数的参数，这里我们举一个更加复杂的例子来展示获取函数参数的用法。
这里选用的是内核中的一个函数来进行跟踪：
![[笔记/01 附件/Linux分析与调试-BPF编程/file-20241118143059702.png|笔记/01 附件/Linux分析与调试-BPF编程/file-20241118143059702.png]]
![[笔记/01 附件/Linux分析与调试-BPF编程/file-20241118143132197.png|笔记/01 附件/Linux分析与调试-BPF编程/file-20241118143132197.png]]

### 准备
在上面的基础上(`BPF`，`Kprobe`配置开启)，我们还需要开启以下配置：
```shell
# 开启调试信息(BPF可读取内核数据结构)
CONFIG_DEBUG_INFO_BTF=y
CONFIG_COMPILE_TEST=y
CONFIG_CLK_RK3288=n       #该配置因为代码原因不得不关闭，实际可能没有这个
CONFIG_DEBUG_INFO_SPLIT=n
CONFIG_DEBUG_INFO_REDUCED=n
CONFIG_DEBUG_INFO_DWARF5=n
```
上面的配置具体不定，因为内核版本不一定一致，我门应该使用`make ARCH=arm64 CROSS_COMPILE=${CROSS_COMPILE-自己的路径} menuconfig`在里面进行确认`CONFIG_DEBUG_INFO_BTF`具体的依赖情况：
![[笔记/01 附件/Linux分析与调试-BPF编程/file-20241118142508514.png|笔记/01 附件/Linux分析与调试-BPF编程/file-20241118142508514.png]]
接着重新编译内核(*时间比以前长了*)，烧录，我们会发现开机后多出了：
![[笔记/01 附件/Linux分析与调试-BPF编程/file-20241118142621818.png|笔记/01 附件/Linux分析与调试-BPF编程/file-20241118142621818.png]]
编译期间可能缺少某些组件，需要安装：
```shell
sudo apt install dwarves
```

### 代码
`eBPF `程序：
```c
//trace_rk3x_i2c_xfer.c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>
#include "trace_rk3x_i2c_xfer.h"  // 包含共享头文件

// 定义环形缓冲区
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 256 * 1024);
} rb SEC(".maps");

SEC("kprobe/rk3x_i2c_xfer")
int BPF_KPROBE(trace_rk3x_i2c_xfer, struct i2c_adapter *adap, struct i2c_msg *msgs, int num)
{
    struct event *e;
    
    // 创建事件
    e = bpf_ringbuf_reserve(&rb, sizeof(*e), 0);
    if (!e) {
        return 0;
    }
    
    // 获取进程信息
    e->pid = bpf_get_current_pid_tgid() >> 32;
    bpf_get_current_comm(&e->comm, sizeof(e->comm));
    
    // 读取适配器号
    e->adapter_nr = BPF_CORE_READ(adap, nr);
    
    // 读取消息数量
    e->msg_count = num;
    
    // 如果有消息，读取第一条消息的信息
    if (num > 0) {
        e->addr = BPF_CORE_READ(msgs, addr);
        e->flags = BPF_CORE_READ(msgs, flags);
        e->len = BPF_CORE_READ(msgs, len);
    } else {
        e->addr = 0;
        e->flags = 0;
        e->len = 0;
    }
    
    // 提交事件
    //bpf_printk("Event submitted: %d\n", e);
    bpf_ringbuf_submit(e, 0); //提交到环形缓冲器，然后用户层的代码可以获取
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

用户空间程序来读取和显示这些数据：
```c
/* trace_rk3x_i2c_xfer_user.c */
#include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <string.h>
#include <errno.h>
#include <sys/resource.h>
#include <bpf/libbpf.h>
#include "trace_rk3x_i2c_xfer.h"        // 共享头文件
#include "trace_rk3x_i2c_xfer.skel.h"   // 自动生成的骨架头文件

static volatile bool exiting = false;

static void sig_handler(int sig)
{
    exiting = true;
}

static int handle_event(void *ctx, void *data, size_t data_sz)
{
    const struct event *e = data;
    
    printf("%-16s %-6d adapter:%d msgs:%d addr:0x%02x flags:0x%02x len:%d\n",
           e->comm, e->pid, e->adapter_nr, e->msg_count, 
           e->addr, e->flags, e->len);
    return 0;
}

/*
* 下面的trace_rk3x_i2c_xfer__open并非标准的函数
* 而是通过bpftool生成的骨架文件，使得用户层可以直接使用
* 简化了代码。
*/
int main(int argc, char **argv)
{
    struct ring_buffer *rb = NULL;
    struct trace_rk3x_i2c_xfer *skel;  // 修改这里：删除 _bpf 后缀
    int err;

    /* 设置 signal handler */
    signal(SIGINT, sig_handler);
    signal(SIGTERM, sig_handler);

    /* Open BPF application */
    skel = trace_rk3x_i2c_xfer__open();  // 修改这里：使用正确的函数名
    if (!skel) {
        fprintf(stderr, "Failed to open BPF skeleton\n");
        return 1;
    }

    /* Load & verify BPF programs */
    err = trace_rk3x_i2c_xfer__load(skel);  // 修改这里：使用正确的函数名
    if (err) {
        fprintf(stderr, "Failed to load and verify BPF skeleton\n");
        goto cleanup;
    }

    /* Attach tracepoint handler */
    err = trace_rk3x_i2c_xfer__attach(skel);  // 修改这里：使用正确的函数名
    if (err) {
        fprintf(stderr, "Failed to attach BPF skeleton\n");
        goto cleanup;
    }

    /* Set up ring buffer polling */
    rb = ring_buffer__new(bpf_map__fd(skel->maps.rb), handle_event, NULL, NULL);
    if (!rb) {
        err = -1;
        fprintf(stderr, "Failed to create ring buffer\n");
        goto cleanup;
    }

    printf("Successfully started! Press Ctrl+C to stop.\n");

    /* Main polling loop */
    while (!exiting) {
        err = ring_buffer__poll(rb, 100 /* timeout, ms */);
        /* Ctrl-C will cause -EINTR */
        if (err == -EINTR) {
            err = 0;
            break;
        }
        if (err < 0) {
            printf("Error polling ring buffer: %d\n", err);
            break;
        }
    }

cleanup:
    ring_buffer__free(rb);
    trace_rk3x_i2c_xfer__destroy(skel);  // 修改这里：使用正确的函数名
    return err < 0 ? 1 : 0;
}
```

编写一个中间头文件：
```c
/* trace_rk3x_i2c_xfer.h */
#ifndef __TRACE_I2C_WRITE_H
#define __TRACE_I2C_WRITE_H

struct event {
    __u32 pid;
    __u32 adapter_nr;
    __u32 msg_count;
    __u16 addr;
    __u16 flags;
    __u16 len;
    char comm[16];
};

#endif /* __TRACE_I2C_WRITE_H */
```

`Makefile`如下：
```c
# 编译器和工具设置
CLANG ?= clang
BPFTOOL ?= bpftool
ARCH = arm64
CROSS_COMPILE = aarch64-linux-gnu-

# 编译标志
CFLAGS = -g -O2
BPF_CFLAGS = -target bpf \
             -D__TARGET_ARCH_arm64 \
             -I/usr/include/$(CROSS_COMPILE:/=-) \
             -I.

# 目标文件
BPF_OBJ = trace_# 编译器和工具设置
CLANG ?= clang
BPFTOOL ?= bpftool
ARCH = arm64
CROSS_COMPILE = aarch64-linux-gnu-

# 编译标志
CFLAGS = -g -O2
BPF_CFLAGS = -target bpf \
             -D__TARGET_ARCH_arm64 \
             -I/usr/include/$(CROSS_COMPILE:/=-) \
             -I.

# 目标文件
BPF_OBJ = trace_rk3x_i2c_xfer.o
USER_BIN = trace_rk3x_i2c_xfer
SKEL_HEAD = trace_rk3x_i2c_xfer.skel.h

all: $(USER_BIN)

$(BPF_OBJ): trace_rk3x_i2c_xfer.c trace_rk3x_i2c_xfer.h
	$(CLANG) $(CFLAGS) $(BPF_CFLAGS) -c $< -o $@

$(SKEL_HEAD): $(BPF_OBJ)
	$(BPFTOOL) gen skeleton $< > $@

$(USER_BIN): trace_rk3x_i2c_xfer_user.c $(SKEL_HEAD) trace_rk3x_i2c_xfer.h
	$(CROSS_COMPILE)gcc $(CFLAGS) -I. $< -lbpf -lelf -o $@

clean:
	rm -f $(BPF_OBJ) $(USER_BIN) $(SKEL_HEAD)

```

编译过程中会出现找不到`vmlinux.h`，该文件主要是：
- `vmlinux.h` 是一个自动生成的头文件，包含了整个 Linux 内核的类型定义和结构体
- 它让 eBPF 程序能够访问内核数据结构，而不需要手动定义或包含大量的内核头文件
- 提供了内核符号的完整类型信息，这对于`eBPF`程序的编译和运行是必需的
我们使用工具生成它：
方法1：
```shell
# 首先交叉编译 bpftool
cd tools/bpf/bpftool
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-

# 使用编译好的 bpftool 生成 vmlinux.h
./bpftool btf dump file ../../vmlinux format c > vmlinux.h
```
方法2：
```shell
# 安装 
pahole sudo apt-get install dwarves

# 使用 pahole 生成 vmlinux.h
pahole -J vmlinux > vmlinux.h
```
上面**方法1**中使用的是交叉编译的方法生成`bpftool`，实际上可以参考[[Linux调试与分析-嵌入式交叉编译bpftool工具|本地编译]]进行安装工具。
我们可以将`vmlinux.h`放到工程目录下，也可以放在`/usr/include`中。

### 测试
我们启动`ebpf`程序：
```shell
# 非root用户使用sudo
./trace_rk3x_i2c_xfer
```
![[笔记/01 附件/Linux分析与调试-BPF编程/file-20241118144108605.png|笔记/01 附件/Linux分析与调试-BPF编程/file-20241118144108605.png]]

接着使用命令触发：
```shell
# 命令1
i2cset 0  0x11 0x99 44 b

# 命令2
i2cset 3  0x13 0x77 44 b
```
![[笔记/01 附件/Linux分析与调试-BPF编程/file-20241118144250685.png|笔记/01 附件/Linux分析与调试-BPF编程/file-20241118144250685.png]]

结果是：
![[笔记/01 附件/Linux分析与调试-BPF编程/file-20241118144340180.png|笔记/01 附件/Linux分析与调试-BPF编程/file-20241118144340180.png]]
我们可以清楚的看到函数的`adapter`序号被打印出来，然后器件地址`0x11,0x13`都被打印出来了。

# 交叉编译
## 问题1
https://lore.kernel.org/bpf/20211021123913.48833-1-pulehui@huawei.com/t/
> bpf程序的交叉编译主要依赖于`rootfs`，所以如果是`buildroot`，请先看看其是否支持`libbpf`。