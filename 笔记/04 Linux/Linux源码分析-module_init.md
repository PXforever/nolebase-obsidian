---
share: "true"
---
> 在编写驱动的时候我们常常会使用`module_init`或者含有该宏的函数或者宏。但是该部分如何工作的我们今天需要探讨清楚。
> 这里使用的是`RK3588-kernel-6.1.75`

这里有几个重要的宏会影响`module_init`展开的结果，我们现在来先确定一下：
```shell
CONFIG_LTO_CLANG=n #是 Linux 内核中用于支持“链接时间优化”（Link Time Optimization，LTO）的配置选项。具体来说，它是一个内核配置宏，用于指示编译内核时是否启用 Clang 编译器的链接时间优化功能。
CONFIG_HAVE_ARCH_PREL32_RELOCATIONS=y #是一个用于 Linux 内核的配置选项，指示特定体系结构（arch）是否支持使用 32 位相对重定位（prel32 relocations）。这通常与二进制代码的地址计算和内存布局有关。
```

# 宏展开实例分析
## 主线分析
接下来我们就开始慢慢来展开一个宏，并且解读其中的内容，这里以`drivers/input/evdev.c`中的：
```c
module_init(evdev_init);
```

接着我们来开始展开第一层：
```c
#define module_init(x)	__initcall(x);
#define __initcall(fn) device_initcall(fn)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)

//当前展开结果为：
___define_initcall( evdev_init, 6, .initcall6)
```

接着展开`___define_initcall`：
```c
#define ___define_initcall(fn, id, __sec)			\
	__unique_initcall(fn, id, __sec, __initcall_id(fn))
// 当前展开结果为：
__unique_initcall( evdev_init, 6, .initcall6, __initcall_id( evdev_init))

// 这里，参考后面的章节可得
__initcall_id( evdev_init) = kmod_evdev__412_1441_evdev_init

// 当前展开结果为：
__unique_initcall( evdev_init, 6, .initcall6, kmod_evdev__412_1441_evdev_init)
```

`__unique_initcall`的定义为：
```c
#define __unique_initcall(fn, id, __sec, __iid)			\
	____define_initcall(fn,					\
		__initcall_stub(fn, __iid, id),			\
		__initcall_name(initcall, __iid, id),		\
		__initcall_section(__sec, __iid))

// 当前展开结果为：
____define_initcall( evdev_init, __initcall_stub( evdev_init, kmod_evdev__412_1441_evdev_init, 6), __initcall_name( initcall, kmod_evdev__412_1441_evdev_init, 6), __initcall_section( .initcall6, kmod_evdev__412_1441_evdev_init))
```
上面的`__unique_initcall`有4个参数，其中`__initcall_stub(fn, __iid, id)` 可展开为：
```c
#define __initcall_stub(fn, __iid, id)	fn

// __initcall_stub展开结果为：
evdev_init
```
而`__initcall_name`展开为：
```c
/* Format: __<prefix>__<iid><id> */
#define __initcall_name(prefix, __iid, id)			\
	__PASTE(__,						\
	__PASTE(prefix,						\
	__PASTE(__,						\
	__PASTE(__iid, id))))

// __initcall_name展开结果为：
__initcall__kmod_evdev__412_1441_evdev_init6
```
接着就是`__initcall_section`，它被展开为：
```c
#define __initcall_section(__sec, __iid)			\
	#__sec ".init"

// __initcall_section展开结果为：
".initcall6.init"
```
合并到主宏上去就是：
```c
// 当前展开结果为：
____define_initcall( evdev_init, evdev_init, __initcall__kmod_evdev__412_1441_evdev_init6, ".initcall6.init")
```

我们继续看下去：
```c
#define ____define_initcall(fn, __stub, __name, __sec)		\
	__define_initcall_stub(__stub, fn)			\
	asm(".section	\"" __sec "\", \"a\"		\n"	\
	    __stringify(__name) ":			\n"	\
	    ".long	" __stringify(__stub) " - .	\n"	\
	    ".previous					\n");

// 当前展开结果为：
__define_initcall_stub(evdev_init, evdev_init) \
asm(".section	\".initcall6.init\", \"a\"		\n"	\
	__stringify(__initcall__kmod_evdev__412_1441_evdev_init6) ":			\n"	\
	".long	" __stringify(evdev_init) " - .	\n"	\
	".previous					\n");
```
这里需要分两步分来解:
+ `__define_initcall_stub`
+ `asm`
先处理`__define_initcall_stub`：
```c
#define __define_initcall_stub(__stub, fn)			\
	__ADDRESSABLE(fn)

#define __ADDRESSABLE(sym) \
	static void * __section(".discard.addressable") __used \
		__UNIQUE_ID(__PASTE(__addressable_,sym)) = (void *)&sym;

// __define_initcall_stub展开结果为：
static void * __attribute__((__section__(section)))(".discard.addressable") __used 
	__UNIQUE_ID(__PASTE(__addressable_, evdev_init)) = (void *)&evdev_init;

// 这里的__UNIQUE_ID(有不同的定义，我们去其中一个代替)
#define __UNIQUE_ID(prefix) __PASTE(__PASTE(__UNIQUE_ID_, prefix), __COUNTER__)

// __define_initcall_stub展开结果为：
static void * __attribute__((__section__(section)))(".discard.addressable") __used 
__UNIQUE_ID___addressable_evdev_init_412 = (void *)&evdev_init;
```

接着处理内嵌汇编代码部分，汇编代码为：
```assembly
.section ".initcall6.init", "a"
__initcall__kmod_evdev__412_1441_evdev_init6:
.long evdev_init - . 
.previous
```
这里我们参考[GUN汇编代码](http://tigcc.ticalc.org/doc/gnuasm.html#SEC98)手册，可以大致分析：
+ `.section ".initcall6.init", "a"`：为`.initcall6.init`段追加内容。
+ `.long`：指令用于在段中插入一个长整型数（通常为4字节）。
+ `evdev_init - .`：这里 `evdev_init` 是一个初始化函数的名称，`- .` 表示计算 `evdev_init` 到当前地址（`.`）之间的偏移量。这个偏移量是用来在运行时找到目标函数的。这种方式允许内核在初始化过程中动态查找和调用对应的函数。
+ **`.previous`**：指令结束当前段的定义，返回到之前的段。这在汇编代码中用于切换回先前的段，确保后续代码不会影响当前定义的段。
所以就是在`.initcall6.init`定义了一个`label(__initcall__kmod_evdev__412_1441_evdev_init6)`，该`label`储存着初始化函数`evdev_init`的偏移量。

**那么最终的展开结果是：**
```c
static void * __attribute__((__section__( ".discard.addressable"))) __used 
__UNIQUE_ID___addressable_evdev_init_412 = (void *)&evdev_init;
asm://这里汇编就不以内嵌形式展示
.section ".initcall6.init", "a"
__initcall__kmod_evdev__412_1441_evdev_init6:
.long evdev_init - . 
.previous
```
这里`__attribute__((__section__))`是`GCC`的功能，表示该声明放入指定`section（段）`中。
`__used`防止未显示的调用而导致的报错。

## `__initcall_id`
这里我们需要单独展开`__initcall_id`，该宏函数主要提供唯一的ID号：
```c
/* Format: <modname>__<counter>_<line>_<fn> */
#define __initcall_id(fn)					\
	__PASTE(__KBUILD_MODNAME,				\
	__PASTE(__,						\
	__PASTE(__COUNTER__,					\
	__PASTE(_,						\
	__PASTE(__LINE__,					\
	__PASTE(_, fn))))))

#define ___PASTE(a,b) a##b
#define __PASTE(a,b) ___PASTE(a,b)
```
这里的`__PASTE`是一个拼接函数，主要是将`a`和`b`拼接起来。这里有几个重要的定义与内建变量：
+ `__KBUILD_MODNAME__`：它是一个宏定义，是由`makfile`控制的，在编译`evdev.c`过程中会自动判断该模块的名称并设置，我们可以在`evdev.c`目录下找到隐藏文件`.evdev.o.cmd`，里面有该宏定义。
  ![[笔记/01 附件/Linux源码分析-module_init/file-20241026230538013.png|笔记/01 附件/Linux源码分析-module_init/file-20241026230538013.png]]
+ `__COUNTER__`：它是一个`GCC`内建的变量，但是GCC手册并没有说明，我们参考[网络](https://blog.csdn.net/qq_36428903/article/details/132410271)上的解释大致可以理解该变量使用一次就自增一次。
那么该宏在这里的展开为：
```c
kmod_evdev__412_1441_evdev_init
```
当热，`__COUNTER__`是无法预测展开的，我们从
![[笔记/01 附件/Linux源码分析-module_init/file-20241026231113682.png|笔记/01 附件/Linux源码分析-module_init/file-20241026231113682.png]]
上面这些文件获取(实际上上图是`__initcall_name`的，而`__initcall_id`是其中部分)。

# 总结
`module_init`主要是：
+ 声明一个为整个工程唯一的函数名`__UNIQUE_ID___addressable_evdev_init_412`，它是指向`evdev_init`初始化函数。
+ 将该函数指针放入到段`.discard.addressable`中
+ 将`evdev_init`的偏移位置写入到段(`section`)中的`__initcall__kmod_evdev__412_1441_evdev_init6`符号中。