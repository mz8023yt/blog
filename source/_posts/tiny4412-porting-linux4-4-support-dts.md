---
title: '[Tiny4412] 移植 Linux4.4 到 Tiny4412 开发板上'
date: 2018-03-19 12:43:39
tags:
  - tiny4412
categories: Tiny4412
---

## 一. 前言

一直以来都是基于 Linux-2.6 内核学习嵌入式，但是工作之后发现，主流的 kernel 早已经不再使用 platform device 结构去描述设备信息了，而是换成了更为简洁的 device tree 方式。

这里贴出 wowotech 的 device tree 的引言：  
作为一个多年耕耘在linux 2.6.23内核的开发者，各个不同项目中各种不同周边外设驱动的开发以及各种琐碎的、扯皮的俗务占据了大部分的时间。当有机会下载3.14的内核并准备学习的时候，突然发现linux kernel对于我似乎变得非常的陌生了，各种新的机制，各种framework、各种新的概念让我感到阅读内核代码变得举步维艰。 还好，剖析内核的热情还在，剩下的就交给时间的。首先进入视线的是Device Tree机制，这是和porting内核非常相关的机制，如果想让将我们的硬件平台迁移到高版本的内核上，Device Tree是一个必须要扫清的障碍。 

参考文章：[Device Tree 背景介绍](http://www.wowotech.net/device_model/why-dt.html)

## 二. 移植 U-Boot

本小节主要是修改 U-Boot 源码，让其支持 dts 方式启动内核。

### 2.1 获取 U-Boot 源码

u-boot 使用的是友善之臂提供的 u-boot 源码，可以在友善之臂光盘获取。  
U-Boot 源码光盘路径：Disk-A\uboot\uboot_tiny4412-20130729.tgz

我同时上传了一份到我的 github 上，可以通过 git clone 获取到。

```bash
## 克隆 git 仓库获取源码
user@lenovo:~/workspace/tiny4412$ git clone git@github.com:tiny4412/source.code.git

## 进入 u-boot 源码目录
user@lenovo:~/workspace/tiny4412$ cd source.code/uboot/friendlyarm/
user@lenovo:~/workspace/tiny4412/source.code/uboot/friendlyarm$ ls
uboot_tiny4412-20130729.tgz

## 解压 u-boot
user@lenovo:~/workspace/tiny4412/source.code/uboot/friendlyarm$ tar -zxvf uboot_tiny4412-20130729.tgz -C ~/workspace/tiny4412/
```

### 2.2 修改源码以支持 dts 方式启动

前面已经提到，友善之臂光盘中提供的 u-boot 不支持引导 uImage 格式的 kernel，也不支持 device tree。针对这两个问题，对 u-boot 要做以下修改：

开启了 MMU 的话，u-boot 访问的都是虚拟地址，因为后期要使用 dnw 工具烧写文件到开发板 RAM 中，需要指定烧写到 RAM 中的具体物理地址，因此需要关闭 MMU。

```c
diff --git a/include/configs/tiny4412.h b/include/configs/tiny4412.h
old mode 100644
new mode 100755
index 2e83d54..413da6e
--- a/include/configs/tiny4412.h
+++ b/include/configs/tiny4412.h
@@ -308,7 +308,9 @@
 #define CONFIG_SYS_MONITOR_LEN         (256 << 10)     /* 256 KiB */
 #define CONFIG_IDENT_STRING            " for TINY4412"
 
-#define CONFIG_ENABLE_MMU
+#undef CONFIG_ENABLE_MMU
 
 #ifdef CONFIG_ENABLE_MMU
 #define CONFIG_SYS_MAPPED_RAM_BASE     0xc0000000
```

关闭 MMU 后，还需要修改 u-boot 的链接地址到真实物理地址范围内。

```c
diff --git a/board/samsung/tiny4412/config.mk b/board/samsung/tiny4412/config.mk
old mode 100644
new mode 100755
index dd7ec07..cc3488e
--- a/board/samsung/tiny4412/config.mk
+++ b/board/samsung/tiny4412/config.mk
@@ -9,5 +9,7 @@
 #  published by the Free Software Foundation.
 #
 #
-CONFIG_SYS_TEXT_BASE = 0xc3e00000
+CONFIG_SYS_TEXT_BASE = 0x43e00000
```

开启设备树相关功能宏控，使其支持设备树。

```c
diff --git a/include/configs/tiny4412.h b/include/configs/tiny4412.h
index 413da6e..b52dfa0 100755
--- a/include/configs/tiny4412.h
+++ b/include/configs/tiny4412.h
@@ -28,6 +28,11 @@
 #ifndef __CONFIG_H
 #define __CONFIG_H
 
+#define CONFIG_OF_LIBFDT
+#define CONFIG_SYS_BOOTMAPSZ (20 << 20)

 /*
  * High Level Configuration Options
  * (easy to change)
```

原生的 u-boot 使用 bootm 命令只可以引导 zImage 类型的 kernel，不支持 uImage，通过修改 bootm 命令可以使其支持 uImage。

```c
diff --git a/common/cmd_bootm.c b/common/cmd_bootm.c
old mode 100644
new mode 100755
index 04622dd..2cf6ea9
--- a/common/cmd_bootm.c
+++ b/common/cmd_bootm.c
@@ -591,6 +591,10 @@ int do_bootm (cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
        int             ret;
        boot_os_fn      *boot_fn;
 
+       int             iszImage = 0;
+
 #ifdef CONFIG_SECURE_BOOT
 #ifndef CONFIG_SECURE_BL1_ONLY
        security_check();
@@ -627,6 +631,10 @@ int do_bootm (cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
 
                images.legacy_hdr_valid = 1;
 
+               iszImage = 1;
+
                goto after_header_check;
        }
 #endif
@@ -723,8 +731,14 @@ int do_bootm (cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
 
 #if defined(CONFIG_ZIMAGE_BOOT)
 after_header_check:
-       images.os.os = hdr->ih_os;
-       images.ep = image_get_ep (&images.legacy_hdr_os_copy);
+
+       if (iszImage) {
+               images.os.os = hdr->ih_os;
+               images.ep = image_get_ep (&images.legacy_hdr_os_copy);
+       }
+
 #endif
 
 #ifdef CONFIG_SILENT_CONSOLE
```

### 2.3 安装交叉编译器

对于 kernel 和 u-boot 编译，最好使用同一个编译器，不妨在这 u-boot 编译之初，先配置一个同时支持 u-boot 以及 linux-4.4 版本的交叉编译器。  
我个人使用的是 arm-none-eabi-gcc 编译器，可以访问 [GNU Arm Embedded Toolchain](https://launchpad.net/gcc-arm-embedded/+download) 获取，也可以通过我的 git 仓库获取。

```bash
## 克隆 git 仓库获取交叉编译器
user@lenovo:~/workspace/tiny4412$ git clone git@github.com:tiny4412/tool.chain.git

## 找到交叉编译器
user@lenovo:~/workspace/tiny4412$ cd tool.chain/compiler/
user@lenovo:~/workspace/tiny4412/tool.chain/compiler$ ls
gcc-arm-none-eabi-5_4-2016q3-20160926-linux.tar.bz2

## 解压到 /opt 目录
user@lenovo:~$ sudo tar -jxvf gcc-arm-none-eabi-5_4-2016q3-20160926-linux.tar.bz2 -C /opt/

## 将交叉编译器路径添加到 PATH 环境变量中，并追加到配置文件中
user@lenovo:~$ export PATH=/opt/gcc-arm-none-eabi-5_4-2016q3/bin:$PATH >> .bashrc

## 执行配置文件，使环境变量生效
user@lenovo:~$ . .bashrc

## 查看交叉编译器版本号，确认交叉编译配置 ok
user@lenovo:~$ arm-none-linux-gnueabi-gcc -v
Using built-in specs.
Target: arm-none-linux-gnueabi
Configured with: /opt/FriendlyARM/mini2440/build-toolschain/working/src/gcc-4.4.3/configure --build=i386-build_redhat-linux-gnu --host=i386-build_redhat-linux-gnu --target=arm-none-linux-gnueabi --prefix=/opt/FriendlyARM/toolschain/4.4.3 --with-sysroot=/opt/FriendlyARM/toolschain/4.4.3/arm-none-linux-gnueabi//sys-root --enable-languages=c,c++ --disable-multilib --with-arch=armv4t --with-cpu=arm920t --with-tune=arm920t --with-float=soft --with-pkgversion=ctng-1.6.1 --disable-sjlj-exceptions --enable-__cxa_atexit --with-gmp=/opt/FriendlyARM/toolschain/4.4.3 --with-mpfr=/opt/FriendlyARM/toolschain/4.4.3 --with-ppl=/opt/FriendlyARM/toolschain/4.4.3 --with-cloog=/opt/FriendlyARM/toolschain/4.4.3 --with-mpc=/opt/FriendlyARM/toolschain/4.4.3 --with-local-prefix=/opt/FriendlyARM/toolschain/4.4.3/arm-none-linux-gnueabi//sys-root --disable-nls --enable-threads=posix --enable-symvers=gnu --enable-c99 --enable-long-long --enable-target-optspace
Thread model: posix
gcc version 4.4.3 (ctng-1.6.1)
```

### 2.4 配置编译 U-Boot

修改 U-Boot 编译器为交叉编译器。

```c
diff --git a/Makefile b/Makefile
old mode 100644
new mode 100755
index 48e8eaa..04e0132
--- a/Makefile
+++ b/Makefile
@@ -157,7 +157,7 @@ export      ARCH CPU BOARD VENDOR SOC
 
 # set default to nothing for native builds
 ifeq ($(HOSTARCH),$(ARCH))
-CROSS_COMPILE ?=
+CROSS_COMPILE ?= arm-none-eabi-
 endif
 
 # load other configuration
```

配置编译 U-Boot。

```bash
user@lenovo:~/workspace/tiny4412/uboot.support.dts$ make distclean 
user@lenovo:~/workspace/tiny4412/uboot.support.dts$ make clean 
user@lenovo:~/workspace/tiny4412/uboot.support.dts$ make tiny4412_config
Configuring for tiny4412 board...
user@lenovo:~/workspace/tiny4412/uboot.support.dts$ make -j24

```

### 2.5 烧写 U-Boot 到 SD 卡

在烧写的时候经常遇到这两个错误：

在 uboot 根目录下执行 sd_fusing.sh 脚本，报错如下：

```bash
## 在 uboot 根目录下执行 sd_fusing.sh 脚本
user@lenovo:~/workspace/tiny4412/uboot.support.dts$ sudo ./sd_fuse/tiny4412/sd_fusing.sh /dev/sdb
/dev/sdb reader is identified.
Error: u-boot.bin NOT found, please build it & try again.
./sd_fuse/tiny4412/sd_fusing.sh: 49: exit: Illegal number: -1

## 找不到 mkbl2 镜像
user@lenovo:~/workspace/tiny4412/uboot.support.dts/sd_fuse/tiny4412$ sudo ./sd_fusing.sh /dev/sdb
/dev/sdb reader is identified.
Error: can not find host tool - mkbl2, stop.
./sd_fusing.sh: 54: exit: Illegal number: -1
```

在 uboot 根目录下执行 sd_fusing.sh 脚本报错是因为脚本中使用的相对路径访问的 uboot 镜像，只有在 src/sd_fuse/tiny4412/ 目录下执行，路径才对的上。  
另外找到 mkbl2 是因为执行 uboot 需要依赖 mkbl2。这个文件是三星官方给的，可以在 src/sd_fuse/ 目录下执行 make 生成。

```bash
user@lenovo:~/workspace/tiny4412/uboot.support.dts/sd_fuse/tiny4412$ cd ../
user@lenovo:~/workspace/tiny4412/uboot.support.dts/sd_fuse$ make
gcc -o	mkbl2 V310-EVT1-mkbl2.c 
gcc -o	sd_fdisk sd_fdisk.c
```

正确的烧写流程：

```bash
user@lenovo:~/workspace/tiny4412/uboot.support.dts$ cd sd_fuse/
user@lenovo:~/workspace/tiny4412/uboot.support.dts/sd_fuse$ make
gcc -o	mkbl2 V310-EVT1-mkbl2.c 
gcc -o	sd_fdisk sd_fdisk.c
user@lenovo:~/workspace/tiny4412/uboot.support.dts/sd_fuse$ cd tiny4412/
user@lenovo:~/workspace/tiny4412/uboot.support.dts/sd_fuse/tiny4412$ sudo ./sd_fusing.sh /dev/sdb
/dev/sdb reader is identified.
---------------------------------------
BL1 fusing
记录了16+0 的读入
记录了16+0 的写出
8192 bytes (8.2 kB, 8.0 KiB) copied, 0.096504 s, 84.9 kB/s
---------------------------------------
BL2 fusing
记录了28+0 的读入
记录了28+0 的写出
14336 bytes (14 kB, 14 KiB) copied, 0.2371 s, 60.5 kB/s
---------------------------------------
u-boot fusing
记录了569+1 的读入
记录了569+1 的写出
291484 bytes (291 kB, 285 KiB) copied, 2.68892 s, 108 kB/s
---------------------------------------
TrustZone S/W fusing
记录了184+0 的读入
记录了184+0 的写出
94208 bytes (94 kB, 92 KiB) copied, 0.923827 s, 102 kB/s
---------------------------------------
U-boot image is fused successfully.
Eject SD card and insert it again.
```

将 u-boot 下载 SD 卡后，将 SD 卡插入 Tiny4412 中，使用 SD 启动方式，可以进入 u-boot 命令行。

```bash
U-Boot 2010.12-00000-gef514bc-dirty (Mar 28 2018 - 21:58:58) for TINY4412


CPU:	S5PC220 [Samsung SOC on SMP Platform Base on ARM CortexA9]
	APLL = 1400MHz, MPLL = 800MHz

Board:	TINY4412
DRAM:	1023 MiB

vdd_arm: 1.2
vdd_int: 1.0
vdd_mif: 1.1

BL1 version:  N/A (TrustZone Enabled BSP)


Checking Boot Mode ... SDMMC
REVISION: 1.1
MMC Device 0: 7460 MB
MMC Device 1: 3728 MB
MMC Device 2: N/A
*** Warning - using default environment

Net:	No ethernet found.
Hit any key to stop autoboot:  0 
reading kernel..device 0 Start 1057, Count 12288 
MMC read: dev # 0, block # 1057, count 12288 ... 12288 blocks read: OK
completed
reading RFS..device 0 Count 13345, Start 2048 
MMC read: dev # 0, block # 13345, count 2048 ... 2048 blocks read: OK
completed
Wrong Image Format for bootm command
ERROR: can't get kernel image!
TINY4412 # 
```

在 u-boot 中执行以下命令，可以将 u-boot 从 sd 中拷贝到 emmc 中，以后便可以通过 emmc 启动了。
但是很遗憾，我试验下来并不行，原因暂时没有找到。

```bash
## 将 u-boot 从 SD 卡复制到 DDR 中
TINY4412 # mmc read 0 0x40000000 1 390
MMC read: dev # 0, block # 1, count 912 ... 912 blocks read: OK

## 打开 EMMC
TINY4412 # emmc open 1
eMMC OPEN Success.!!
			!!!Notice!!!
!You must close eMMC boot Partition after all image writing!
!eMMC boot partition has continuity at image writing time.!
!So, Do not close boot partition, Before, all images is written.!

## 将 DDR 地址中的 u-boot 写到 EMMC
TINY4412 # mmc write 1 0x40000000 0 390
MMC write: dev # 1, block # 0, count 912 ... 912 blocks written: OK

## 关闭 EMMC
TINY4412 # emmc close 1
eMMC CLOSE Success.!!
```

## 三. 移植 Kernel

主要是配置好 dts 移植好 usb 网卡驱动，这些操作我都已经做好了，可以直接访问 github 获取。  
具体的修改内容可以查看 github 仓库的提交记录。

```bash
user@lenovo:~/workspace/tiny4412$ git clone git@github.com:tiny4412/linux-4.4.x.git
user@lenovo:~/workspace/tiny4412$ cd linux-4.4.x/
## 清除内核配置
user@lenovo:~/workspace/tiny4412/linux-4.4.x$ make distclean
user@lenovo:~/workspace/tiny4412/linux-4.4.x$ make clean
## 配置内核
user@lenovo:~/workspace/tiny4412/linux-4.4.x$ make exynos_defconfig
## 使配置生效，生成 .config 文件
user@lenovo:~/workspace/tiny4412/linux-4.4.x$ make menuconfig
## 开始编译
user@lenovo:~/workspace/tiny4412/linux-4.4.x$ make -j24 uImage LOADADDR=0x40008000
## 将内核 uImage 拷贝到根目录下，方便后面使用 dts 下载。
user@lenovo:~/workspace/tiny4412/linux-4.4.x$ ./maziot.sh 
'arch/arm/boot/dts/exynos4412-tiny4412.dtb' -> './exynos4412-tiny4412.dtb'
'arch/arm/boot/uImage' -> './uImage'
```

## 四. 构建根文件系统

已经做好，可以直接访 github 获取。

```bash
user@lenovo:~/workspace/tiny4412$ git clone git@github.com:tiny4412/rootfs.git
```

如何使用呢？

1. 开发板在 u-boot 中设置 bootargs 启动参数

```bash
# nfs 启动方式的 root 属性格式为 nfsroot=<服务器IP地址>:<服务器上的文件目录>  ip=<开发板IP地址>:<服务器IP地址>:<网关>:<子网掩码>::eth0:off
setenv bootargs 'root=/dev/nfs rw nfsroot=192.168.1.7:/home/user/workspace/tiny4412/rootfs ethmac=1C:6F:65:34:51:7E ip=192.168.1.100:192.168.1.7:192.168.1.1:255.255.255.0::eth0:off console=ttySAC0,115200 init=/linuxrc'
save
```


