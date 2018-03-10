---
title: '[Linux] 内核启动流程分析'
date: 2018-03-10 22:46:39
tags:
  - boot
categories: Linux
---


## 一. 内核编译初体验

### 1.1 获取 Linux 内核

视频和资料可以访问[百问网](http://www.100ask.net/bbs/forum.php)论坛获取。  
下载好资料后在 "JZ2440资料光盘" 文件中可以获取到内核和对应的补丁文件。

1. 内核源码包：光盘/system/linux-2.6.22.6.tar.bz2
2. 内核补丁包：光盘/system/linux-2.6.22.6_jz2440.patch。

### 1.2 解压内核并给内核打补丁

将获取到的内核文件放在虚拟机 ~/workspace/jz2440/package 目录下。

```bash
user@vmware:~/workspace/jz2440/package$ tar -jxf linux-2.6.22.6.tar.bz2 -C ../
user@vmware:~/workspace/jz2440/package$ cd ../linux-2.6.22.6/
user@vmware:~/workspace/jz2440/linux-2.6.22.6$ patch -p1 < ../package/linux-2.6.22.6_jz2440.patch
```

先介绍一下打补丁的方式：

```
patch -pnum < <补丁文件>
```

打到哪里补丁文件中会指出，-p 选项则表示忽略补丁文件的前级目录，num 表示忽略的目录级数。

### 1.3 配置内核

#### 方式一：通过 menu 菜单直接配置

使用这种方式，从头到尾每一项都要我们自己配置。

```bash
make menuconfig
```

#### 方式二：使用默认配置文件配置，默认配置的文件位于：arch/arm/configs。

```bash
make s3c2440_defconfig
make menuconfig
```

#### 方式三：使用厂家提供的配置文件配置。

```bash
cp config_ok .config
make menuconfig
```

所有的配置目的与结果都是生成一个 .config 文件，看看 .config 都有什么东西？  
随便截取一段出来看看：

```bash
#
# Ethernet (10 or 100Mbit)
#
CONFIG_NET_ETHERNET=y
CONFIG_MII=y
# CONFIG_SMC91X is not set
CONFIG_DM9000=y
# CONFIG_CS89x0 is not set
```

不难看出 .config 文件中其实就是很多配置项，可以配置为 y、m 和数值，甚至有很多的配置项被注释掉了，后面跟着 is not set 字符串。  
为什么 is not set 格式还有必须要注释？这样约定格式有什么意义？

... ... 待补充

### 1.4 开始编译内核

直接 make 就可以执行编译的过程了，将会在 arch/arm/boot/zImage。但实际编译过程可能由于使用的交叉编译器工具链的区别会出现一些错误，请自行百度解决。  
或者使用 make uImage 编译生成 uImage，那么什么是 uImage？  
zImage 是 ARM Linux 常用的一种压缩映像文件，而 uImage 是 U-boot 专用的映像文件，它是在 zImage 之前加上一个长度为 0x40 的"头部"，说明这个映像文件的类型、加载位置、生成时间、大小等信息。换句话说，如果直接从 uImage 的 0x40 位置开始执行，zImage 和 uImage没有任何区别。  
另外，Linux2.4 内核不支持 uImage，Linux2.6 内核加入了很多对嵌入式系统的支持，uImage 在 u-boot 里用 bootm 加载，而 zImage 用 go 加载。

## 二. 内核实现剪裁的方式？

### 2.1 追踪分析 CONFIG_DM9000 配置项

我们都知道 Linux 是一个可剪裁的内核，内核具备哪些功能，我们可以手动配置，而配置内核我们之前已经了解过了，通过三种方式，最终通过 .config 文件确认内核具备哪些功能。  
我们以 CONFIG_DM9000 为例，看看内核是如何通过 .config 实现功能剪裁的。在 kernel 中 grep 搜索一下，看看 CONFIG_DM9000 这个宏出现在哪些文件中。

我们给出现 CONFIG_DM9000 宏的文件分个类，大概有此 4 类：

 - C 语言源码
 - 子目录的 Makefile
 - include/config/auto.conf
 - include/linux/autoconf.h  

源码中使用到的宏定义只能是定义在 c 文件或者头文件中的，那很显然，源码中使用的宏定义就是在 include/linux/autoconf.h 文件中定义的。  
看看这个文件的内容，仍然以 CONFIG_DM9000 为例，截取一部分看看：

```c
#define CONFIG_DM9000 1
#define CONFIG_SOLARIS_X86_PARTITION 1
#define CONFIG_SERIAL_NONSTANDARD 1
```

如果你仔细的对比 .config 和 autoconf.h 文件，很容易发现这个规律：
 - 配置为 y 的选项，在 autoconf.h 文件中被定义成为了一个值为 1 的宏。
 - 配置为 m 的选项，在 autoconf.h 文件中没有被定义为宏。
 - 配置为数值的选项，在 autoconf.h 文件中被定义成为了一个值为对应数值的宏。

接下来我们看看字目录下的 Makefile，在此此前先插播一下子目录的 makefile 的规则。  
子目录下的 Makefile 通常都是将源文件加入到编译命令中。一般的格式是这样的：

```
obj-y += yyy.o   # 静态方式编译进内核
obj-m += mmm.o   # 模块方式编译进内核
```

因此，子目录的 Makefile 多半是这样的(继续以 CONFIG_DM9000 为例)：

```
obj-$(CONFIG_DM9000) += dm9dev9000c.o
```

很显然，子目录的 Makefile 是使用了 CONFIG_DM9000 变量，那 CONFIG_DM9000 变量是在哪里定义的呢？  
看来看去也就只能是在 include/config/auto.conf 文件定义了，但是 Makefile 中只能引用 Makefile 中定义的变量，这个 auto.conf 又是什么东西？子目录中如果真的引用了 auto.conf 文件的内容，那就只有一种可能，某个 Makefile 将 auto.conf 文件包含进 Makefile 中了。  
没错，还真的是这样，看看顶层 Makefile：

```
-include include/config/auto.conf
```

这里尝试包含 auto.conf 文件。由于使用 -include 进行声明，所以即使这两个文件不存在也不会报错退出。

### 2.2 必要的总结一下

配置的目的只有一个，根据我们想要内核具备的功能，通过配置项的方式，配置好并保存在 .config 文件中。  
而在编译的时候，会自动的创建 auto.conf 和 autoconf.h 两个文件，一个在源码中作为宏控开关量，控制着哪些代码被编译；一个在 Makefile 控制做为宏控开关量，控制着哪些文件被编译进内核。

## 三. 内核从哪里开始执行

从 Makefile 开始分析，在顶层 Makefile 里面看看 make uImage 和 make 的过程：

```
# make uImage
zImage Image xipImage bootpImage uImage: vmlinux

# make
all: vmlinux
```

可以发现都是依赖 vmlinux，接下来再看看 vmlinux 依赖些什么？

```
vmlinux: $(vmlinux-lds) $(vmlinux-init) $(vmlinux-main) $(kallsyms.o) FORCE

vmlinux-init := $(head-y) $(init-y)

head-y		:= arch/arm/kernel/head.o arch/arm/kernel/init_task.o

init-y		:= init/
init-y		:= $(patsubst %/, %/built-in.o, $(init-y))
最后会将 init 目录下的所有文件编译成为一个 init/built-in.o

vmlinux-main := $(core-y) $(libs-y) $(drivers-y) $(net-y)
core-y		:= usr/
core-y		+= kernel/ mm/ fs/ ipc/ security/ crypto/ block/
core-y		:= $(patsubst %/, %/built-in.o, $(core-y))
最后编译结果为 usr/built-in.o kernel/built-in.o mm/built-in.o fs/built-in.o ipc/built-in.o security/built-in.o crypto/built-in.o block/built-in.o

libs-y		:= lib/
libs-y		:= arch/arm/lib/ $(libs-y)
libs-y		:= $(libs-y1) $(libs-y2)
libs-y1		:= $(patsubst %/, %/lib.a, $(libs-y))
libs-y2		:= $(patsubst %/, %/built-in.o, $(libs-y)
最后编译结果为 lib/lib.a lib/built-in.o

drivers-y	:= drivers/ sound/
drivers-y	:= $(patsubst %/, %/built-in.o, $(drivers-y)
最后编译结果为 drivers/built-in.o sound/built-in.o

net-y		:= net/
net-y		:= $(patsubst %/, %/built-in.o, $(net-y))
最后编译结果为 net/built-in.o

vmlinux-lds  := arch/$(ARCH)/kernel/vmlinux.lds
链接脚本位置
```

可以看出，vmlinux 将内核中的各个目录下编译出来的 .o 文件分成了几大类，最终几乎将所有的代码包含在 vmlinux 中。  
Make 追踪下来回比较麻烦，其实还有更简单粗暴的方法，直接根据 make 命令的控制台输出确认。  
通过在 make uImage 命令加一个 V=1 的属性值打印更详细的 log 信息。

```bash
user@vmware:~/workspace/jz2440/linux-2.6.22.6$ rm -rf vmlinux
user@vmware:~/workspace/jz2440/linux-2.6.22.6$ make uImage V=1
  arm-linux-ld -EL  -p --no-undefined -X -o vmlinux -T arch/arm/kernel/vmlinux.lds arch/arm/kernel/head.o arch/arm/kernel/init_task.o init/built-in.o --start-group usr/built-in.o arch/arm/kernel/built-in.o arch/arm/mm/built-in.o arch/arm/common/built-in.o arch/arm/mach-s3c2410/built-in.o arch/arm/mach-s3c2400/built-in.o arch/arm/mach-s3c2412/built-in.o arch/arm/mach-s3c2440/built-in.o arch/arm/mach-s3c2442/built-in.o arch/arm/mach-s3c2443/built-in.o arch/arm/nwfpe/built-in.o arch/arm/plat-s3c24xx/built-in.o kernel/built-in.o mm/built-in.o fs/built-in.o ipc/built-in.o security/built-in.o crypto/built-in.o block/built-in.o arch/arm/lib/lib.a lib/lib.a arch/arm/lib/built-in.o lib/built-in.o drivers/built-in.o sound/built-in.o net/built-in.o --end-group .tmp_kallsyms2.o
```

最关键的一句，我没有分行，建议复制出来看看，和我们之前分析 Makefile 得出的结论是一致的。

通过上面的分析，得出两个结论：

1. 最先执行文件：arch/arm/kernel/head.S
2. 内核链接脚本：arch/arm/kernel/vmlinux.lds

## 四. 分析内核启动过程

### 4.1 建立 source insight 工程

1. 添加所有文件  
2. 删除 arch 目录  
3. 添加公用的代码  
   arch/arm/boot/  
   arch/arm/common/  
   arch/arm/configs/  
   arch/arm/kernel/  
   arch/arm/lib/  
   arch/arm/mach-s3c2410/  
   arch/arm/mach-s3c2440/  
   arch/arm/mm/  
   arch/arm/nwfpe/  
   arch/arm/oprofile/  
   arch/arm/plat-s3c24xx/  
   arch/arm/tools/  
   arch/arm/vfp/  
4. 删除 include 目录  
5. 添加非 asm 开头的目录  
   include/acpi/  
   include/config/  
   include/crypto/  
   include/Kbuild  
   include/keys/  
   include/linux/  
   include/math-emu/  
   include/media/  
   include/mtd/  
   include/net/  
   include/pcmcia/  
   include/rdma/  
   include/rxrpc/  
   include/scsi/  
   include/sound/  
   include/video/  
6. 添加 include/asm-arm/ 目录下的文件(目录不添加)  
7. 添加 include/asm-arm/ 目录下的目录  
   include/asm-arm/arch-s3c2410/  
   include/asm-arm/hardware/  
   include/asm-arm/plat-s3c24xx/  
   include/asm-arm/mach/  

