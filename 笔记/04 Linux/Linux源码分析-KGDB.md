---
share: "true"
---
> 前面我们学习了如何使用`KGDB`工具让内核进入调试状态，这样哦我们就可以进行硬件级的调试，这会比`printk`获取到更多的运行信息，并且无需重复插入打印点，并且烧录，大大提高了效率。
> 为了更加深入理解`KGDB`，我们在此进行分析。
> 这里使用的是RK3588内核源码，内核版本为：`6.1.75`
# 源码结构
我们先从几个关键的`CONFIG`开始：
```shell
CONFIG_DEBUG_KERNEL=y 
CONFIG_DEBUG_INFO=y
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y
```
`CONFIG_KGDB`管理着如下代码的编译：
```shell
arch/arm64/kernel/Makefile:59:obj-$(CONFIG_KGDB)                        += kgdb.o
kernel/Makefile:90:obj-$(CONFIG_KGDB) += debug/
kernel/debug/Makefile:6:obj-$(CONFIG_KGDB) += debug_core.o gdbstub.o
```
**注意**：我们常常会遇到另外一个选项`CONFIG_KGDB_KDB`，这两个是由区别的，我们接下来一个个讲解。

# 代码分析
## KGDB的启动
我们找到代码：`arch/arm64/kernel/kgdb.c`，和文件夹：`kernel/debug`。先看`kgdb.c:
```c
/*
 * kgdb_arch_init - Perform any architecture specific initialization.
 * This function will handle the initialization of any architecture
 * specific callbacks.
 */
int kgdb_arch_init(void)
{
	int ret = register_die_notifier(&kgdb_notifier);

	if (ret != 0)
		return ret;

	register_kernel_break_hook(&kgdb_brkpt_hook);
	register_kernel_break_hook(&kgdb_compiled_brkpt_hook);
	register_kernel_step_hook(&kgdb_step_hook);
	return 0;
}
```
这是一个`kgdb`架构上的初始化。但是该函数并不是在这里并调用的，而是在`kernel/debug/debug_core.c`
```c
static void kgdb_register_callbacks(void)
{
	if (!kgdb_io_module_registered) {
		kgdb_io_module_registered = 1;
		kgdb_arch_init();
		if (!dbg_is_early)
			kgdb_arch_late();
		register_module_notifier(&dbg_module_load_nb);
		register_reboot_notifier(&dbg_reboot_notifier);
#ifdef CONFIG_MAGIC_SYSRQ
		register_sysrq_key('g', &sysrq_dbg_op);
#endif
		if (kgdb_use_con && !kgdb_con_registered) {
			register_console(&kgdbcons);
			kgdb_con_registered = 1;
		}
	}
}
```
而上面的调用层次：
```c
kgdb_register_io_module
	kgdb_register_callbacks
		kgdb_arch_init
```
`kgdb_register_io_module`有如下几个地方调用：
```shell
drivers/misc/kgdbts.c:1091:     err = kgdb_register_io_module(&kgdbts_io_ops);
drivers/tty/serial/kgdboc.c:209:        err = kgdb_register_io_module(&kgdboc_io_ops);
drivers/tty/serial/kgdboc.c:563:        if (kgdb_register_io_module(&kgdboc_earlycon_io_ops) != 0) {
drivers/usb/early/ehci-dbgp.c:1055:     kgdb_register_io_module(&kgdbdbgp_io_ops);
```

`kgdbts`是`KGDB_TEST`，这里先不讨论，我们看看`tty（drivers/tty/serial/kgdboc.c）`这里：
```C
// 注意需要开启CONFIG_KGDB_SERIAL_CONSOLE才会编译该代码
/*
早期参数解析时，earlycon 和 kgdboc_earlycon 都会被初始化。我们无法保证 earlycon 会首先被初始化，而且在 ACPI 系统中，earlycon 可能会推迟自己的初始化（通常是在 setup_arch() 中）。为了应对这两种情况，我们可以将自己的初始化推迟到启动的稍后阶段。
*/
static int __init kgdboc_earlycon_init(char *opt)
{
......
	kgdboc_earlycon_io_ops.cons = con;
	pr_info("Going to register kgdb with earlycon '%s'\n", con->name);
	if (kgdb_register_io_module(&kgdboc_earlycon_io_ops) != 0) {
		kgdboc_earlycon_io_ops.cons = NULL;
		pr_info("Failed to register kgdb with earlycon\n");
	} else {
		/* Trap exit so we can keep earlycon longer if needed. */
		earlycon_orig_exit = con->exit;
		con->exit = kgdboc_earlycon_deferred_exit;
	}
......
}
early_param("kgdboc_earlycon", kgdboc_earlycon_init);

/*
 * 这仅适用于早期控制台的后期采用。
 *
 * 这不是采用常规控制台的可靠方法，因为我们无法控制控制台初始调用的顺序，而且在任何情况下，许多常规控制台在启动过程中的注册时间都远远晚于控制台的注册时间。
 * 常规控制台在启动过程中的注册时间远远晚于
 * 控制台初始调用！
*/
static int __init kgdboc_earlycon_late_init(void)
{
	if (kgdboc_earlycon_late_enable)
		kgdboc_earlycon_init(kgdboc_earlycon_param);
	return 0;
}
console_initcall(kgdboc_earlycon_late_init);
```
这是一个地方调用，启动`KGDB初始化`则是被[[Linux源码分析-early_param|early_param]]和`console_initcall`控制。**前者**可以通过在`command line`中添加：
```shell
"kgdboc_earlycon=ttyS0,115200"
```
来启动，而添加`command line`，我们又可以在设备树中的`chosen`或者`uboot`下修改`bootargs`来加入。或者在`xxx_deconfig`中使用：
```c
CONFIG_CMDLINE="kgdboc_earlycon=ttyS0,115200"
```

而**后者**则是......
 另一个初始化`kgdb`的地方在`configure_kgdboc`：
```c
static int configure_kgdboc(void)
{
	......
	do_register:
		err = kgdb_register_io_module(&kgdboc_io_ops);
		if (err)
			goto noconfig;
	......
}

static int kgdboc_probe(struct platform_device *pdev)
{
	......
	if (configured != 1) {
		ret = configure_kgdboc();

		/* Convert "no device" to "defer" so we'll keep trying */
		if (ret == -ENODEV)
			ret = -EPROBE_DEFER;
	}
	......
}

static struct platform_driver kgdboc_platform_driver = {
	.probe = kgdboc_probe,
	.driver = {
		.name = "kgdboc",
		.suppress_bind_attrs = true,
	},
};

static int __init init_kgdboc(void)
{
	ret = platform_driver_register(&kgdboc_platform_driver);
	if (ret)
		return ret;
}

module_init(init_kgdboc);
```
所以以一共有三个地方启动`KGDB`：
1. `command line`
2. `console_initcall`
3. `加载模块kgdboc`
## KGDB开机的启动
> 在另一个文档中，我们已经讲解了如何在系统启动后，通过`Magic Key`进入`KGDB`，而现在需要探究如何在开机时就进入`KGDB`。

我们在`kernel/debug_core.c`：
```c
// kernel/debug_core.c
static int __init opt_kgdb_wait(char *str)
{
	kgdb_break_asap = 1;

	kdb_init(KDB_INIT_EARLY);
	if (kgdb_io_module_registered &&
	    IS_ENABLED(CONFIG_ARCH_HAS_EARLY_DEBUG))
		kgdb_initial_breakpoint();

	return 0;
}

early_param("kgdbwait", opt_kgdb_wait);

static void kgdb_initial_breakpoint(void)
{
	kgdb_break_asap = 0;

	pr_crit("Waiting for connection from remote gdb...\n");
	kgdb_breakpoint();
}

/**
 * kgdb_breakpoint - generate breakpoint exception
 *
 * This function will generate a breakpoint exception.  It is used at the
 * beginning of a program to sync up with a debugger and can be used
 * otherwise as a quick means to stop program execution and "break" into
 * the debugger.
 */
noinline void kgdb_breakpoint(void) //该函数就是进入调试模式的函数
{
	atomic_inc(&kgdb_setting_breakpoint);
	wmb(); /* Sync point before breakpoint */
	arch_kgdb_breakpoint();
	wmb(); /* Sync point after breakpoint */
	atomic_dec(&kgdb_setting_breakpoint);
}
```
我们发现开机时就进入`KGDB`需要两个条件:
1. `kgdb_io_module`注册成功
2. `CONFIG_ARCH_HAS_EARLY_DEBUG`需要开启
不幸的是，我找到如下：
![[Linux源码分析-KGDB/file-20241025142030150.png]]
同时在`arch/arm64/Kconfig`并没有找到`ARCH_HAS_EARLY_DEBUG`：
![[Linux源码分析-KGDB/file-20241025142115306.png]]
还有在`lib/Kconfig.kgdb`:
![[Linux源码分析-KGDB/file-20241025142831346.png]]
所以在`RK3588(ARM64)`上，我们并不能在开机时立马进行调试。

幸运的是，我们还可以在**后段**进入调试，具体是在进行`opt_kgdb_wait`失败后，我们可以通过如下的函数栈再次进入调试：
```c
start_kernel
	dbg_late_init
		kgdb_initial_breakpoint
```
在`dbg_late_init`中：
```c
void __init dbg_late_init(void)
{
	dbg_is_early = false;
	if (kgdb_io_module_registered)
		kgdb_arch_late();
	kdb_init(KDB_INIT_FULL);

	if (kgdb_io_module_registered && kgdb_break_asap)
		kgdb_initial_breakpoint();
}
```
这里需求`kgdb_io_module_registered == 1`，而`driver/tty/serial/kgdboc.c`中有：`early_param("kgdboc_earlycon", kgdboc_earlycon_init);`已经执行过1次`kgdboc_earlycon_init`。所以第二次执行是在`consol`终端注册完成后在`kgdboc_earlycon_late_init`执行。我们来再次看看`kgdboc_earlycon_init`：
```c
// driver/tty/serial/kgdboc.c
static int __init kgdboc_earlycon_init(char *opt)
{
	struct console *con;

	kdb_init(KDB_INIT_EARLY);
	
	console_lock();
	//遍历出一个终端，在early_param("kgdboc_earlycon", kgdboc_earlycon_init);执行的那次并不能找到终端，因为还未注册
	//第二次的执行该函数便可以找到终端，实测终端名为:uart
	for_each_console(con) {
		if (con->write && con->read &&
		    (con->flags & (CON_BOOT | CON_ENABLED)) &&
		    (!opt || !opt[0] || strcmp(con->name, opt) == 0))
			break;
	}
	.......
}
```
所以我们需要在`uboot`或者设备树中写下：
```shell
"kgdboc_earlycon=uart kgdbwait"
```
才能在开机时就启动`KGDB`。
**注意**：网络上大都是使用`kgdboc=ttyFIQ0,115200 kgdbwait`，这可能时旧版本的写法，我们需要根据实际情况来编写启动参数。