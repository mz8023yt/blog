---
title: '[CAR] 学习总结(二) 搭建环境'
date: 2018-07-09 23:26:49
tags:
  - train
categories: Experience
---

### 实践环境要求

实践环境基本需要有：

- 一台 ubuntu 主机：用于编译驱动和测试程序
- 一块开发板：用于验证驱动和测试程序
- 其他小东西：下载器，usb线，串口线等

### 虚拟实践环境

ubuntu 主机的问题很好处理，使用 vmware 创建一个虚拟机，并安装一个 ubuntu 系统即可。开发板的话就比较麻烦了，这里本系列实践记录欲使用“宋宝华”老师《Linux设备驱动开发详解：基于最新的Linux 4.0内核》书中的 QEMU 实验平台验证，不能保证一定 ok，目前正在尝试中。  
先尝试下，如果可以的实现的就基于 VMware(或 Virtual Box) + Qemu 做实践，不行的话，只能拿出我的 mini2440 开发板搞事情了或者搞一套公司的硬件验证了，顺便熟悉项目的编译，烧写，启动，或许操作会麻烦点，但是可以提前熟悉项目还是很不错的。

总结下就是，用 vmware 虚拟一台 ubuntu 主机，用 qemu 虚拟一台 develop board。


#### QEMU 介绍

介绍 QEMU 的相关博客：

<https://www.cnblogs.com/bakari/p/7858029.html>
<https://blog.csdn.net/zhaihaifei/article/details/51028823>
<https://www.ibm.com/developerworks/cn/linux/l-qemu/>

### 尝试搭建环境

#### 第一步：下载书籍对应的光盘资料

下载《Linux设备驱动开发详解：基于最新的Linux 4.0内核》配套的光盘，光盘中有宋宝华老师已经搭建好的ubuntu虚拟机和qemu cortex-A9 开发板。
光盘资料在百度云可以下载，老的百度云链接已经失效了，需要关注宋宝华老师的微信公众号："Linux阅码场"，在公众号中可以获取最新的光盘下载地址。  
获取书籍配套光盘百度云链接的步骤：关注 "Linux阅码场" 公众号 -> 书和课程 -> 书籍FAQ -> 配套光盘下载地址。

#### 第二步：使用 VMware(或Virtual Box)安装ubuntu

参考光盘中的《Linux虚拟机安装手册.docx》文档，安装光盘中提供的虚拟机，此虚拟机中已经配置好了开发环境，当然包括 QEMU 了。  
先尝试使用 VMware 14 安装虚拟机，安装后 ubuntu 开机会 panic，进不了系统。因此从新使用光盘中提供的 Virtual Box 成功安装。至于 VMware 7.0 没有尝试，谁有兴趣可以试试，并帮忙补全下。

安装好 ubuntu 后，虚拟机账户密码都是 baohua，账户：baohua，密码：baohua。  
为了在 Virtual Box 和 windows 之间方便拷贝，需要设置共享粘贴板：设备 -> 共享粘贴板 -> 双向。

#### 第三步：编译、烧写、运行书籍配套内核

接下来就是看看内核怎么编译，如何烧写到 QEMU 中，以及如何启动 QEMU 开发板。  

源码在哪？源码在 `/home/baohua/develop/linux` 位置，编译脚本宝华已经做好了，`build.sh`  脚本位于源码根目录。  
尝试着编译了下，没问题，可以编译过：

```
baohua@baohua-VirtualBox:~/develop/linux$ ./build.sh 
  CHK     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
make[1]: `include/generated/mach-types.h' is up to date.
  CALL    scripts/checksyscalls.sh
  CHK     include/generated/compile.h
  CHK     kernel/config_data.h
  Kernel: arch/arm/boot/Image is ready
  Kernel: arch/arm/boot/zImage is ready
  CHK     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
make[1]: `include/generated/mach-types.h' is up to date.
  CALL    scripts/checksyscalls.sh
  Building modules, stage 2.
  MODPOST 5 modules
  CHK     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
make[1]: `include/generated/mach-types.h' is up to date.
  CALL    scripts/checksyscalls.sh
```

简单看一下编译脚本 build.sh

```
export ARCH=arm
export EXTRADIR=${PWD}/extra
export CROSS_COMPILE=arm-linux-gnueabi-
#make LDDD3_vexpress_defconfig
make zImage -j8
make modules -j8
make dtbs
cp arch/arm/boot/zImage extra/
cp arch/arm/boot/dts/*ca9.dtb   extra/
cp .config extra/
```

可以看到最后将编译得到的 zImage 和 dtb 文件都拷贝到了源码根目录下的 extra/ 目录中，配合《Linux设备驱动开发详解》中介绍，extra 是一张虚拟的 SD 卡，作为 QEMU 板子的文件系统存放介质，因此这里的拷贝动作其实就是下载的动作。  
编译和烧写的动作，build.sh 已经做好了，剩下要做的就是要知道，如何编写自己的内核模块，以及修改设备树，最后又该如何 insmod 到 QEMU 上。先启动下 QEMU 版子看看：

```bash
baohua@baohua-VirtualBox:~$ cd develop/linux/extra/
baohua@baohua-VirtualBox:~/develop/linux/extra$ ls
img         run-nolcd.sh  u-boot.bin  vexpress.img          zImage
run-lcd.sh  u-boot        u-boot.sh   vexpress-v2p-ca9.dtb
baohua@baohua-VirtualBox:~/develop/linux/extra$ ./run-lcd.sh 
WARNING: Image format was not specified for 'vexpress.img' and probing guessed raw.
         Automatically detecting the format is dangerous for raw images, write operations on block 0 will be restricted.
         Specify the 'raw' format explicitly to remove the restrictions.
audio: Could not init `oss' audio driver
```

运行 run-lcd.sh 脚本后，将自动调起 QEMU，成功运行。

![image.png](https://upload-images.jianshu.io/upload_images/11006334-cd1d96db3af8ba4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 编写内核驱动模块和测试程序

试着编写一个最简单的内核模块 `module.c`，通过动态加载的方式加载到内核：

```c
#include <linux/module.h>
#include <linux/init.h>

#define log(fmt, arg...) printk(KERN_INFO "[Paul][%s][%d] "fmt"\n", __func__, __LINE__, ##arg);

static int __init mod_init(void)
{
	log("mod_init ok");
	return 0;
}

static void __exit mod_exit(void)
{
        log("mod_exit ok");
}
module_init(mod_init);
module_exit(mod_exit);

MODULE_LICENSE("GPL");
```

接下来编写对应的 Makefile：

```
obj-m += module.o

KERNEL = /home/baohua/develop/linux

all:
	make -C $(KERNEL) M=`pwd` modules
	@echo ">>>>>> make all successful <<<<<<"

clean:
	make -C $(KERNEL) M=`pwd` modules clean
	@echo ">>>>>> make clean successful <<<<<<"
```

这两个文件都放在 `/home/baohua/develop/wangbing` 目录下，可是编译一下，居然报错，很是尴尬，看看宝华提供的编译 modules 的脚本 `module.sh` 是怎么编译模块的

```
baohua@baohua-VirtualBox:~/develop/wangbing$ cat ../linux/module.sh 
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- modules
sudo mount -o loop,offset=$((2048*512)) extra/vexpress.img extra/img
sudo make ARCH=arm modules_install INSTALL_MOD_PATH=extra/img
sudo umount extra/img
```

将 arch 和 cross 加上试试，修改 Make 后：

```
obj-m += module.o

KERNEL = /home/baohua/develop/linux

all:
	make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -C $(KERNEL) M=`pwd` modules
	@echo ">>>>>> make all successful <<<<<<"

clean:
	make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -C $(KERNEL) M=`pwd` modules clean
	@echo ">>>>>> make clean successful <<<<<<"
```

加上后果然成功编译出 module.ko 模块，再看看怎么拷贝到开发板上，同样参考宝华提供的 `module.sh` 脚本。配套书籍上有这么一句话：

> extra 目录下的 vexpress.img 是一张虚拟 SD 卡，将作为根文件系统的存放介质。它能以 loop 的形式被挂载，如执行: `sudo mount -o loop,offset=1048576 $(KERNEL)/extra/vexpress.img $(KERNEL)/extra/img` 就可以将  vexpress.img 挂载到 `$(KERNEL)/extra/img` 位置。

因此，尝试一下，这么干：

```
sudo mount -o loop,offset=1048576 /home/baohua/develop/linux/extra/vexpress.img /home/baohua/develop/linux/extra/img
baohua@baohua-VirtualBox:~/develop/wangbing$ sudo cp module.ko /home/baohua/develop/linux/extra/img/
```

看了一眼 QEMU，不行呀，没有 module.ko 文件。重启一下试试，重启之后果然有了(如果没有就再重启一下)。

![image.png](https://upload-images.jianshu.io/upload_images/11006334-415a41942f17596e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

至此，我们已经搭建好了基于 Virtual Box 和 QEMU 的开发环境。

### 备注

鼠标点进 QEMU 虚拟的开发板终端后，鼠标将被屏蔽，此时，切回到 ubuntu 其他窗口，鼠标依然是隐藏的状态，并不会主动显示出来，需要使用 Ctrl + Alt + G 组合键，释放鼠标光标。

