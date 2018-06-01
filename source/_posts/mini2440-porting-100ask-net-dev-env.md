---
title: '[Mini2440] 搭建韦东山二期视频学习环境'
date: 2018-05-19 19:18:11
tags:
  - mini2440
  - 100ask
categories: Mini2440
---

### 前言

韦东山老师的视频使用的是 JZ2440 开发板，本来使用视频配套的 JZ2440 开发板学习起来才会事半功倍，但是目前我手头上已经有了 mini2440 开发板以及配套的 X35 LCD，想将现有资源充分利用起来。并且韦东山老师经常说天下 2440 开发板都一样，让我更加断绝了购买 JZ2440 的欲望。因此决定使用 mini2440 学习韦东山老师的视频，才有了本文。  
本文主要记录在 mini2440 上移植 u-boot-1.1.6 和 linux-2.6.22.6 学习环境，总结此文，以便日后查阅。  

### 一 配置编译 u-boot

#### 1.1 获取 u-boot 源码并打补丁

解压 u-boot 源码

    user@vmware:~/mini2440$ tar jxf u-boot-1.1.6.tar.bz2
    user@vmware:~/mini2440$ cd u-boot-1.1.6/

清除配置

    user@vmware:~/mini2440/u-boot-1.1.6$ make distclean

创建 git 版本库管理源码

    user@vmware:~/mini2440/u-boot-1.1.6$ git init
    Initialized empty Git repository in /home/user/mini2440/u-boot-1.1.6/.git/
    user@vmware:~/mini2440/u-boot-1.1.6$ git add --all
    user@vmware:~/mini2440/u-boot-1.1.6$ git commit -m "init: get the u-boot source code by 100ask.net"

给 u-boot 打补丁

    user@vmware:~/mini2440/u-boot-1.1.6$ patch -p1 < ../u-boot-1.1.6_jz2440.patch
    user@vmware:~/mini2440/u-boot-1.1.6$ git add --all
    user@vmware:~/mini2440/u-boot-1.1.6$ git commit -m "patch: u-boot-1.1.6_jz2440.patch"

#### 1.2 配置、编译 u-boot

配置

    user@vmware:~/mini2440/u-boot-1.1.6$ make 100ask24x0_config
    Configuring for 100ask24x0 board...

编译

    user@vmware:~/mini2440/u-boot-1.1.6$ make
    ... ...
    /opt/gcc/arm/4.3.2/bin/../lib/gcc/arm-none-linux-gnueabi/4.3.2/armv4t/libgcc.a(_udivsi3.o): In function `__aeabi_uidiv':
    (.text+0x0): multiple definition of `__udivsi3'
    lib_arm/libarm.a(_udivsi3.o):/home/user/mini2440/u-boot-1.1.6/lib_arm/_udivsi3.S:17: first defined here
    arm-linux-ld: ERROR: Source object /opt/gcc/arm/4.3.2/bin/../lib/gcc/arm-none-linux-gnueabi/4.3.2/armv4t/libgcc.a(_udivdi3.o) has EABI version 5, but target u-boot has EABI version    0
    arm-linux-ld: failed to merge target specific data of file /opt/gcc/arm/4.3.2/bin/../lib/gcc/arm-none-linux-gnueabi/4.3.2/armv4t/libgcc.a(_udivdi3.o)
    arm-linux-ld: ERROR: Source object /opt/gcc/arm/4.3.2/bin/../lib/gcc/arm-none-linux-gnueabi/4.3.2/armv4t/libgcc.a(_udivsi3.o) has EABI version 5, but target u-boot has EABI version    0
    arm-linux-ld: failed to merge target specific data of file /opt/gcc/arm/4.3.2/bin/../lib/gcc/arm-none-linux-gnueabi/4.3.2/armv4t/libgcc.a(_udivsi3.o)
    arm-linux-ld: ERROR: Source object /opt/gcc/arm/4.3.2/bin/../lib/gcc/arm-none-linux-gnueabi/4.3.2/armv4t/libgcc.a(_dvmd_lnx.o) has EABI version 5, but target u-boot has EABI version    0
    arm-linux-ld: failed to merge target specific data of file /opt/gcc/arm/4.3.2/bin/../lib/gcc/arm-none-linux-gnueabi/4.3.2/armv4t/libgcc.a(_dvmd_lnx.o)
    arm-linux-ld: ERROR: Source object /opt/gcc/arm/4.3.2/bin/../lib/gcc/arm-none-linux-gnueabi/4.3.2/armv4t/libgcc.a(_clz.o) has EABI version 5, but target u-boot has EABI version 0
    arm-linux-ld: failed to merge target specific data of file /opt/gcc/arm/4.3.2/bin/../lib/gcc/arm-none-linux-gnueabi/4.3.2/armv4t/libgcc.a(_clz.o)
    /opt/gcc/arm/4.3.2/bin/../lib/gcc/arm-none-linux-gnueabi/4.3.2/armv4t/libgcc.a(_dvmd_lnx.o): In function `__aeabi_ldiv0':
    (.text+0x8): undefined reference to `raise'
    Makefile:263: recipe for target 'u-boot' failed
    make: *** [u-boot] Error 1

编译报错了，百度确认到是因为我使用的编译器是友善之臂的提供的 4.3.2 版本的编译器，编译器的库文件和 U-Boot 源码中的函有重复定义，换成 100ask 光盘提供的 3.4.5 版本的编译器就不存在此问题。  
解压 100ask 提供的交叉编译器

    user@vmware:~/mini2440/u-boot-1.1.6$ cd ../
    user@vmware:~/mini2440$ tar jxf arm-linux-gcc-3.4.5-glibc-2.3.6.tar.bz2
    user@vmware:~/mini2440$ cd gcc-3.4.5-glibc-2.3.6/bin/
    user@vmware:~/mini2440/gcc-3.4.5-glibc-2.3.6/bin$ pwd
    /home/user/mini2440/gcc-3.4.5-glibc-2.3.6/bin

修改 u-boot 的 Makefile 指定编译使用刚刚解压的交叉编译器，修改点如下：

    user@vmware:~/mini2440/gcc-3.4.5-glibc-2.3.6/bin$ cd ../../u-boot-1.1.6/
    user@vmware:~/mini2440/u-boot-1.1.6$ vim Makefile 
    user@vmware:~/mini2440/u-boot-1.1.6$ git diff
    diff --git a/Makefile b/Makefile
    index a8fdbb1..f7ed826 100644
    --- a/Makefile
    +++ b/Makefile
    @@ -125,7 +125,7 @@ ifeq ($(ARCH),ppc)
     CROSS_COMPILE = powerpc-linux-
     endif
     ifeq ($(ARCH),arm)
    -CROSS_COMPILE = arm-linux-
    +CROSS_COMPILE = /home/user/mini2440/gcc-3.4.5-glibc-2.3.6/bin/arm-linux-
     endif
     ifeq ($(ARCH),i386)
     ifeq ($(HOSTARCH),i386)
    user@vmware:~/mini2440/u-boot-1.1.6$ git add --all
    user@vmware:~/mini2440/u-boot-1.1.6$ git commit -m "conf: modify the cross compiler"

重新配置编译

    user@vmware:~/mini2440/u-boot-1.1.6$ make distclean
    user@vmware:~/mini2440/u-boot-1.1.6$ make 100ask24x0_config
    user@vmware:~/mini2440/u-boot-1.1.6$ make

确认到 u-boot.bin 镜像成功生成

    user@vmware:~/mini2440/u-boot-1.1.6$ ls -l u-boot.bin
    -rwxrwxr-x 1 user user 198324 5月  17 22:39 u-boot.bin

#### 1.3 给 u-boot 添加 .gitignore 文件

编译完 u-boot 后，通过 git st 查看发现全是编译的中间文件，这些文件根本不需使用版本库进行管理，因此我们要忽略它们。  
随便解压一个 linux 内核，拷贝内核的忽略规则

    user@vmware:~/mini2440/u-boot-1.1.6$ cd ../
    user@vmware:~/mini2440$ tar jxf linux-2.6.22.6.tar.bz2
    user@vmware:~/mini2440$ cd u-boot-1.1.6/
    user@vmware:~/mini2440/u-boot-1.1.6$ cp ../linux-2.6.22.6/.gitignore ./

将 kernel 默认的忽略文件拷贝到 u-boot 源码中，很好，大部分的中间文件都成功忽略了，但是通过 git st 查看还是有部分中间文件没有被忽略

    user@vmware:~/mini2440/u-boot-1.1.6$ git st
    On branch master
    Untracked files:
      (use "git add <file>..." to include in what will be committed)
    
    	examples/hello_world
    	examples/hello_world.bin
    	examples/hello_world.srec
    	include/asm-arm/arch
    	include/asm-arm/proc
    	include/bmp_logo.h
    	include/config.h
    	include/config.mk
    	include/version_autogenerated.h
    	tools/bmp_logo
    	tools/crc32.c
    	tools/envcrc
    	tools/environment.c
    	tools/gen_eth_addr
    	tools/img2srec
    	tools/mkimage
    	u-boot
    	u-boot.bin
    	u-boot.map
    	u-boot.srec
    
    nothing added to commit but untracked files present (use "git add" to track)

将这些文件手动添加到 .gitignore 文件中，值得注意的是，内核默认的忽略规则会将 . 开头的隐藏文件一并忽略掉，这里需要强制指定不忽略 .gitignore 文件
注意：`diff` 命令的 `<` 相当于 `git diff` 的 `+`，`>` 相当于 `git diff` 的 `-`

    user@vmware:~/mini2440/u-boot-1.1.6$ vim .gitignore
    user@vmware:~/mini2440/u-boot-1.1.6$ diff .gitignore ../linux-2.6.22.6/.gitignore
    5,29d4
    < 
    < #
    < # maiot added rules
    < #
    < examples/hello_world
    < examples/hello_world.bin
    < examples/hello_world.srec
    < include/asm-arm/arch
    < include/asm-arm/proc
    < include/bmp_logo.h
    < include/config.h
    < include/config.mk
    < include/version_autogenerated.h
    < tools/bmp_logo
    < tools/crc32.c
    < tools/envcrc
    < tools/environment.c
    < tools/gen_eth_addr
    < tools/img2srec
    < tools/mkimage
    < u-boot
    < u-boot.bin
    < u-boot.map
    < u-boot.srec
    < 
    34,35d8
    < # don't ignore the .gitignore file
    < !.gitignore

此时成功将编译的中间文件全部过滤

    user@vmware:~/mini2440/u-boot-1.1.6$ git st
    On branch master
    Untracked files:
      (use "git add <file>..." to include in what will be committed)
    
    	.gitignore
    
    nothing added to commit but untracked files present (use "git add" to track)

很好，提交忽略规则到版本库中

    user@vmware:~/mini2440/u-boot-1.1.6$ git add --all
    user@vmware:~/mini2440/u-boot-1.1.6$ git commit -m "conf: add the gitignore rule"

#### 1.4 备份源码到 github 方便下次使用

在 github 上创建一个空的仓库，我创建的仓库地址为：`git@github.com:mini2440/100ask.u-boot-1.1.6.git`  
查看目前本地仓库的状态，确认到并没有绑定远程仓库

    user@vmware:~/mini2440/u-boot-1.1.6$ git remote -v

为本地仓库添加远程仓库

    user@vmware:~/mini2440/u-boot-1.1.6$ git remote add origin git@github.com:mini2440/100ask.u-boot-1.1.6.git
    user@vmware:~/mini2440/u-boot-1.1.6$ git remote -v
    origin	git@github.com:mini2440/100ask.u-boot-1.1.6.git (fetch)
    origin	git@github.com:mini2440/100ask.u-boot-1.1.6.git (push)

将本地仓库上传到 github 远程仓库中

    user@vmware:~/mini2440/u-boot-1.1.6$ git push origin -u master

#### 1.5 使用 github 仓库快速获取源码

直接克隆远程仓库配置编译即可  
注意：成功克隆工程后，您需要修改 Makefile 中交叉编译器路径为您的主机上编译器安装的实际路径。

    user@vmware:~/mini2440$ git clone git@github.com:mini2440/100ask.u-boot-1.1.6.git
    user@vmware:~/mini2440$ cd 100ask.u-boot-1.1.6/
    user@vmware:~/mini2440/100ask.u-boot-1.1.6$ make 100ask24x0_config
    user@vmware:~/mini2440/100ask.u-boot-1.1.6$ make
    user@vmware:~/mini2440/100ask.u-boot-1.1.6$ ls -l u-boot.bin
    -rwxrwxr-x 1 user user 198324 5月  20 13:15 u-boot.bin

### 二 配置编译 linux 内核

#### 2.1 获取 linux 内核源码并打补丁

在上面 u-boot 中已经解压好了内核源码，这里直接创建仓库就好

    user@vmware:~/mini2440$ cd linux-2.6.22.6/
    user@vmware:~/mini2440/linux-2.6.22.6$ git init
    Initialized empty Git repository in /home/user/mini2440/linux-2.6.22.6/.git/
    user@vmware:~/mini2440/linux-2.6.22.6$ git add --all
    user@vmware:~/mini2440/linux-2.6.22.6$ git commit -m "init: upload the linux-2.6.22.6 source code by 100ask.net"

给 kernel 打补丁

    user@vmware:~/mini2440/linux-2.6.22.6$ patch -p1 < ../linux-2.6.22.6_jz2440.patch
    user@vmware:~/mini2440/linux-2.6.22.6$ git add --all
    user@vmware:~/mini2440/linux-2.6.22.6$ git commit -m "patch: linux-2.6.22.6_jz2440.patch"


#### 2.2 修改编译器

修改顶层 Makefile 指定编译使用刚刚解压的交叉编译器，修改点如下：

    user@vmware:~/mini2440/linux-2.6.22.6$ vim Makefile 
    user@vmware:~/mini2440/linux-2.6.22.6$ git diff
    diff --git a/Makefile b/Makefile
    index 9b456d0..e98b548 100644
    --- a/Makefile
    +++ b/Makefile
    @@ -184,7 +184,7 @@ SUBARCH := $(shell uname -m | sed -e s/i.86/i386/ -e s/sun4u/sparc64/ \
     
     #ARCH          ?= $(SUBARCH)
     ARCH           ?= arm
    -CROSS_COMPILE  ?= arm-linux-
    +CROSS_COMPILE  ?= /home/user/mini2440/gcc-3.4.5-glibc-2.3.6/bin/arm-linux-
     
     # Architecture as present in compile.h
     UTS_MACHINE := $(ARCH)
    user@vmware:~/mini2440/linux-2.6.22.6$ git add --all
    user@vmware:~/mini2440/linux-2.6.22.6$ git commit -m "conf: modify the cross compiler"


#### 2.3 配置编译内核

直接使用 100ask 提供的配置文件配置内核

    user@vmware:~/mini2440/linux-2.6.22.6$ cp config_ok .config

执行 make 命令开始编译

    user@vmware:~/mini2440/linux-2.6.22.6$ make uImage
    Makefile:1451: *** mixed implicit and normal rules: deprecated syntax
    /home/user/mini2440/linux-2.6.22.6/Makefile:418: *** mixed implicit and normal rules: deprecated syntax
    /home/user/mini2440/linux-2.6.22.6/Makefile:1451: *** mixed implicit and normal rules: deprecated syntax
    make[1]: *** No rule to make target 'silentoldconfig'.  Stop.
      CHK     include/linux/version.h
    make: *** No rule to make target 'include/config/auto.conf', needed by 'include/asm-arm/.arch'.  Stop.

很遗憾，报错了，百度得到的结论是由于我的系统的 make 工具太新，新工具不兼容旧版规则，参考[Maxwell的博客](https://blog.csdn.net/sinat_24088685/article/details/51009472)修改，修改点如下：

    user@vmware:~/mini2440/linux-2.6.22.6$ vim Makefile +416
    user@vmware:~/mini2440/linux-2.6.22.6$ git diff
    diff --git a/Makefile b/Makefile
    index d6806d0..a52ab44 100644
    --- a/Makefile
    +++ b/Makefile
    @@ -415,7 +415,7 @@ ifeq ($(config-targets),1)
     include $(srctree)/arch/$(ARCH)/Makefile
     export KBUILD_DEFCONFIG
     
    -config %config: scripts_basic outputmakefile FORCE
    +%config: scripts_basic outputmakefile FORCE
            $(Q)mkdir -p include/linux include/config
            $(Q)$(MAKE) $(build)=scripts/kconfig $@
     
    @@ -1448,7 +1448,7 @@ endif
            $(Q)$(MAKE) $(build)=$(build-dir) $(target-dir)$(notdir $@)
     
     # Modules
    -/ %/: prepare scripts FORCE
    +%/: prepare scripts FORCE
            $(Q)$(MAKE) KBUILD_MODULES=$(if $(CONFIG_MODULES),1) \
            $(build)=$(build-dir)
     %.ko: prepare scripts FORCE
    user@vmware:~/mini2440/linux-2.6.22.6$ git add Makefile 
    user@vmware:~/mini2440/linux-2.6.22.6$ git commit -m "conf: fix new make tool can't compatible the old kernel"

修改好后重新 make 试试

    user@vmware:~/mini2440/linux-2.6.22.6$ make uImage
    ... ...
      LD      arch/arm/boot/compressed/vmlinux
      OBJCOPY arch/arm/boot/zImage
      Kernel: arch/arm/boot/zImage is ready
      UIMAGE  arch/arm/boot/uImage
    "mkimage" command not found - U-Boot images will not be built
      Image arch/arm/boot/uImage is read

很遗憾 uImage 还是没有成功编译出来，提示说需要 mkimage 工具，安装 mkimage 后试试

    user@vmware:~/mini2440/linux-2.6.22.6$ sudo apt-get install u-boot-tools
    user@vmware:~/mini2440/linux-2.6.22.6$ make uImage
    ... ...
    Image Name:   Linux-2.6.22.6
    Created:      Tue May  1 19:39:40 2018
    Image Type:   ARM Linux Kernel Image (uncompressed)
    Data Size:    1812732 Bytes = 1770.25 kB = 1.73 MB
    Load Address: 30008000
    Entry Point:  30008000
      Image arch/arm/boot/uImage is ready
    user@vmware:~/mini2440/linux-2.6.22.6$ ls -l arch/arm/boot/uImage 
    -rw-rw-r-- 1 user user 1845932 5月  20 20:59 arch/arm/boot/uImage

#### 2.4 修改 lcd 驱动

由于我使用的是 mini2440 开发板，配套的 LCD 是 X35，使用 100ask 提供的 lcd 驱动无法直接点亮，因此需要修改 lcd 驱动文件。参考[comwise的博客](https://blog.csdn.net/comwise/article/details/11727445)将 X35 屏幕的驱动移植到 kenrel 中，可以点亮 LCD。  
注意：博客提供的 LCD 驱动，直接移植好后编译会报错，需要屏蔽其中三行报错的头文件。  
我将修改好的 LCD 驱动和网卡驱动放了一份在 `git@github.com:mini2440/100ask.lcd.and.net.driver.git` 仓库中，可以克隆这个仓库获取，获取仓库的命令如下

    user@vmware:~/mini2440$ git clone git@github.com:mini2440/100ask.lcd.and.net.driver.git

拷贝 LCD 驱动到内核中

    user@vmware:~/mini2440$ cp -av 100ask.lcd.and.net.driver/drivers/video/mini2440_lcd_x35.c linux-2.6.22.6/drivers/video/
    '100ask.lcd.and.net.driver/drivers/video/mini2440_lcd_x35.c' -> 'linux-2.6.22.6/drivers/video/mini2440_lcd_x35.c'

修改驱动目录下的 Makefile 使驱动被编译到

    user@vmware:~/mini2440/linux-2.6.22.6$ vim drivers/video/Makefile
    user@vmware:~/mini2440/linux-2.6.22.6$ git diff
    diff --git a/drivers/video/Makefile b/drivers/video/Makefile
    old mode 100644
    new mode 100755
    index bd8b052..6f40cae
    --- a/drivers/video/Makefile
    +++ b/drivers/video/Makefile
    @@ -106,7 +106,11 @@ obj-$(CONFIG_FB_MAXINE)    += maxinefb.o
     obj-$(CONFIG_FB_TX3912)                        += tx3912fb.o
     obj-$(CONFIG_FB_S1D13XXX)                      += s1d13xxxfb.o
     obj-$(CONFIG_FB_IMX)                           += imxfb.o
    -obj-$(CONFIG_FB_S3C2410)                       += s3c2410fb.o
    +
    +# mz8023yt@163.com 20180520 begin >>> [2/2] realize the mini2440 x35 lcd driver
    +obj-$(CONFIG_FB_S3C2410)                       += mini2440_lcd_x35.o
    +# mz8023yt@163.com 20180520 end   <<< [2/2] realize the mini2440 x35 lcd driver
    +
     obj-$(CONFIG_FB_PNX4008_DUM)                   += pnx4008/
     obj-$(CONFIG_FB_PNX4008_DUM_RGB)               += pnx4008/
     obj-$(CONFIG_FB_IBM_GXT4500)                   += gxt4500.o
    user@vmware:~/mini2440/linux-2.6.22.6$ git add --all
    user@vmware:~/mini2440/linux-2.6.22.6$ git commit -m "feature: realize the mini2440 x35 lcd driver"

重新 make

    user@vmware:~/mini2440/linux-2.6.22.6$ make uImage


#### 2.5 修改网卡驱动

同理，修改网卡也是一样的，拷贝网卡驱动到内核中

    user@vmware:~/mini2440$ cp -av 100ask.lcd.and.net.driver/drivers/net/mini2440_dm9000.c linux-2.6.22.6/drivers/net/
    '100ask.lcd.and.net.driver/drivers/net/mini2440_dm9000.c' -> 'linux-2.6.22.6/drivers/net/mini2440_dm9000.c'

修改驱动目录下的 Makefile 使驱动被编译到

    user@vmware:~/mini2440/linux-2.6.22.6$ git diff
    diff --git a/drivers/net/Makefile b/drivers/net/Makefile
    old mode 100644
    new mode 100755
    index e819674..0e714ce
    --- a/drivers/net/Makefile
    +++ b/drivers/net/Makefile
    @@ -194,7 +194,11 @@ obj-$(CONFIG_S2IO)     += s2io.o
     obj-$(CONFIG_MYRI10GE)                     += myri10ge/
     obj-$(CONFIG_SMC91X)                       += smc91x.o
     obj-$(CONFIG_SMC911X)                      += smc911x.o
    -obj-$(CONFIG_DM9000)                       += dm9dev9000c.o
    +
    +# mz8023yt@163.com 20180520 begin >>> [2/2] realize the dm9000 net driver
    +obj-$(CONFIG_DM9000)                       += mini2440_dm9000.o
    +# mz8023yt@163.com 20180520 end   <<< [2/2] realize the dm9000 net driver
    +
     #obj-$(CONFIG_DM9000)                      += dm9000.o
     #obj-$(CONFIG_DM9000)                      += dm9ks.o
     obj-$(CONFIG_FEC_8XX)                      += fec_8xx/
    user@vmware:~/mini2440/linux-2.6.22.6$ git add --all
    user@vmware:~/mini2440/linux-2.6.22.6$ git commit -m "feature: realize the mini2440 x35 lcd driver"

重新 make

    user@vmware:~/mini2440/linux-2.6.22.6$ make uImage

#### 2.6 提交忽略规则文件

修改 .gitignore 文件，在 `.*` 后追加 `!.gitignore`，不忽略 .gitignore 文件。

    user@vmware:~/mini2440$ vim .gitignore
     #
     # Normal rules
     #
     .*
    +
    +# don't ignore the .gitignore file
    +!.gitignore
    +
     *.o
    user@vmware:~/mini2440/linux-2.6.22.6$ git add --all
    user@vmware:~/mini2440/linux-2.6.22.6$ git commit -m "conf: add the git ignore files"


#### 2.7 备份源码到 github 方便下次使用

在 github 上创建一个空的仓库，我创建的仓库地址为：`git@github.com:mini2440/100ask.linux-2.6.22.6.git`  
查看目前本地仓库的状态，确认到并没有绑定远程仓库

    user@vmware:~/mini2440/linux-2.6.22.6$ git remote -v

为本地仓库添加远程仓库

    user@vmware:~/mini2440/linux-2.6.22.6$ git remote add origin git@github.com:mini2440/100ask.linux-2.6.22.6.git
    user@vmware:~/mini2440/linux-2.6.22.6$ git remote -v
    origin	git@github.com:mini2440/100ask.linux-2.6.22.6.git (fetch)
    origin	git@github.com:mini2440/100ask.linux-2.6.22.6.git (push)

将本地仓库上传到 github 远程仓库中

    user@vmware:~/mini2440/linux-2.6.22.6$ git push origin -u master

#### 2.8 使用 github 仓库快速获取源码

    user@vmware:~/mini2440$ git clone git@github.com:mini2440/100ask.linux-2.6.22.6.git
    user@vmware:~/mini2440$ cd 100ask.linux-2.6.22.6/
    user@vmware:~/mini2440/100ask.linux-2.6.22.6$ cp config_ok .config
    user@vmware:~/mini2440/100ask.linux-2.6.22.6$ make uImage

### 三 重烧整个系统

#### 3.1 下载 u-boot 到 nandflash 上

将 u-boot.bin 拷贝到 windows 目录下，这里我拷贝到 `D:\work\` 目录下，使用 oflash 工具下载到 nandflash 中

```
Microsoft Windows [版本 6.1.7600]
版权所有 (c) 2009 Microsoft Corporation。保留所有权利。

C:\Users\Administrator>d:

D:\>cd work

D:\work>oflash 0 1 0 0 0 u-boot.bin

+---------------------------------------------------------+
|   Flash Programmer v1.5.2 for OpenJTAG of www.100ask.net  |
|   OpenJTAG is a USB to JTAG & RS232 tool based FT2232   |
|   This programmer supports both of S3C24X0 & S3C6410    |
|   Author: Email/MSN(thisway.diy@163.com), QQ(17653039)  |
+---------------------------------------------------------+
Usage:
1. oflash, run with cfg.txt or prompt
2. oflash [file], write [file] to flash with prompt
3. oflash [-f config_file]
4. oflash [jtag_type] [cpu_type] [flash_type] [read_or_write] [offset] [file]
Select the JTAG type:
0. OpenJTAG
1. Dongle JTAG(parallel port)
2. Wiggler JTAG(parallel port)
Enter the number: 0
Select the CPU:
0. S3C2410
1. S3C2440
2. S3C6410
Enter the number: 1

device: 4 "2232C"
deviceID: 0x14575118
SerialNumber: FThecwJmA
Description: USB<=>JTAG&RS232 AS3C2440 detected, cpuID = 0x0032409d

[Main Menu]
 0:Nand Flash prog     1:Nor Flash prog   2:Memory Rd/Wr     3:Exit
Select the function to test:0

[NAND Flash JTAG Programmer]
Scan nand flash:
Device 0: NAND 256MiB 3,3V 8-bit, sector size 128 KiB
Total size: 256 MiB
 0:Nand Flash Program      1:Nand Flash Print BlkPage   2:Exit
Select the function to test :0

[NAND Flash Writing Program]

Source size: 0x306b4

Available target block number: 0~2047
Input target block number:0
target start block number     =0
target size        (0x20000*2) =0x40000
STATUS:
Epppppppppppppppppppppppppppppppppppppppppppppppppppppppppppppppp
Eppppppppppppppppppppppppppppppppp
```

#### 3.2 使用 nandflash 启动开发板

成功下载 u-boot.bin 到 nandflash 后，将启动选项设置为 nand 启动，上电有以下 log 输出说明 u-boot 成功运行

```
U-Boot 1.1.6-g48373b63 (May 17 2018 - 22:46:07)

DRAM:  64 MB
Flash:  0 kB
NAND:  256 MiB
In:    serial
Out:   serial
Err:   serial
UPLLVal [M:38h,P:2h,S:2h]
MPLLVal [M:5ch,P:1h,S:1h]
CLKDIVN:5h


+---------------------------------------------+
| S3C2440A USB Downloader ver R0.03 2004 Jan  |
+---------------------------------------------+
USB: IN_ENDPOINT:1 OUT_ENDPOINT:3
FORMAT: <ADDR(DATA):4>+<SIZE(n+10):4>+<DATA:n>+<CS:2>
NOTE: Power off/on or press the reset button for 1 sec
      in order to get a valid USB device address.

Hit any key to stop autoboot:  0 

##### 100ask Bootloader for OpenJTAG #####
[n] Download u-boot to Nand Flash
[k] Download Linux kernel uImage
[j] Download root_jffs2 image
[y] Download root_yaffs image
[d] Download to SDRAM & Run
[z] Download zImage into RAM
[g] Boot linux from RAM
[f] Format the Nand Flash
[s] Set the boot parameters
[b] Boot the system
[r] Reboot u-boot
[q] Quit from menu
Enter your selection: q
OpenJTAG> 
```

#### 3.3 下载 kernel 到 nandflash 上

u-boot 成功烧写后，便可以使用 u-boot 的 tftp 命令可以下载文件到 SDRAM 中。

使用 tftp 命令下载的前提是：
1. 开发板和 Windows 主机通过网线接在同一个路由器上
2. Windows 主机上启动了 TPTP 服务

配置好 Windows 主机的 ip，这里我的 Windows 主机的 ip 配置为 192.168.1.5。Windows 主机行运行 tftp 服务器软件，服务器 ip 默认就是 Windows 主机 ip。同时将要通过通过 tftp 下载到开发板的文件拷贝到 tftpd32.exe 服务器软件同级目录下。这里我拷贝了内核文件 uImage 和文件系统 fs_qtopia.yaffs2 到 tftp 目录下。  

依次执行以下步骤将 kernel 烧写到 nandflash 中：
1. 配置好开发板的 ip 以及 tftp 服务器 ip  
   上电后，按下空格进入 U-Boot，执行下面的命令设置环境变量中开发板的 ip 为 192.168.1.8，指定 tftp 服务器的 ip 为 192.168.1.5。

       OpenJTAG> setenv ipaddr 192.168.1.100
       OpenJTAG> setenv serverip 192.168.1.5
       OpenJTAG> save
       OpenJTAG> reset

2. 重启 U-Boot 之后使用 tftp 命令将 tftp 服务器中的文件下载到 SDRAM 0x30000000 地址处  

       OpenJTAG> tftp 30000000 uImage

3. 擦除 kernel 分区后再烧写到 nandflash 中
   所谓的 kernel 分区其实只是 nandfalsh 中 0x00060000 - 0x00260000 这段地址空间

       OpenJTAG> nand erase kernel

   接下来要烧写到 nandflash 中，下面两条命令效果一致，二选一即可

       OpenJTAG> nand write.jffs2 30000000 kernel
       OpenJTAG> nand write.jffs2 30000000 60000 200000

   这条命令的意思是：将 SDRAM 中 0x30000000 地址开始连续 0x200000 个字节写入 nandflash 0x60000 地址中去

#### 3.4 下载文件系统到 nandflash 上

同样的方式，执行下列命令可以将文件系统烧写到 nandflash 中

    OpenJTAG> tftp 30000000 fs_qtopia.yaffs2
    OpenJTAG> nand erase root

烧写到 nandflash 中，下面三条命令效果一致，三选一即可

    OpenJTAG> nand write.yaffs 30000000 root
    OpenJTAG> nand write.yaffs 30000000 260000 $(filesize)
    OpenJTAG> nand write.yaffs 30000000 260000 2f76b40

上面烧写命令的含义其实就是，将 SDRAM 中 0x30000000 地址开始连续 0x2f76b40(49769280) 个字节写入 nandflash 0x260000 地址中去。不难发现其实 2f76b40 这个值就是 fs_qtopia.yaffs2 文件的大小。



