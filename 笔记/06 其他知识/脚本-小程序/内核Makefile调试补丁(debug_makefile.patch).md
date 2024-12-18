---
share: "true"
---
> 这是一个补丁文件，我们使用`git apply`将其加入到我们的内核中，它会在关键位置打入调试信息，使得我们可以研究内核编译过程

```shell
#-----信息----------
# 该文件使用来调试内核的Makefile过程的补丁文件
# 我们使用命令: git apply <当前文件名>		即可应用补丁
# 如果是非git目录：patch -p1 < <当前文件名>
# 应用该补丁的目录应该是：kernel目录下
# 内核版本推荐：6.6.29

diff --git a/Makefile b/Makefile
index bb1035057..0d61c9289 100644
--- a/Makefile
+++ b/Makefile
@@ -231,6 +231,7 @@ $(filter-out $(this-makefile), $(MAKECMDGOALS)) __all: __sub-make
 
 # Invoke a second make in the output directory, passing relevant variables
 __sub-make:
+	@echo "targrt: $@"
 	$(Q)$(MAKE) $(no-print-directory) -C $(abs_objtree) \
 	-f $(abs_srctree)/Makefile $(MAKECMDGOALS)
 
@@ -363,6 +364,7 @@ __build_one_by_one:
 
 else # !mixed-build
 
+$(info >>>>>>> 第一次 $(srctree)/scripts/Kbuild.include,当前：$(MAKECMDGOALS))
 include $(srctree)/scripts/Kbuild.include
 
 # Read KERNELRELEASE from include/config/kernel.release (if it exists)
@@ -630,6 +632,7 @@ export RCS_TAR_IGNORE := --exclude SCCS --exclude BitKeeper --exclude .svn \
 # Basic helpers built in scripts/basic/
 PHONY += scripts_basic
 scripts_basic:
+	@echo "Target: scripts_basic"
 	$(Q)$(MAKE) $(build)=scripts/basic
 
 PHONY += outputmakefile
@@ -795,6 +798,7 @@ $(KCONFIG_CONFIG):
 # so you cannot notice that Kconfig is waiting for the user input.
 %/config/auto.conf %/config/auto.conf.cmd %/generated/autoconf.h %/generated/rustc_cfg: $(KCONFIG_CONFIG)
 	$(Q)$(kecho) "  SYNC    $@"
+	@echo "再次进入$(srctree)/Makefile, 目标:syncconfig"
 	$(Q)$(MAKE) -f $(srctree)/Makefile syncconfig
 else # !may-sync-config
 # External modules and some install targets need include/generated/autoconf.h
@@ -1198,6 +1202,7 @@ archprepare: outputmakefile archheaders archscripts scripts include/config/kerne
 	include/generated/compile.h include/generated/autoconf.h remove-stale-files
 
 prepare0: archprepare
+	@echo "running parpare0"
 	$(Q)$(MAKE) $(build)=scripts/mod
 	$(Q)$(MAKE) $(build)=. prepare
 
@@ -1381,13 +1386,16 @@ endif
 ifneq ($(dtstree),)
 
 %.dtb: dtbs_prepare
+	@echo "Target=%.dtb"
 	$(Q)$(MAKE) $(build)=$(dtstree) $(dtstree)/$@
 
 %.dtbo: dtbs_prepare
+	@echo "Target=%.dtbo"
 	$(Q)$(MAKE) $(build)=$(dtstree) $(dtstree)/$@
 
 PHONY += dtbs dtbs_prepare dtbs_install dtbs_check
 dtbs: dtbs_prepare
+	@echo "Target=%.dtbs"
 	$(Q)$(MAKE) $(build)=$(dtstree)
 
 # include/config/kernel.release is actually needed when installing DTBs because
@@ -1416,6 +1424,7 @@ endif
 
 PHONY += scripts_dtc
 scripts_dtc: scripts_basic
+	@echo "target: $@"
 	$(Q)$(MAKE) $(build)=scripts/dtc
 
 ifneq ($(filter dt_binding_check, $(MAKECMDGOALS)),)
@@ -1424,10 +1433,12 @@ endif
 
 PHONY += dt_binding_check
 dt_binding_check: scripts_dtc
+	@echo "target: $@"
 	$(Q)$(MAKE) $(build)=Documentation/devicetree/bindings
 
 PHONY += dt_compatible_check
 dt_compatible_check: dt_binding_check
+	@echo "target: $@"
 	$(Q)$(MAKE) $(build)=Documentation/devicetree/bindings $@
 
 # ---------------------------------------------------------------------------
@@ -1460,6 +1471,7 @@ modules: modules_prepare
 
 # Target to prepare building external modules
 modules_prepare: prepare
+	@echo "Target: modules_prepare"
 	$(Q)$(MAKE) $(build)=scripts scripts/module.lds
 
 endif # CONFIG_MODULES
@@ -1909,7 +1921,10 @@ endif
 # make menuconfig etc.
 # Error messages still appears in the original language
 PHONY += $(build-dir)
+# 关键的第一步启动编译内核
+$(info !!! build_dir=$(build-dir))
 $(build-dir): prepare
+	@echo "Target: build-dir"
 	$(Q)$(MAKE) $(build)=$@ need-builtin=1 need-modorder=1 $(single-goals)
 
 clean-dirs := $(addprefix _clean_, $(clean-dirs))
diff --git a/scripts/Kbuild.include b/scripts/Kbuild.include
index 7778cc97a..984391b31 100644
--- a/scripts/Kbuild.include
+++ b/scripts/Kbuild.include
@@ -2,6 +2,8 @@
 ####
 # kbuild: Generic definitions
 
+$(info $(shell echo "  >>> Kbuild.include"))
+
 # Convenient variables
 comma   := ,
 quote   := "
@@ -64,6 +66,9 @@ stringify = $(squote)$(quote)$1$(quote)$(squote)
 # The path to Kbuild or Makefile. Kbuild has precedence over Makefile.
 kbuild-dir = $(if $(filter /%,$(src)),$(src),$(srctree)/$(src))
 kbuild-file = $(or $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Makefile)
+$(info $(shell echo "    src=$(src), srctree=$(srctree)"))
+$(info $(shell echo "    kbuild-dir is $(kbuild-dir)"))
+$(info $(shell echo "    kbuild-file is $(kbuild-file)"))
 
 ###
 # Read a file, replacing newlines with spaces
diff --git a/scripts/Makefile.build b/scripts/Makefile.build
index 82e3fb19f..df40b38f5 100644
--- a/scripts/Makefile.build
+++ b/scripts/Makefile.build
@@ -2,6 +2,7 @@
 # ==========================================================================
 # Building
 # ==========================================================================
+$(info >>> Makefile.build| obj=$(obj), taget=$(MAKECMDGOALS))
 
 src := $(obj)
 
@@ -38,7 +39,8 @@ subdir-ccflags-y :=
 
 include $(srctree)/scripts/Kbuild.include
 include $(srctree)/scripts/Makefile.compiler
-include $(kbuild-file)
+$(info $(shell echo "  进入 $(kbuild-file)"))
+include $(kbuild-file) #引入新的obj内容
 include $(srctree)/scripts/Makefile.lib
 
 # Do not include hostprogs rules unless needed.
@@ -240,6 +242,7 @@ endef
 
 # Built-in and composite module parts
 $(obj)/%.o: $(src)/%.c $(recordmcount_source) FORCE
+	@echo "  编译:$<"
 	$(call if_changed_rule,cc_o_c)
 	$(call cmd,force_checksrc)
 
@@ -476,7 +479,9 @@ $(single-subdir-goals): $(single-subdirs)
 # ---------------------------------------------------------------------------
 
 PHONY += $(subdir-ym)
+# 递归执行的关键语法，自己调用自己
 $(subdir-ym):
+	@echo "子目录=$@"
 	$(Q)$(MAKE) $(build)=$@ \
 	need-builtin=$(if $(filter $@/built-in.a, $(subdir-builtin)),1) \
 	need-modorder=$(if $(filter $@/modules.order, $(subdir-modorder)),1) \
diff --git a/scripts/Makefile.lib b/scripts/Makefile.lib
index 68d0134bd..574cadc7c 100644
--- a/scripts/Makefile.lib
+++ b/scripts/Makefile.lib
@@ -1,5 +1,8 @@
 # SPDX-License-Identifier: GPL-2.0
 # Backward compatibility
+
+$(info $(shell echo "  >>> Makefile.lib"))
+
 asflags-y  += $(EXTRA_AFLAGS)
 ccflags-y  += $(EXTRA_CFLAGS)
 cppflags-y += $(EXTRA_CPPFLAGS)
@@ -22,6 +25,10 @@ obj-m := $(filter-out $(obj-y),$(obj-m))
 lib-y := $(filter-out $(obj-y), $(sort $(lib-y) $(lib-m)))
 
 # Subdirectories we need to descend into
+$(info $(shell echo "    obj-y: $(obj-y)"))
+$(info $(shell echo "    obj-m: $(obj-m)"))
+$(info $(shell echo "    subdir-y: $(subdir-y)"))
+$(info $(shell echo "    subdir-m: $(subdir-m)"))
 subdir-ym := $(sort $(subdir-y) $(subdir-m) \
 			$(patsubst %/,%, $(filter %/, $(obj-y) $(obj-m))))
 
diff --git a/scripts/Makefile.modfinal b/scripts/Makefile.modfinal
index b3a6aa8fb..e38cc6954 100644
--- a/scripts/Makefile.modfinal
+++ b/scripts/Makefile.modfinal
@@ -2,7 +2,7 @@
 # ===========================================================================
 # Module final link
 # ===========================================================================
-
+$(info !! Enter $(MAKEFILE_LIST) !!)
 PHONY := __modfinal
 __modfinal:
 
diff --git a/scripts/Makefile.modpost b/scripts/Makefile.modpost
index 739402f45..2ccbfcf14 100644
--- a/scripts/Makefile.modpost
+++ b/scripts/Makefile.modpost
@@ -32,6 +32,8 @@
 # Step 4 is solely used to allow module versioning in external modules,
 # where the CRC of each module is retrieved from the Module.symvers file.
 
+$(info >>> Makefile.modpost)
+
 PHONY := __modpost
 __modpost:
 

```