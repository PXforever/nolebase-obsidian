---
share: "true"
---
# Device Tree语法

> 背景就不再累述
### 参考链接：
[一文搞懂设备树基础语法](https://blog.csdn.net/weixin_39674445/article/details/118398831)
[设备树基本语法](https://blog.csdn.net/challenglistic/article/details/131867382)
## DTS DTSI DTB
> 这个三个文件是开发中常见的文件
![[笔记/01 附件/Linux驱动学习-Device tree语法/设备树转换.drawio.png|设备树转换.drawio]]
通过如下代码实现转换(脚本在kernel里有)：
```shell
./scripts/dtc/dtc -I dts -O dtb -o tmp.dtb arch/arm/boot/dts/xxx.dts  # 编译dts为dtb
./scripts/dtc/dtc -I dtb -O dts -o tmp.dts arch/arm/boot/dts/xxx.dtb  #反编译dtb为dts
```
`*.dtsi`一般是由soc厂商编写，主要是描述硬件信息，我们拿到之后二次开发需要编写`*.dts`文件，在该文件使用`#include "sun8iw11p1_pwm1.dtsi"` 来包含其他设备树文件。
### 如何添加自己的dts文件
文件名：`user.dts`,内容：定义一个`my_dev`节点。
在目录`linux-3.10\arch\arm\boot\dts`下的`Makefile`文件控制编译位置加入：
```makefile
dtb-$(CONFIG_ARCH_SUN8IW11P1) += sun8iw11p1-fpga.dtb \
	sun8iw11p1-soc.dtb \
	sun8iw11p1-OKA40i_C.dtb \
	sun8iw11p1-OKA40i_S.dtb \
	sun8iw11p1-OKT3_C.dtb \
	sun8iw11p1-OKT3_S.dtb \
	user.dtb
```
**注意：`CONFIG_ARCH_SUN8IW11P1`是在配置过程中产生的，这里不作展开讲解**
使用命令行(`kernel`目录)：
```shell
source ./misc_config
AW_LICHEE_ROOT=/home/forlinx/work/lichee
export CROSS_COMPILE_DIR=$AW_LICHEE_ROOT/out/${MISC_CHIP}/linux/common/buildroot/host/opt/ext-toolchain/bin/
export CROSS_COMPILE=$CROSS_COMPILE_DIR/arm-linux-${MISC_GNUEABI}-
export AS=${CROSS_COMPILE}as
export LD=${CROSS_COMPILE}ld
export CC=${CROSS_COMPILE}gcc
export ARCH=arm

make dtbs
```
设备树如下写：
```c
/*
* user add a devices tree file
*
*/

/dts-v1/; //版本

#include "sun8iw11p1-OKA40i_C.dts"

/{
	my_dev {
		name = "user";
	};
}
```
似乎不能使用`#include "sun8iw11p1-OKA40i_C.dts"`这样会产生混乱。猜想可能最终的`dtb`文件只能有一个,所以目标不知道是使用`user.dtb`还是`sun8iw11p1-OKA40i_C.dtb`。所以将该文件写成`user.dtsi`,在`sun8iw11p1-OKA40i_C.dts`文件`include`。
## 基础语法
```C
/dts-v1/;

/{
	cpu {
		name = val;
	};

	spi {
		compatible = "spi-controlor";
		SHT4X {
		
		};
	};
};
```

![[笔记/01 附件/Linux驱动学习-Device tree语法/image-20230401195957509.png|image-20230401195957509]]
**注意：val有三种情况**
+ string: `"val"`
+ 32位无符号整型(用尖括号)：`<1>`
+ 二进制数据（方括号）：`binary-property = [0x01 0x23 0x45 0x67]`

### node节点语法
```assembly
/{
	[property definitions]
	[child nodes]
	
	[lable:]child-node-name[@address] {
        [property definitions]
        [child nodes]
    };
};
```
##### 示例
```c
/{
    twi4: twi@0x01c2c000{
        #address-cells = <1>;
        #size-cells = <0>;
        compatible = "allwinner,sun8i-twi";
        status = "disabled";
    };
    cpus {
        #address-cells = <1>;
        #size-cells = <0>;

        cpu@0 {
            device_type = "cpu";
            compatible = "arm,cortex-a7";
            reg = <0x0>;
            clock-frequency = <1008000000>;
        };
        cpu@1 {
            device_type = "cpu";
            compatible = "arm,cortex-a7";
            reg = <0x1>;
            clock-frequency = <1008000000>;
        };
    };
};
```

```C
//需要在twi4中添加子设备,或者修改
&twi4 {
    status = "okay";
    
    lcd_i2c {
      #address-cell = <1>;
    };
};
```

### 特殊属性
```c
#address-cells = <2>; //在它的子节点的reg属性中，用多少个32位整数来描述地址。
#size-cells = <2>; //在它的子节点的reg属性中，用多少个u32来描述大小。
model = "sun8iw11p1"; //当前的这个板子是什么
compatible = "arm,sun8iw11p1", "arm,sun8iw11p1"; //定义一系列的字符串，用来表示这个板子兼容哪些平台。
```
**注意：以上四个属性根节点必须要有。**

##### 示例
```c
soc: soc@01c00000 {
    compatible = "simple-bus";
    #address-cells = <2>; 	//子节点的reg有2个描述地址，看twi1
    #size-cells = <2>;		//子节点有2个u32来描述大小,看twi1
    device_type = "soc";
    
    twi1: twi@0x01c2b000{
        #address-cells = <1>;
        #size-cells = <0>;
        compatible = "allwinner,sun8i-twi";
        device_type = "twi1";
        reg = <0x0 0x01c2b000 0x0 0x400>; //因此，2个32表示地址开始，2个32表示地址跨度，这样解释：0x0000 0000 0x01c2 b000是twi1外设的起始地址，该外设所有寄存器地址长度为：0x0000 0000 0x0000 0400。
        flash:m25p80@0{
            #address-cells=<1>;
            #size-cells=<1>;
            compatible = "w25q128","w25q64","w25q256";
            reg=<0>; //这里应该是错误示范，应该 #size-cells=<0>;
            spi-max-frequency=<40000000>;
            mode=<0>;
            m25p,fast-read;
        };
    };
};
```

> ```text
> #address-cells = <2> //address1与address2几个32位
> #size-cells = <2> //length1与length2几个32位
> reg = <address1 length1 address2 length2 >
> ```

## 特殊的节点
+ 根节点，这是必须的节点
+ memory节点: 设置内存大小
+ chosen节点：`通过设备树文件给内核传入一些参数，需要在chosen节点中设置bootargs属性`
+ cpu节点

### 示例
```C
/{
    #address-cells = <2>;
	#size-cells = <2>;
    cpus {
		#address-cells = <1>;
		#size-cells = <0>;
		cpu@0 {
			device_type = "cpu";
		};
		cpu@1 {

		};
	};
    chosen {
        bootargs = "earlyprintk=sunxi-uart,0x01c28000 loglevel=8 initcall_debug=1 console=ttyS0 init=/init";
        linux,initrd-start = <0x0 0x0>;
        linux,initrd-end = <0x0 0x0>;
	};
    memory@40000000 {
		device_type = "memory";
		reg = <0x00000000 0x40000000 0x00000000 0x40000000>;
	};
};
```

## 常用的属性
+ `compatible`
```C
spi{
	compatible = "spi-1","spi-2","spi-3";
}; // 表示这个spi或者硬件兼容驱动spi-1,spi-2,spi-3.
```
+ `model`
```c
//model用来准确的定义这个硬件是什么。
model = "sun8iw11p1"; //当前的这个板子是什么
compatible = "arm,sun8iw11p1", "arm,sun8iw11p1"; //定义一系列的字符串，用来表示这个板子兼容哪些平台。
```
+ `status`
```C
&spi1{
	status = "disabled";
    //status = "okay";
};
```
+ `reg`
```c
//在设备树里用来描述一段空间。
#address-cells = <2>;
#size-cells = <2>;
memory@40000000 {
    device_type = "memory";
    reg = <0x00000000 0x40000000 0x00000000 0x40000000>;
};//内存起始0x0000000040000000 长度0x0000000040000000
```

## 代码获取属性
> 部分属性会被内核使用，所以不用我们来解析，但是有些自定义需要我们来敲入代码解析，使用的函数一般如下：
```c
//找到节点
of_find_node_by_path // 根据路径找到节点
of_find_node_by_name // 根据名字找到节点
of_find_compatible_node // 根据属性找到节点
of_find_compatible_node // 找到父节点
of_get_next_child // 取出下一个节点
of_get_next_available_child // 取出下一个可用的节点
of_get_child_by_name // 根据名字取出子节点
//找到属性
of_find_property // 找到节点中的属性
//获取属性的值
of_get_property // 根据名字找到节点的属性，并且返回它的值。
```

## 高级用法
> 略
+ port，phandle， endpoint
