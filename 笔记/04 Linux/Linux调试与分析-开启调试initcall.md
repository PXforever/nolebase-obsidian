---
share: "true"
---

> 在调试内核的过程中，我们常常需要知道模块的运行情况，确定模块是否执行，`Probe`函数是否执行成功。以及`initcall`的启动顺序。
> 我们可以利用内核参数`initcall_debug`进行调试。

我们可以选择如下2种方式进行调试：
+ 通过`uboot`命令行，添加到`bootargs`中
```shell
# uboot下执行
setenv bootargs "${bootargs} initcall_debug"
saveenv
```
+ 使用设备树：
```shell
&chosen {
  bootargs = "initcall_debug earlycon=uart8250,mmio32,0xfeb50000 console=ttyFIQ0 irqchip.gicv3_pseudo_nmi=0 rw rootwait rcupdate.rcu_expedited=1 rcu_nocbs=all";
};
```

效果如下：
![[笔记/01 附件/Linux调试与分析-开启调试initcall/file-20241205100934014.png|笔记/01 附件/Linux调试与分析-开启调试initcall/file-20241205100934014.png]]
可以看到先打印的是`Probe`函数结果，然后就是打印`init`执行的返回结果。