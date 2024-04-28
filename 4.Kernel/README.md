<!--
 * @Author: meteor
 * @Date: 2024-04-28
 * @LastEditTime: 2024-04-28
 * @Description: 
 * 
 * Copyright (c) 2024
-->
# Kernel部分

- [4.移植Linux内核](#head0)

## <span id="head0">4.移植Linux内核</span>

首先进入u-boot配置过的docker环境，并检查必要库是否安装：
```
docker run -it f1c200s:latest
apt-get install flex bison -y
apt install libssl-dev -y
```

> 然后，下载LiCh-Pi的Linux配置文件，我已经下载好了，有需要自取，在[这里](bin/)。

接下来下载Linux-v5.7.1的内核文件，可以去[官网](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags?h=v5.10.161)找到	linux-5.7.1.tar.gz (sig)这一项进行下载并cp到docker中，也在docker可以使用下面的命令下载：
```
cd ~
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.7.1.tar.gz
```
*再用`tar -vxf linux-5.7.1.tar.gz`解压。*

解压完成后，进入该文件夹，首先编辑顶层的Makefile：
```
cd linux-5.7.1/
vim Makefile
```

> 在约363行左右，注释掉原来的ARCH ?= $(SUBARCH)，并添加自己的交叉编译器：
```
ARCH ?= arm
CROSS_COMPILE ?= arm-linux-gnueabi-
```

接下来将前述的LiCh-Pi的Linux配置文件，放到内核源码中的`arch/arm/configs`目录中，再用下面的命令配置Linux内核：
```
cd ~/linux-5.7.1/
make linux-licheepi_nano_defconfig
```

如果系统输出硬件上接的是uart0，那么直接进行编译就可以了：
```
make -j4
```

---
> **如果系统输出硬件上接的是uart1，那就需要修改内核配置文件；同时又因为文件系统会放到SD卡上，因此一并修改SD卡设备树信息，参考[这里](https://blog.csdn.net/GJF712/article/details/125213150)。**

> **1. 修改suniv-f1c100s.dtsi文件**
```
cd linux-5.7.1/arch/arm/boot/dts/
vim suniv-f1c100s.dtsi
```

首先添加头文件：
```
#include <dt-bindings/clock/suniv-ccu-f1c100s.h>
#include <dt-bindings/reset/suniv-ccu-f1c100s.h>
```

然后在soc->pio 下（第100行）添加如下代码：
```
mmc0_pins: mmc0-pins {
    pins = "PF0", "PF1", "PF2", "PF3", "PF4", "PF5";
    function = "mmc0";
};
```

在soc下（第113行）添加如下代码：
```
mmc0: mmc@1c0f000 {
    compatible = "allwinner,suniv-f1c100s-mmc",
                            "allwinner,sun7i-a20-mmc";
    reg = <0x01c0f000 0x1000>;
    clocks = <&ccu CLK_BUS_MMC0>,
                    <&ccu CLK_MMC0>,
                    <&ccu CLK_MMC0_OUTPUT>,
                    <&ccu CLK_MMC0_SAMPLE>;
    clock-names = "ahb",
                            "mmc",
                            "output",
                            "sample";
    resets = <&ccu RST_BUS_MMC0>;
    reset-names = "ahb";
    interrupts = <23>;
    pinctrl-names = "default";
    pinctrl-0 = <&mmc0_pins>;
    status = "disabled";
    #address-cells = <1>;
    #size-cells = <0>;
};
```

将第95行改为如下：
```
uart1_pins_a: uart1-pins-pa {
    pins = "PA2", "PA3";
    function = "uart1";
};
（我记得也可以叫uart1_pa_pins，看到文件中要注意，其实前后无所谓，保持所有用到的地方一致即可）
```
> 完成之后保存退出

> **2. 修改suniv-f1c100s-licheepi-nano.dts文件**
```
cd linux-5.7.1/arch/arm/boot/dts/
vim suniv-f1c100s-licheepi-nano.dts
```

在第21行添加以下代码，注意要与`chosen`在同一级：
```
reg_vcc3v3: vcc3v3 {
    compatible = "regulator-fixed";
    regulator-name = "vcc3v3";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
};
```

在该文件的最后添加以下代码：
```
&mmc0 {
    vmmc-supply = <&reg_vcc3v3>;
    bus-width = <4>;
    broken-cd;
    status = "okay";
};
```

再将第14行改为如下：
```
serial1 = &uart1;
```

将第18行改为如下：
```
stdout-path = "serial1:115200n8";
```

将第29~33行改为如下：
```
&uart1 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart1_pins_a>;
    status = "okay";
};
（注意uart1_pins_a这里要跟前面的一致）
```
> 完成之后保存退出

最后进行编译：
```
make -j4
```

> ~~电脑比较拉~~ 经过漫长的等待后，终于编译成功了，注意需要用的两个文件为内核文件（zImage）和设备树（.dtb文件），将其从docker中复制到主机的用户目录下，便于后续使用。我的两个文件在[这里](bin/)。
```
（在主机终端）
docker cp <container_id_or_name>:/root/linux-5.7.1/arch/arm/boot/zImage ~
docker cp <container_id_or_name>:/root/linux-5.7.1/arch/arm/boot/dts/suniv-f1c100s-licheepi-nano.dtb ~
```

---
> **然后在主机终端下进行操作。**

首先插入TF卡到虚拟机中，这里使用gparted来管理分区：
```
sudo apt install gparted -y
```

使用gparted将TF卡在物理上分为三个区：
- 前面1 MB用给u-boot（这并不是一个分区，而是分区的时候在前面留下1 MB空间，在gparted界面也看不到，fs格式为fat16）
- 中间分个32 MB的区来放置Linux Kernel（fs格式为fat16）
- 最后剩下的所有空间作为一个区，用于放置rootfs以及用户文件（fs格式为ext4）

| 分区 | 区域1 | 区域2 | 区域3 |
| ---- | ---- | ---- | ---- |
| 名称 | u-boot | kernel | rootfs |
| fs格式 | fat16 | fat16 | ext4 |
| 大小 | 1 MB | 32 MB <br>(可以随意写，</br>能放下kernel就行) | 剩余空间 |

![](../Docs/Images/kernel%20part.png)

> 最后将`zImage`和`suniv-f1c100s-licheepi-nano.dtb`放到kernel分区中

弹出U盘再拔出，然后将TF卡插入卡槽，上电后就可以看到Starting Kernel...。此时还没有rootfs，因此并不能使用用户文件。

![](../Docs/Images/kernel%20start.png)
*注意如果没有看到串口输出可以复位一下。*

最后退出docker环境，并保存该镜像，同时清除缓存：
```
exit
docker ps -a
docker commit <container_id_or_name> f1c200s:latest
docker rm <container_id_or_name>
```
