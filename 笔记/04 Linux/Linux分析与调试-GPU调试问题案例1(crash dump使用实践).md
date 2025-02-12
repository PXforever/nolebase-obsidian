---
share: "true"
---

> 在移植`Kernel-6.6.0`至`RK3588`时遇到`Unable to handle kernel paging request at virtual address xxxxxxx`

# 问题
具体情况如下：
```shell
[    4.602486] Unable to handle kernel paging request at virtual address ffffffc275059000
[    6.489052] Mem abort info:
[    6.491856]   ESR = 0x0000000096000005
[    6.495609]   EC = 0x25: DABT (current EL), IL = 32 bits
[    6.500922]   SET = 0, FnV = 0
[    6.503975]   EA = 0, S1PTW = 0
[    6.507112]   FSC = 0x05: level 1 translation fault
[    6.511993] Data abort info:
[    6.514876]   ISV = 0, ISS = 0x00000005, ISS2 = 0x00000000
[    6.520356]   CM = 0, WnR = 0, TnD = 0, TagAccess = 0
[    6.525402]   GCS = 0, Overlay = 0, DirtyBit = 0, Xs = 0
[    6.530712] swapper pgtable: 4k pages, 39-bit VAs, pgdp=0000000001f2b000
[    6.537404] [ffffffc275059000] pgd=0000000000000000, p4d=0000000000000000, pud=0000000000000000
[    6.546103] Internal error: Oops: 0000000096000005 [#1] SMP
[    6.551677] Modules linked in:
[    6.554735] CPU: 3 PID: 1016 Comm: Xorg Not tainted 6.6.0-g66f5bff17667-dirty #85
[    6.562219] Hardware name: TOPEET RK3588 LP4 LP5 Board (DT)
[    6.567784] pstate: 804000c9 (Nzcv daIF +PAN -UAO -TCO -DIT -SSBS BTYPE=--)
[    6.574739] pc : percpu_counter_add_batch+0x28/0x128
[    6.579712] lr : kbasep_os_process_page_usage_update+0x64/0xd0
[    6.585547] sp : ffffffc084ffba80
[    6.588864] x29: ffffffc084ffba80 x28: ffffff8105f29080 x27: 0000000000000000
[    6.596005] x26: 0000007fe9707c98 x25: 0000000000000000 x24: 0000000000000001
[    6.603137] x23: ffffff810b60f000 x22: ffffffc084fea020 x21: 0000000000000000
[    6.610279] x20: 0000000000000001 x19: ffffff810daada00 x18: 0000000000000030
[    6.617421] x17: 3a5d305b746e756f x16: 633e2d746174735f x15: 7373723e2d6d6d3e
[    6.624562] x14: 2d746e6572727563 x13: 0000000000000000 x12: 3030303030303030
[    6.631701] x11: 0000000000002f84 x10: 000000000000a708 x9 : ffffffc080aa3650
[    6.638841] x8 : ffffff81018ccd30 x7 : fffffffe042d83c0 x6 : 0000000000000000
[    6.645973] x5 : 0000000000008000 x4 : ffffffc275059000 x3 : 0000000000000000
[    6.653105] x2 : 0000000000000020 x1 : 0000000000000001 x0 : ffffff810daadd18
[    6.660244] Call trace:
[    6.662687]  percpu_counter_add_batch+0x28/0x128
[    6.667308]  kbasep_os_process_page_usage_update+0x64/0xd0
[    6.672791]  kbase_mmu_alloc_pgd+0xd8/0x2c0
[    6.676978]  kbase_mmu_init+0x84/0x148
[    6.680732]  kbase_context_mmu_init+0x38/0x90
[    6.685086]  kbase_create_context+0xb4/0x1e0
[    6.689356]  kbase_file_create_kctx+0x70/0x238
[    6.693798]  kbase_ioctl+0x250/0x4270
[    6.697468]  __arm64_sys_ioctl+0x3f4/0xc58
[    6.701571]  invoke_syscall.constprop.0+0x58/0x100
[    6.706361]  do_el0_svc+0xb0/0xd8
[    6.709682]  el0_svc+0x3c/0x160
[    6.712824]  el0t_64_sync_handler+0xc0/0xc8
[    6.717010]  el0t_64_sync+0x1a4/0x1a8
[    6.720683] Code: d53b4235 d50343df d53cd044 f9401003 (b8636893) 
```
# 分析
这里的`ffffffc275059000`我们并没有办法确定是什么地址，因为它大概率是某个变量，在访问该变量时，发现它缺页，不仅缺页，改地址根本就没有分配页(`[ffffffc275059000] pgd=0000000000000000, p4d=0000000000000000, pud=0000000000000000`)。那么我们来看看栈。
从栈的调用情况来看，它是一个`ioctl`的系统调用，那么该程序应该是在用户层进程了`ioctl`，那么可以判断是一个用户的进程因为调用`syscall`，在内核发生了错误。
`kbase_xxx`函数是`maliG610`的代码，最终进入到`percpu_counter_add_batch`发生了错误。我们先看这里：
```shell
Internal error: Oops: 0000000096000005 [#1] SMP
```
这个表示的时寄存器`ESR_EL1`的值，查看[ARM aarch64手册](https://developer.arm.com/documentation/ddi0601/2024-12/AArch64-Registers/ESR-EL1--Exception-Syndrome-Register--EL1-?lang=en)：
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221104908084.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221104908084.png]]
`EC`字段为数据异常，也就是在访问`MMU`时数据异常，更加印证了是访问了不存在的页，同时也可以看这个：
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221105024836.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221105024836.png]]
上面的`FSR`表示一级页表翻译错误。
# 找到问题点
那么，该问题我们首先需要找到触发的位置，通过`add2line`或者`decode_stacktrace.sh`，当然前提是需要我们开启内核调试信息表。
因为最终我们需要使用`KGDB`来找到问题点，这里直接是使用`KGDB`:
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221112044027.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221112044027.png]]
我们`dump`出所有的内存：
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221112256547.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221112256547.png]]
这个地址变了，因为重新启动了，这个可以忽略，只要是相同地址就行。我们从上面发现是该内存放在了`x4`，该寄存器为通用寄存器，一般用来做临时存放的，我们可以使用通过`frame x`来切换不同的帧，找到操作`x4`的代码段。
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221112528044.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221112528044.png]]
在第一个`frame`中就找到了，那么对应的是：
```shell
percpu_counter_add_batch (fbc=fbc@entry=0xffffff8105c81718, amount=amount@entry=1, batch=32)
    at ./arch/arm64/include/asm/percpu.h:46
```
我们需要定位`C`代码与汇编的位置，我们使用`objdump(交叉工具)`：
```shell
# 因为percpu_counter_add_batch源码在lib/percpu_counter.c
../prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-objdump -S lib/percpu_counter.o
```
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221112958163.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221112958163.png]]
我们找到代码：
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221113035483.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221113035483.png]]
指令`mrs     x4, tpidr_el1`，它实际上来自函数`arch/arm64/include/asm/percpu.h`(通过`grep查找`):
```c
static inline unsigned long __kern_my_cpu_offset(void)
{
	unsigned long off;

	/*
	 * We want to allow caching the value, so avoid using volatile and
	 * instead use a fake stack read to hazard against barrier().
	 */
	asm(ALTERNATIVE("mrs %0, tpidr_el1",
			"mrs %0, tpidr_el2",
			ARM64_HAS_VIRT_HOST_EXTN)
		: "=r" (off) :
		"Q" (*(const unsigned long *)current_stack_pointer));

	return off;
}
```
它被内联到`percpu_counter_add_batch`中，这里实际上跳转是比较复杂的，这里不作展开，我们在读取`percpu`值时会利用到寄存器`tpidr_el1`，所以我们需要检查传入的`*fbc->counters`：
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221115337243.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221115337243.png]]
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221115423253.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221115423253.png]]
这里时一个指针，但是指针为`0`，问题在这里，但是这只是表象，我们需要找到为什么它为`0`。
# 找到引发问题的本质
> 上面我们找到问题点，那么现在我们需要找到引发问题的根本在哪

这里需要向上推栈，这里我就不展开，具体调用情是(下面是反方向调用)：
```c
void percpu_counter_add_batch(struct percpu_counter *fbc, s64 amount, s32 batch)
	static inline void percpu_counter_add(struct percpu_counter *fbc, s64 amount)
		static void kbasep_add_mm_counter(struct mm_struct *mm, int member, long value)
```
这里的`kbasep_add_mm_counter`：
```c
static void kbasep_add_mm_counter(struct mm_struct *mm, int member, long value)
{
#if (KERNEL_VERSION(6, 3, 0) <= LINUX_VERSION_CODE)
	percpu_counter_add(&mm->rss_stat[member], value);
#elif (KERNEL_VERSION(4, 19, 0) <= LINUX_VERSION_CODE)
	/* To avoid the build breakage due to an unexported kernel symbol
	 * 'mm_trace_rss_stat' from later kernels, i.e. from V4.19.0 onwards,
	 * we inline here the equivalent of 'add_mm_counter()' from linux
	 * kernel V5.4.0~8.
	 */
	atomic_long_add(value, &mm->rss_stat.count[member]);
#else
	add_mm_counter(mm, member, value);
#endif
}
```
我们可以看到`&mm->rss_stat[member]`被传入到`percpu_counter_add`的`fbc`中。
那么我们来看看谁会设置`mm`或者`rss_stat`。`kbasep_add_mm_counter`的被调用栈如下(反向)：
```c
kbasep_add_mm_counter
	kbasep_os_process_page_usage_update
		kbase_process_page_usage_inc
			kbase_mmu_alloc_pgd
				kbase_mmu_init
					kbase_context_mmu_init
						kbase_create_context
							kbase_file_create_kctx
```
这里寻找的过程就不做赘述，主要的目标就是，不断寻找`mm`或者`rss_stat`的实例化或者赋值的关键代码。
我们寻找到:
```c
struct kbase_context {
......
	struct mm_struct *process_mm;
......	
}
struct mm_struct {
......
	struct mm_rss_stat rss_stat;
......
}
```
`struct kbase_context *kctx`变量在`kbase_file_create_kctx`中还是`NULL`,但是他会在`kbase_create_context`初始化，我们重点看这部分代码：
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221151551976.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221151551976.png]]
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221151635756.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221151635756.png]]
上面红框中的循环调用的是：
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221151701121.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221151701121.png]]
在循环代用中，我们使用正常可用的代码(另外一套)加入断点(或者使用KGDB单步)，监测哪个`err = context_init[i].init(kctx);`：
```c
# 加入的断点调试
printk("[gpu/%s] kctx->process_mm->rss_stat[0]:%p\n", __func__, &kctx->process_mm->rss_stat[0]);
```
发现在`i=1`时，`&kctx->process_mm->rss_stat[0]`数值**不为0**。
**注意**：如果没有正常代码作为对比，则需要审查这部分的所有代码。
对比我们调试的代码，在执行完`kbase_context_common_init`还是`0`那么需要深入挖掘里面的问题。
我们看到如下：
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221152325943.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221152325943.png]]
这里可以清晰看到`kctx->process_mm`实际上是来自`current`。那么这里的问题不再是`GPU`驱动问题了。
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221154634208.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221154634208.png]]
正常的代码，这里是有数据的(对比法)。
`current`是一个宏，表示当前的内核线程，我们使用`info threads`，发现当前的内核线程停在了`Xorg`。
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221152708364.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221152708364.png]]
# 深入挖掘进程与内核线程的问题
> 上面寻找问题可以看到，问题不再是`GPU`驱动问题，我们需要深入到进程的创建过程中，我们知道，在内核中是没有进程这个概念的，只有任务以及PID。

现在，我们需要找到创建`Xorg`进程的内核代码，并且跟踪`current->mm->rss_stat[0])->counters`的数值变化。
这里我们需要知道一些基本的知识，`Xorg`是`X Window`系统的服务器部分，我们切换到旧核心(正常进入系统)，然后使用命令查看它的相关进程：
```shell
pstree
```
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221155305169.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221155305169.png]]
这里的`strace`是我加入了调试，实际上`systemd->lightdm->Xorg`这才是真正的调用树。那么我们查看下`lightdm`服务：
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221155554383.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221155554383.png]]
我们希望找到具体的syscall调用`Xorg`，我们可以修改服务：
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221155737041.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221155737041.png]]
```shell
ExecStart=/usr/bin/strace -f -o /var/log/lightdm_strace.log /usr/sbin/lightdm
```
重新启动后，我们将`lightdm_strace.log`导入到`klog`中查看：
![[笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221160005487.png|笔记/01 附件/Linux分析与调试-GPU调试问题案例1(crash dump使用实践)/file-20241221160005487.png]]
这里使用`faccessat`先验证`"/usr/lib/xorg/Xorg"`是否可以执行，然后再执行`Xorg`命令。
**注意**：在这里，我们需要先知道一个知识点，用户层执行一个命令，通常是先`fork`自己，形成一个子进程，然后在子进程中执行`execve`，读取程序的`elf文件`。
所以，在这里，我们先需要找到`lightdm`进程的创建，接着我们需要找到`Xorg`。在花费一些时间后，我们找到了`lightdm`的`fork`过程：
在我们服务(命令行)执行了`/usr/lib/xorg/Xorg`，首先内核中会进入`fork`：
```c
# kernel\fork.c
#ifdef __ARCH_WANT_SYS_FORK
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_MMU
	struct kernel_clone_args args = {
		.exit_signal = SIGCHLD,
	};

	return kernel_clone(&args);
#else
	/* can not support in nommu mode */
	return -EINVAL;
#endif
}
#endif

pid_t kernel_clone(struct kernel_clone_args *args)
{
......
	p = copy_process(NULL, trace, NUMA_NO_NODE, args); //复制一个进程
	add_latent_entropy();
......
}
```
我们需要找的是关键的复制`struct task_struct`位置。
同时我们也找到了`Xorg`的执行位置：
```c
SYSCALL_DEFINE3(execve,
		const char __user *, filename,
		const char __user *const __user *, argv,
		const char __user *const __user *, envp)
{
	return do_execve(getname(filename), argv, envp);
}
```
我们在`do_execve`加入调试点：
```c
static int do_execve(struct filename *filename,
	const char __user *const __user *__argv,
	const char __user *const __user *__envp)
{
	struct user_arg_ptr argv = { .ptr.native = __argv };
	struct user_arg_ptr envp = { .ptr.native = __envp };
	//printk("[Ktask/%s] kernel_filename:%s\n", __func__, filename->name);
	if( !strcmp( filename->name, "/usr/lib/xorg/Xorg"))
	{
		printk("[Ktask/%s] current->mm->rss_stat->count[0]:%p\n", __func__, ((struct percpu_counter *)&current->mm->rss_stat[0])->counters);
		printk("[Ktask/%s] running /usr/lib/xorg/Xorg ,PID(%d)\n", __func__, task_pid_vnr(current));
	}
	return do_execveat_common(AT_FDCWD, filename, argv, envp, 0);
}
```
同时在`fork`过程也加入调试点，最终确定了在`fork`过程错误点(具体就是通过旧版本/正常本部同时对比)：
```c
copy_process
	dup_task_struct
		arch_dup_task_struct
```
上面并没有问题，问题是在`copy_process`复制完父进程结构体后，进行的`copy_mm`位置：
```c
copy_mm
	dup_mm
		mm_init
```
里面的部分代码被修改，去除了`percpu_counter_init_many`代码(`某个同事修改了`)，导致`mm`被清空后，也不会赋值一个有效值。
最终加入该条代码，`GPU`可正常启动。
# 其他
> 上面的分析内容大家不一定会经常遇到。但是，可以借鉴上面的分析过程，在这个前提是需要**完备的`OS`系统知识，编译工具链使用，精通C语言，KGDB/GDB，ARM架构，系统调用，同时对图像系统(用户层)需要有大概的了解**。
> 所以我们在分析一个复杂问题时，这些并不突出的技能，会帮助我们一步步解决该问题。
