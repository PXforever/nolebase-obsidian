---
share: "true"
---

> 我么来简单的分析在编译内核模块时，编译脚本会怎样的执行。

# 前言
当我们执行：
```shell
make modules
```
后，**编译脚本时如何遍历内核所有目录下的程序代码，这次我们主要是弄明白这个**。

# kernel6.6.0/Makefile
> 这次使用的是`kernel 6.6.0`作为示例，来进行分析。

当我们执行`make modules`时，`kernel/Makefile`会执行与`modules`相关的`target`:
```makefile
# ---------------------------------------------------------------------------
# Modules

ifdef CONFIG_MODULES

# By default, build modules as well

all: modules

# 在使用 modversions 构建模块时，我们还需要在降序过程中考虑内置对象，
# 以便在记录校验和之前确保它们是最新的。
ifdef CONFIG_MODVERSIONS
  KBUILD_BUILTIN := 1
endif

# Build modules
#

# *.ko 通常与 vmlinux 无关，但 CONFIG_DEBUG_INFO_BTF_MODULES 是个例外。
ifdef CONFIG_DEBUG_INFO_BTF_MODULES
KBUILD_BUILTIN := 1
modules: vmlinux
endif

modules: modules_prepare

# 准备构建外部模块的目标
modules_prepare: prepare
	$(Q)$(MAKE) $(build)=scripts scripts/module.lds

endif # CONFIG_MODULES

# KBUILD_MODPOST_NOFINAL 可用于跳过模块的最终链接。这完全是为了加快测试编译速度。
modules: modpost
ifneq ($(KBUILD_MODPOST_NOFINAL),1)
	$(Q)$(MAKE) -f $(srctree)/scripts/Makefile.modfinal
endif

PHONY += modpost
modpost: $(if $(single-build),, $(if $(KBUILD_BUILTIN), vmlinux.o)) \
	 $(if $(KBUILD_MODULES), modules_check)
	$(Q)$(MAKE) -f $(srctree)/scripts/Makefile.modpost

modules:
	@:

```
我们可以看到编译`modules`依赖于这么几个：
+ `vmlinux`
+ `modules_prepare`
+ `modpost`
而上面几个还不是依赖链上的终端，具体我们看看下面的依赖关系：
![[笔记/01 附件/Linux-Makefile编译过程-如何递归编译子目录/file-20241108145618414.png|笔记/01 附件/Linux-Makefile编译过程-如何递归编译子目录/file-20241108145618414.png]]
上面以`modules`开始发散的依赖，我们先需要关注那部分会遍历所有的目录，这里在观察上面的部分编译`recipe`，大致可以确定是从`build-dir`这个目标开始进行遍历的。

# 遍历原理
> 在开始描述遍历过程前，我们需要知道如何去调试`makefile`，这里有几个知识点需要知道：
> + `$(info xxxx)`可以在非`recipe`环境下执行。
> + `$(MAKEFILE_LIST)`表示当前的`Makefile`列表，也就是嵌套了多少层
> + `$(MAKECMDGOALS)`表示当前进入该`Makefile`的命令行所需要执行的`target`
> + `make -j1`可以单线程运行，这样`log`可以按逻辑顺序打印

好了，我们可以开始来弄明白是如何执行的了。
首先，`build-dir`执行展开如下：
```makefile
make -f $(srctree)/scripts/Makefile.build obj=. need-builtin=1 need-modorder=1

# 上面的$(build)来自scripts/Kbuild.include
build := -f $(srctree)/scripts/Makefile.build obj

# 上面的build-dir实际上在Makefile中赋值
build-dir   := .
```
好了，那么现在需要进入`$(srctree)/scripts/Makefile.build`，部分关键赋值如下：
```makefile
src := $(obj)
# 接着清空部分变量
obj-y :=
obj-m :=
lib-y :=
lib-m :=
......

# Kbuild.include里面会做查找准备下一阶段的kbuild-file
include $(srctree)/scripts/Kbuild.include
include $(srctree)/scripts/Makefile.compiler
# 引入新的obj内容
include $(kbuild-file)
# 处理引入的各部分obj-m,obj-y,subdir-ym
include $(srctree)/scripts/Makefile.lib

quiet_cmd_cc_o_c = CC $(quiet_modtag)  $@
      cmd_cc_o_c = $(CC) $(c_flags) -c -o $@ $< \
		$(cmd_ld_single_m) \
		$(cmd_objtool)

$(obj)/%.o: $(src)/%.c $(recordmcount_source) FORCE
	$(call if_changed_rule,cc_o_c)
	$(call cmd,force_checksrc)

PHONY += $(subdir-ym)
# 递归执行的关键语法，自己调用自己$(build)暗藏该脚本
$(subdir-ym):
	$(Q)$(MAKE) $(build)=$@ \
	need-builtin=$(if $(filter $@/built-in.a, $(subdir-builtin)),1) \
	need-modorder=$(if $(filter $@/modules.order, $(subdir-modorder)),1) \
	$(filter $@/%, $(single-subdir-goals))
```
在`scripts/Makefile.lib`,它会处理关键的：
```makefile
subdir-ym := $(sort $(subdir-y) $(subdir-m) \
			$(patsubst %/,%, $(filter %/, $(obj-y) $(obj-m))))
```
这里会将`subdir-y`，`subdir-y`，`obj-y`，`obj-m`进行排序，然后赋值给`subdir-ym`。

好了，我们可以大致窥探其中的递归调用逻辑，现在我们需要再次通过调试来实际理解：
![[笔记/01 附件/Linux-Makefile编译过程-如何递归编译子目录/file-20241108164918078.png|笔记/01 附件/Linux-Makefile编译过程-如何递归编译子目录/file-20241108164918078.png]]
我们也可以通过下图来进行理解：
![[笔记/01 附件/Linux-Makefile编译过程-如何递归编译子目录/PixPin_2024-11-08_18-07-39.jpg|笔记/01 附件/Linux-Makefile编译过程-如何递归编译子目录/PixPin_2024-11-08_18-07-39.jpg]]
而其他目录的`kbuild-file`基本都是各个目录下的`Makefile`，因为他们里面有`obj-y`或者`obj-m`的定义。
![[笔记/01 附件/Linux-Makefile编译过程-如何递归编译子目录/file-20241108181336676.png|笔记/01 附件/Linux-Makefile编译过程-如何递归编译子目录/file-20241108181336676.png]]