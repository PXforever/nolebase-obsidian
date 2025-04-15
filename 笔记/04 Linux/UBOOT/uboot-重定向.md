---
share: "true"
---

# 前言



# 准备

这里使用`qemu`进行调试，如果有硬件`jtag`调试那更好。

```shell
# 拉取uboot源码
git clone https://source.denx.de/u-boot/u-boot.git


# 安装交叉编译器
sudo apt install gcc-aarch64-linux-gnu

# 设置调试信息
vim qemu_arm64_defconfig
# 加入
CONFIG_CC_OPTIMIZE_FOR_DEBUG=y

# 编译
make ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- qemu_arm64_defconfig
make ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

更加详细的调试预计编译请参考[[uboot-使用|uboot-使用]]。

这里不使用沙箱(`sandbox`)，而是使用`qemu`。

# 第一段重定向分析

由于`uboot`并不会引入外部函数，所有重定向主要集中在：

+ 函数列表，如：`init_sequence_f`
+ 命令表中的`help`等
+ ....

![[笔记/01 附件/uboot-重定向/image-20250414110956931.png|image-20250414110956931]]

注意：地址相差`0x100000`因为`ghidra`分析时对地址进行了偏移。

![[笔记/01 附件/uboot-重定向/image-20250414111132660.png|image-20250414111132660]]

## 分析

现在我们先进行初步分析：

+ 观察读取重定向段的过程
+ 针对`init_sequence_f`符号的重定向详细过程

如果直接调试，`_start`的地址与`_TEXT_BASE`都是`0x00`，这里可以设置`_TEXT_BASE`，也可以不设置。我们通过一些办法来修改`_start`:

```shell
# 服务端
qemu-system-aarch64 -machine virt -nographic -cpu cortex-a57 -device loader,file=u-boot.bin,addr=0x100000,cpu-num=0 -gdb tcp::3333 -S

# 客户端
gdb-multiarch u-boot
(gdb) target extended-remote :3333
```

![[笔记/01 附件/uboot-重定向/image-20250414153903177.png|image-20250414153903177]]

这里会无法是被符号，我们重新修改符号位置：

```shell
(gdb):
symbol-file 
add-symbol-file u-boot 0x100000
```

![[笔记/01 附件/uboot-重定向/image-20250414154053149.png|image-20250414154053149]]

这样，便可以让`_start`设置为`0x100000`。

![[笔记/01 附件/uboot-重定向/image-20250414154237173.png|image-20250414154237173]]

## 观察读取重定向段的过程

### 汇编分析

这部分会仔细跟踪第一次重定的过程，代码主要以`arm64`架构为主，代码位置为`u-boot/arch/arm/cpu/armv8/start.S`:

```assembly
#if CONFIG_POSITION_INDEPENDENT && !defined(CONFIG_XPL_BUILD)
	/* Verify that we're 4K aligned.  */
	adr	x0, _start
	ands	x0, x0, #0xfff
	b.eq	1f
0:
	/*
	 * FATAL, can't continue.
	 * U-Boot needs to be loaded at a 4K aligned address.
	 *
	 * We use ADRP and ADD to load some symbol addresses during startup.
	 * The ADD uses an absolute (non pc-relative) lo12 relocation
	 * thus requiring 4K alignment.
	 */
	wfi
	b	0b
1:

	/*
	 * Fix .rela.dyn relocations. This allows U-Boot to be loaded to and
	 * executed at a different address than it was linked at.
	 */
pie_fixup:
	adr	x0, _start		/* x0 <- Runtime value of _start */
	ldr	x1, _TEXT_BASE		/* x1 <- Linked value of _start */
	subs	x9, x0, x1		/* x9 <- Run-vs-link offset */
	beq	pie_fixup_done
	adrp    x2, __rel_dyn_start     /* x2 <- Runtime &__rel_dyn_start */
	add     x2, x2, #:lo12:__rel_dyn_start
	adrp    x3, __rel_dyn_end       /* x3 <- Runtime &__rel_dyn_end */
	add     x3, x3, #:lo12:__rel_dyn_end
pie_fix_loop:
	ldp	x0, x1, [x2], #16	/* (x0, x1) <- (Link location, fixup) */
	ldr	x4, [x2], #8		/* x4 <- addend */
	cmp	w1, #1027		/* relative fixup? */
	bne	pie_skip_reloc
	/* relative fix: store addend plus offset at dest location */
	add	x0, x0, x9
	add	x4, x4, x9
	str	x4, [x0]
pie_skip_reloc:
	cmp	x2, x3
	b.lo	pie_fix_loop
pie_fixup_done:
#endif
```

```assembly
pie_fixup:
    adr	x0, _start
```

- 取 `_start` 在 **当前运行时位置的地址**，即“实际被加载到的地址”。
- 存入 `x0`。

------

```assembly
    ldr	x1, _TEXT_BASE
```

- `_TEXT_BASE` 是链接地址时 `_start` 的地址。
- 把链接时的 `_start` 地址存入 `x1`。

> 📌 如果链接地址是 `0x8FF00000`，那么 `_TEXT_BASE = 0x8FF00000`。

------

```assembly
    subs	x9, x0, x1
```

- 计算运行地址和链接地址之间的偏移量（delta）：

  ```assembly
  x9 = x0 - x1 = runtime - linked
  ```

------

```assembly
    beq	pie_fixup_done
```

- 如果 `x9 == 0`，表示运行地址 == 链接地址，不需要重定位，直接跳过。

------

```assembly
    adrp    x2, __rel_dyn_start
    add     x2, x2, #:lo12:__rel_dyn_start
```

- 计算运行时 `.rela.dyn` 表的起始地址 `x2`。
- `adrp` + `add` 是 ARM64 的标准方式来访问大地址常量。

------

```assembly
    adrp    x3, __rel_dyn_end
    add     x3, x3, #:lo12:__rel_dyn_end
```

- 计算 `.rela.dyn` 表的结束地址 `x3`。

---

```assembly
pie_fix_loop:
    ldp	x0, x1, [x2], #16
```

- 从 `x2` 位置读出两个 8 字节的值（16 字节）：
  - `x0` ← 重定位目标地址（link-time 地址）
  - `x1` ← 重定位类型（如 `R_AARCH64_RELATIVE` 是 `1027`）
- `x2 += 16`

------

```assembly
    ldr	x4, [x2], #8
```

- 读取 addend（附加值），存在 `x4`。
- `x2 += 8`

------

```assembly
    cmp	w1, #1027
    bne	pie_skip_reloc
```

- 检查 relocation 类型是否是 `R_AARCH64_RELATIVE`（值为 1027）。
- 如果不是这个类型，就跳过此次重定位。

------

```assembly
    add	x0, x0, x9
    add	x4, x4, x9
    str	x4, [x0]
```

- `x0 += x9`：将重定位目标地址补偿偏移；
- `x4 += x9`：将 addend 补偿偏移；
- 把修正后的值写回新的地址。

------

```assembly
pie_skip_reloc:
    cmp	x2, x3
    blo	pie_fix_loop
```

- 如果还没有到 `.rela.dyn` 结尾，继续循环。

------

```assembly
pie_fixup_done:
    ret
```

- 修复完成，返回。

### 执行过程详细

![[笔记/01 附件/uboot-重定向/image-20250414161155017.png|image-20250414161155017]]

可以看到上图`_start`开始与`0x100000`，计算出运行时与链接时的数值差存入`x9`。

![[笔记/01 附件/uboot-重定向/image-20250414161324899.png|image-20250414161324899]]

`_start`-`_TEXT_BASE` = `0x100000`。

![[笔记/01 附件/uboot-重定向/image-20250414161839341.png|image-20250414161839341]]

当执行到读取`.rela.dyn`表时，该表的地址可以通过：

```shell
cat u-boot.map |grep __rel_dyn_start
```

或者：

```shell
aarch64-linux-gnu-readelf -r u-boot
```

获取，注意到他们的地址并不完全相同，这是因为，符号表示真实的地址：

![[笔记/01 附件/uboot-重定向/image-20250414162102967.png|image-20250414162102967]]

而到执行时，会增加`_start`的偏移量，所以`0x115680+0x100000=0x215680`。

而使用`aarch64-linux-gnu-readelf`读取的显示是`0x125680`，这是因为在读取的时候有一个`LOAD`段影响。

![[笔记/01 附件/uboot-重定向/image-20250414162304426.png|image-20250414162304426]]

![[笔记/01 附件/uboot-重定向/image-20250414162458605.png|image-20250414162458605]]

从上面可以看到符号表中的第一个进行`PIE`计算，各部分信息如下：

```c
offset = 0x0000000013a8
info = R_AARCH64_RELATIV
addend = 0x13a8    
```

根据[[elf加载|elf加载]]中说明的，我们需要进行如下计算：

```c
*offset = *offset + addend
```

![[笔记/01 附件/uboot-重定向/image-20250414164638991.png|image-20250414164638991]]

这里可以看到，`x0`指向的内存为`0x1013a8`，该位置的数值被修改为`0x13a8([x4=0x0]+[addend=0x13a8])`。

## 针对`init_sequence_f`符号的重定向详细过程

该部分需要使用硬件仿真才可以分析，因为使用上面方法会导致卡死。猜测可能是因为修改了`u-boot`启动位置导致的。



# 第二阶段重定位

参考：

https://blog.sina.com.cn/s/blog_4bd13d4801018gxm.html

https://www.cnblogs.com/leaven/p/6296057.html

https://zhuanlan.zhihu.com/p/477194749

https://www.jianshu.com/p/189a4889c7a4

https://doc.embedfire.com/lubancat/build_and_deploy/zh/latest/building_image/boot_image_analyse/boot_image_analyse_down.html

http://47.111.11.73/thread-302931-1-1.html

uboot启动流程参考：

https://www.cnblogs.com/fuzidage/p/17957251

该阶段的重定位主要是将位于`ROM`中的`uboot`代码进行复制到`RAM`中，转而从`RAM`中取指。

以`RK3588`为例：

+ `ROM`：`emmc`
+ `RAM`：`DDR`

![[笔记/01 附件/uboot-重定向/uboot-内存划分(重定位).png|笔记/01 附件/uboot-重定向/uboot-内存划分(重定位).png]]

这里的`HIDE,Secure,MC`是预留的空间，我们通过打印`log`可以看到：

![[笔记/01 附件/uboot-重定向/image-20250415175345808.png|image-20250415175345808]]

从这里信息可以看到`SPL`将`U-BOOT`加载在了`0x200000`位置，而`ATF`在`0x40000`。

![[笔记/01 附件/uboot-重定向/image-20250415175514980.png|image-20250415175514980]]

从这里可以看到第二段重定位代码需要从`0x200000`出复制代码`gd->mon_len`字节到`reserve`区域的`uboot(edc1a000起始)`段。

我们来看看`reserve`代码：

```c
static const init_fnc_t init_sequence_f[] = {
	setup_mon_len,	//计算uboot镜像大小，主要依靠lds来计算
    ......
	announce_dram_init,
	dram_init,		//初始化dram(DDR)
	......
        INIT_FUNC_WATCHDOG_RESET
    /*
     * 现在 DRAM 已经映射好并可以正常工作，我们可以
     * 将 U-Boot 代码迁移（relocate）到 DRAM 中，并从 DRAM 继续运行。
     *
     * 现在我们需要从 RAM 的高地址向下预留一段内存（以下顺序）：
     *  - 一块 U-Boot 和 Linux 都不会使用的保留区域（可选）
     *  - 内核日志缓冲区（log buffer）
     *  - 受保护的 RAM 区域（protected RAM）
     *  - LCD 显存（framebuffer）
     *  - U-Boot 本体（monitor code）
     *  - 板级信息结构体（board info struct）
     */
	setup_dest_addr,	//设置gd->ram_top(ram的最大位置), relocaddr(还没有完全初始化完成，暂时=ram_top), ram_base(ram的初始位置)
#if defined(CONFIG_OF_BOARD_FIXUP) && !defined(CONFIG_OF_INITIAL_DTB_READONLY)
	fix_fdt,
#endif
#ifdef CFG_PRAM
	reserve_pram,
#endif
	reserve_round_4k,	//对齐4K
	setup_relocaddr_from_bloblist,	//预留
	arch_reserve_mmu,	//预留mmu
	reserve_video,
	reserve_trace,		//为trace预留
	reserve_uboot,		//预留重映射位置为uboot，前面的预留都是移动relocaddr，该函数会最终确定relocaddr
	reserve_malloc,
	reserve_board,
	reserve_global_data,
	reserve_fdt,
#if defined(CONFIG_OF_BOARD_FIXUP) && defined(CONFIG_OF_INITIAL_DTB_READONLY)
	reloc_fdt,
	fix_fdt,
#endif
	reserve_bootstage,
	reserve_bloblist,
	reserve_arch,
	reserve_stacks,
	dram_init_banksize,
	show_dram_config,
	INIT_FUNC_WATCHDOG_RESET
	setup_bdinfo,
	display_new_sp,
	INIT_FUNC_WATCHDOG_RESET
#if !defined(CONFIG_OF_BOARD_FIXUP) || !defined(CONFIG_OF_INITIAL_DTB_READONLY)
	reloc_fdt,
#endif
	reloc_bootstage,
	reloc_bloblist,
    setup_reloc,	//设置重定位，主要是设置reloc_off，重定位的偏移
}
```

目前，重定位的空间已经预留完成，各个参数也成功的设置好(`relocaddr`)。

那么接下来开始进入重定向：

```assembly
#if !defined(CONFIG_XPL_BUILD)
/*
 * 设置中间环境（新 sp 和 gd）并调用
 * relocate_code(addr_moni)。这里的技巧是，我们将返回
 * “此处”，但已重新定位。
 */
 	/* 这里x18寄存器储存的是全局变量gd */
	ldr	x0, [x18, #GD_START_ADDR_SP]	/* x0 <- gd->start_addr_sp */
	bic	sp, x0, #0xf	/* 16-byte alignment for ABI compliance */
	ldr	x18, [x18, #GD_NEW_GD]		/* x18 <- gd->new_gd */

	/* 如果标志位为跳过重定位，则直接返回 */
	ldr	x0, [x18, #GD_FLAGS]		/* x0 <- gd->flags */
	tbnz	x0, 11, relocation_return	/* GD_FLG_SKIP_RELOC is bit 11 */

	adr	lr, relocation_return
#if CONFIG_POSITION_INDEPENDENT
	/* Add in link-vs-runtime offset */
	adrp	x0, _start		/* x0 <- Runtime value of _start */
	add	x0, x0, #:lo12:_start
	ldr	x9, _TEXT_BASE		/* x9 <- Linked value of _start */
	sub	x9, x9, x0		/* x9 <- Run-vs-link offset */
	add	lr, lr, x9
#if defined(CONFIG_SYS_RELOC_GD_ENV_ADDR)
	ldr	x0, [x18, #GD_ENV_ADDR]	/* x0 <- gd->env_addr */
	add	x0, x0, x9
	str	x0, [x18, #GD_ENV_ADDR]
#endif
#endif
	/* Add in link-vs-relocation offset */
	ldr	x9, [x18, #GD_RELOC_OFF]	/* x9储存重定位偏移 */
	add	lr, lr, x9	/* 计算重定位后的返回地址，并赋值给lr，使得重定位relocate_code执行完成后能跳转到新的relocation_return上 */
	ldr	x0, [x18, #GD_RELOCADDR]	/* x0储存重定位地址 */
	b	relocate_code
```

接下来看看重定位代码(`arch/arm/lib/relocate_64.S`)：

```assembly
/*
 * void relocate_code(addr_moni)
 *
 * 该函数重新定位监控代码( monitor code)。
 * x0 保存目标地址。
 * 注意：uboot将自生称为monitor code
 */
ENTRY(relocate_code)
	stp	x29, x30, [sp, #-32]!	/* 堆栈，保证函数跳转能返回 */
	mov	x29, sp
	str	x0, [sp, #16]
	/*
	 * 复制uboot从flash到RAM中(实际上RK的话是RAM->RAM)
	 */
	adrp	x1, __image_copy_start		/* x1 <- address bits [31:12] */
	add	x1, x1, :lo12:__image_copy_start/* x1 <- address bits [11:00] */
	subs	x9, x0, x1				/* x9储存复制位置与relocate_code地址的差值 */
	b.eq	relocate_done			/* 如果是0差值，则跳过重定位，已经重定位过了，这个主要是如果使用的软件复位(reset命令)，则会跳过重定位 */
	/*
    * 不要在这里 ldr x1、__image_copy_start，因为如果代码已经
    * 运行在与它链接的地址不同的地址上，该指令
    * 将加载__image_copy_start的重定位值。为了
    * 正确地应用重定位，我们需要知道链接的值。
    *
    * 链接 &__image_copy_start，我们知道它位于
    * CONFIG_TEXT_BASE，它存储在 _TEXT_BASE 中，作为非
    * 重定位值，因为它不是符号引用。
	*/
	ldr	x1, _TEXT_BASE		/* x1 <- Linked &__image_copy_start */
	subs	x9, x0, x1		/* x9 <- Link to copy offset */

	adrp	x1, __image_copy_start		/* x1 <- address bits [31:12] */
	add	x1, x1, :lo12:__image_copy_start/* x1 <- address bits [11:00] */
	adrp	x2, __image_copy_end		/* x2 <- address bits [31:12] */
	add	x2, x2, :lo12:__image_copy_end	/* x2 <- address bits [11:00] */
copy_loop:/*关键复制*/
	ldp	x10, x11, [x1], #16	/* 从[x1]中取出两个数据到x10,x11 */
	stp	x10, x11, [x0], #16	/* 将x10，x11复制到[x0]的地址 */
	cmp	x1, x2			/* 如果x1的地址到了__image_copy_end，则表示代码copy完成 */
	b.lo	copy_loop	/*否则，继续copy*/
	str	x0, [sp, #24]	/* 保存x0地址到sp[24] */

	/*
	 * 修复PIE代码，和第一次重定位一样
	*/
	adrp	x2, __rel_dyn_start		/* x2 <- address bits [31:12] */
	add	x2, x2, :lo12:__rel_dyn_start	/* x2 <- address bits [11:00] */
	adrp	x3, __rel_dyn_end		/* x3 <- address bits [31:12] */
	add	x3, x3, :lo12:__rel_dyn_end	/* x3 <- address bits [11:00] */
fixloop:
	ldp	x0, x1, [x2], #16	/* (x0,x1) <- (SRC location, fixup) */
	ldr	x4, [x2], #8		/* x4 <- addend */
	and	x1, x1, #0xffffffff
	cmp	x1, #R_AARCH64_RELATIVE
	bne	fixnext

	/* relative fix: store addend plus offset at dest location */
	add	x0, x0, x9
	add	x4, x4, x9
	str	x4, [x0]
fixnext:
	cmp	x2, x3
	b.lo	fixloop

/*完成重定位*/
relocate_done:
    switch_el x1, 3f, 2f, 1f    /* 根据当前异常级别(EL3/EL2/EL1)跳转到相应的标签 */
    bl  hang                    /* 如果不在有效的异常级别，分支到hang函数 */
3:  mrs x0, sctlr_el3           /* 从EL3加载系统控制寄存器到x0 */
    b   0f                      /* 分支到标签0 */
2:  mrs x0, sctlr_el2           /* 从EL2加载系统控制寄存器到x0 */
    b   0f                      /* 分支到标签0 */
1:  mrs x0, sctlr_el1           /* 从EL1加载系统控制寄存器到x0 */
0:  tbz w0, #2, 5f              /* 如果位2(C位，数据缓存)被禁用，跳过缓存刷新 */
    tbz w0, #12, 4f             /* 如果位12(I位，指令缓存)被禁用，跳过指令缓存失效 */
    ic  iallu                   /* 指令缓存全部失效到统一点 */
    isb sy                      /* 指令同步屏障，确保指令缓存失效完成 */
4:  ldp x0, x1, [sp, #16]       /* 从栈偏移16的位置加载寄存器对x0和x1 */
    bl  __asm_flush_dcache_range /* 分支到刷新数据缓存范围的函数 */
    bl  __asm_flush_l3_dcache   /* 分支到刷新L3数据缓存的函数 */
5:  ldp x29, x30, [sp],#32      /* 从栈恢复帧指针和链接寄存器，并调整栈指针增加32 */
    ret                         /* 返回调用者 */
ENDPROC(relocate_code)
```

**注意**：这里的如果是`RK3588`的话，它实际上`uboot`已经在`RAM`中了，可以从上面的`log`看到。

在`ret`执行后，则会跳转到`lr`指示的位置，也就是`relocation_return`。

然后代码接着从重定位继续执行。

# 总结

严格来讲，只有第二次才是重定位，因为只有第二次进行了代码`copy`，第一次只是因为`elf`的`segment`机制导致的进行`PIE`。

而重定位有两个好处：

+ `SRAM(类似MCU中的ram，可能128KB)`执行`uboot`代码，更本无法完全容纳`uboot`，所以需要将其转移到外部的`SDRAM(一般是DDR)`中。
+ 将空间整理，为`kernel`的使用，预留出一个整齐，连续的空间。

同时我们也需要注意的是`uboot`并不是完全对内存进行映射，部分高端内存暂时还无法使用，所以我们看到的是明明是`8GB`，但在上面`log`中实际上是`3.75GB(0xf0000000)`。

# 参考

https://developer.arm.com/documentation/100961/1100-00/Programming-Reference/ARMv8-A-Foundation-Platform-memory-map

https://github.com/Xilinx/embeddedsw/blob/master/lib/bsp/standalone/src/arm/ARMv8/64bit/platform/versal/gcc/translation_table.S

https://github.com/qemu/qemu/blob/master/hw/arm/virt.c#L147

https://blog.csdn.net/weixin_45264425/article/details/127469756

https://www.cnblogs.com/tureno/articles/6533780.html
