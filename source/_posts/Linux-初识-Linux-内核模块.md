---
title: '[Linux] 初识 Linux 内核模块'
date: 2018-01-22 22:59:52
tags:
  - module
categories: Linux
---

## 一. 编写一个内核模块
### 1.1 什么是内核模块
Linux 内核是模块化组成的，它允许内核在运行时动态地向其中插入或从中删除代码。这些代码被一并组合在一个单独的二进制镜像中，即所谓的可装载内核模块中，或简称为模块。支持模块的好处是基本内核镜像尽可能的小，因为可选的功能和驱动程序可以利用模块形式再提供。模块允许我们方便地删除和重新载入内核代码，也方便了调试工作。而且当热插拔新设备时，可通过命令载入新的驱动程序。  
模块是具有独立功能的程序，它可以被单独编译，但不能独立运行。它在运行时被链接到内核作为内核的一部分在内核空间运行，这与运行在用户空间的进程是不同的。模块通常由一组函数和数据结构组成，用来实现一种文件系统、一个驱动程序或其他内核上层的功能。总之，模块是一个为内核或其他内核模块提供使用功能的代码块。

### 1.2 最简单的内核模块
创建一个目录专门保存 linux 驱动代码，并开始编写第一个内核模块：

```bash
user@ubuntu:~$ mkdir -p mini2440/driver.with.linux
user@ubuntu:~$ cd mini2440/driver.with.linux/
user@ubuntu:~/mini2440/driver.with.linux$ mkdir -p module/simplest_module/
```

#### 文件一：simplest_module.c

```c
#include <linux/init.h>
#include <linux/module.h>

static int __init mod_init(void)
{
        printk(KERN_INFO "simplest module init ok!\n");
        return 0;
}

static void __exit mod_exit(void)
{
        printk(KERN_INFO "simplest module exit ok!\n");
}

module_init(mod_init);
module_exit(mod_exit);

MODULE_LICENSE("GPL");
```

#### 文件二：Makefile

```bash
obj-m += simplest_module.o
KERNEL = /home/user/workspace/mini2440/linux-2.6.32.2

all:
        make -C $(KERNEL) M=`pwd` modules

clean:
        make -C $(KERNEL) M=`pwd` modules clean
```

### 1.3 编译加载模块
执行 make 命令，编译内核模块。

```bash
user@vmware:~/mini2440/driver.with.linux/module/simplest_module/$ make
make -C /home/user/workspace/mini2440/linux-2.6.32.2 M=`pwd` modules
make[1]: Entering directory '/home/user/workspace/mini2440/linux-2.6.32.2'
  Building modules, stage 2.
  MODPOST 1 modules
make[1]: Leaving directory '/home/user/workspace/mini2440/linux-2.6.32.2'

user@vmware:~/mini2440/driver.with.linux/module/simplest_module/$ ls
Makefile       Module.symvers  simplest_module.c   simplest_module.mod.c
modules.order  readme.txt      simplest_module.ko  simplest_module.mod.o
simplest_module.o
```

生成的 simplest_module.ko 文件就是可动态加载的内核模块。  
使用 insmod 命令加载模块到内核中，将会执行 module_init(mod_init) 指定的 mod_init 函数。

```bash
[root@FriendlyARM /mnt]# insmod simplest_module.ko
simplest module init ok!
```

使用 rmmod 命令卸载模块，将会执行 module_exit(mod_exit) 指定的 mod_exit 函数。

```bash
[root@FriendlyARM /mnt]# rmmod simplest_module
simplest module exit ok!
```

使用 modprobe -r 卸载模块，同 rmmod 功能。

```bash
[root@FriendlyARM /mnt]# modprobe -r simplest_module
simplest module exit ok!
```

## 二. 带参数的内核模块
### 2.1 前言
Linux 内核允许模块在加载的时候指定参数。模块接受参数传入的机制能够根据参数的不同提供多种不同的服务，增加了模块的灵活性。  
模块参数必须使用 module_param 宏来声明，通常放在文件头部。 module_param 需要 3 个参数：变量名称、类型以及用于 sysfs 入口的访问掩码。模块最好为参数指定一个默认值，以防加载模块的时候忘记传参而带来错误。

 - 内核模块支持的参数类型有： bool、 invbool、 charp、 int、 short、 long、 uint、 ushort 和 ulong。
 - 访问掩码的值在<linux/stat.h>定义， S_IRUGO 表示任何人都可以读取该参数，但不能修改。

### 2.2 示例模块
#### 文件一：param_module.c

```c
#include <linux/init.h>
#include <linux/module.h>

char* name = "unknow";
int age = -1;

module_param(name, charp, S_IRUGO);
module_param(age, int, S_IRUGO);

static int __init mod_init(void)
{
        if(!strcmp(name, "unknow") || -1 == age)
        {
                printk(KERN_INFO "usage: insmod param_module name=<str> age=<int>\n");
                return -1;
        }

        printk(KERN_INFO "hello %s, your age is %d\n", name, age);
        printk(KERN_INFO "param module init ok!\n");
        return 0;
}

static void __exit mod_exit(void)
{
        printk(KERN_INFO "param module exit ok!\n");
}

module_init(mod_init);
module_exit(mod_exit);

MODULE_LICENSE("GPL");
```

#### 文件二：Makefile

```bash
obj-m += param_module.o
KERNEL = /home/user/workspace/mini2440/linux-2.6.32.2
SHARE = /home/user/board

all:
    make -C $(KERNEL) M=`pwd` modules
    rm -rf $(SHARE)/*
    cp -f *.ko $(SHARE)/
    make clean
    @echo "\033[31m >>>>>> make all successful <<<<<< \033[0m"
clean:
    make -C $(KERNEL) M=`pwd` modules clean
    rm -f app
```

### 2.3 加载模块
不带参数加载，加载失败，提示加载参数使用方法：

```bash
[root@FriendlyARM /mnt]# insmod param_module.ko
usage: insmod param_module name=<str> age=<int>
insmod: cannot insert 'param_module.ko': Operation not permitted
```

带参数加载，加载成功：

```bash
[root@FriendlyARM /mnt]# insmod param_module.ko name="bean" age=20
hello bean, your age is 20
param module init ok!
[root@FriendlyARM /mnt]# rmmod param_module
param module exit ok!
```

## 三. 多个c文件编译成一个模块
### 3.1 前言
有的时候，一个驱动模块设计到功能过多，都写在一个文件中导致单个文件过大，文件结构复杂，不便于阅读和修改。那么如何将多个文件编译成为一个驱动模块呢?  
单个文件 main.c 编译成模块，Makefile 中只需要指定 obj-m 为对应的 .o 文件即可。

```bash
obj-m += main.o
```

多个文件 tp_touch.c tp_gesture.c tp_firmware.c 编译成模块 tp_mstar.ko，需要这么干：

```bash
obj-m += tp_mstar.o
tp_mstar-objs = tp_touch.o tp_gesture.o tp_firmware.o
```

### 3.2 示例模块
#### 文件一：tp_common.h

```c
#ifndef __TP_COMMON_H__
#define __TP_COMMON_H__

#include <linux/init.h>
#include <linux/module.h>

void tp_print_support_gesture(void);
void tp_print_firmware_version(void);

#endif /* __TP_COMMON_H__ */
```

#### 文件二：tp_touch.c

```c
#include "tp_common.h"

static int __init mod_init(void)
{
        tp_print_support_gesture();
        tp_print_firmware_version();
        printk(KERN_INFO "touch module init ok!\n");
        return 0;
}

static void __exit mod_exit(void)
{
        printk(KERN_INFO "touch module exit ok!\n");
}

module_init(mod_init);
module_exit(mod_exit);

MODULE_LICENSE("GPL");
```

#### 文件三：tp_firmware.c

```c
#include "tp_common.h"

void tp_print_firmware_version(void)
{
        printk(KERN_INFO "msg2836 tp firmware version = 0x10.0x06!\n");
}
```

#### 文件四：tp_gesture.c

```c
#include "tp_common.h"

void tp_print_support_gesture(void)
{
        printk(KERN_INFO "msg2836 tp support C M W O V e gesture\n");
}
```

#### 文件五：Makefile

```bash
obj-m += tp_mstar.o
tp_mstar-objs = tp_touch.o tp_gesture.o tp_firmware.o
KERNEL = /home/user/workspace/mini2440/linux-2.6.32.2
SHARE = /home/user/board

all:
    make -C $(KERNEL) M=`pwd` modules
    rm -rf $(SHARE)/*
    cp -f *.ko $(SHARE)/
    make clean
    @echo "\033[31m >>>>>> make all successful <<<<<< \033[0m"
clean:
    make -C $(KERNEL) M=`pwd` modules clean
    rm -f app
```

### 3.3 加载模块
加载模块，不难发现，三个文件被编译成了一个模块，三个文件的 log 都打印出来了。

```bash
[root@FriendlyARM /mnt]# insmod tp_mstar.ko
the matar msg2836 tp support C M W O V e gesture
the matar msg2836 tp firmware version = 0x10.0x06!
touch module init ok
[root@FriendlyARM /mnt]# rmmod tp_mstar
touch module exit ok!
```

## 四. 模块导出符号给另一个模块使用
### 4.1 前言
对于模块文件中定义的函数和变量，其他模块如果想要引用的话，被引用的函数或者变量需要使用 EXPORT_SYMBOL(x) 宏导出符号，其中 x 是导出的函数名或者变量名，而引用这个函数或者变量的模块做一下外部声明即可使用。

### 4.2 示例模块
#### 文件一：test_module.c

```c
#include <linux/init.h>
#include <linux/module.h>

extern int calculate_add(int x, int y);
extern int calculate_sub(int x, int y);

static int __init mod_init(void)
{
        printk(KERN_INFO "[%s][%d] 10 - 5 = %d\n", __func__, __LINE__, calculate_add(10, 5));
        printk(KERN_INFO "[%s][%d] 10 - 5 = %d\n", __func__, __LINE__, calculate_sub(10, 5));
        printk(KERN_INFO "test module init ok!\n");
        return 0;
}

static void __exit mod_exit(void)
{
        printk(KERN_INFO "test module exit ok!\n");
}

module_init(mod_init);
module_exit(mod_exit);

MODULE_LICENSE("GPL");
```

#### 文件二：calculate.c

```c
#include <linux/init.h>
#include <linux/module.h>

int calculate_add(int x, int y)
{
        printk(KERN_INFO "[%s][%d] is running\n", __func__, __LINE__);
        return x + y;
}
EXPORT_SYMBOL(calculate_add);

int calculate_sub(int x, int y)
{
        printk(KERN_INFO "[%s][%d] is running\n", __func__, __LINE__);
        return x - y;
}
EXPORT_SYMBOL(calculate_sub);

static int __init mod_init(void)
{
        printk(KERN_INFO "calculate module init ok!\n");
        return 0;
}

static void __exit mod_exit(void)
{
        printk(KERN_INFO "calculate module exit ok!\n");
}

module_init(mod_init);
module_exit(mod_exit);

MODULE_LICENSE("GPL");
```

#### 文件三：Makefile

```bash
obj-m += test_module.o
obj-m += calculate.o
KERNEL = /home/user/workspace/mini2440/linux-2.6.32.2
SHARE = /home/user/board

all:
    make -C $(KERNEL) M=`pwd` modules
    rm -rf $(SHARE)/*
    cp -f *.ko $(SHARE)/
    make clean
    @echo "\033[31m >>>>>> make all successful <<<<<< \033[0m"
clean:
    make -C $(KERNEL) M=`pwd` modules clean
    rm -f app
```

### 4.3 加载模块
若一个模块(模块A)引用了另一个模块(模块B)导出的符号，那么要求加载模块A的时候，模块B就已经被加载进内核了。否则的话，模块A将无法成功加载进内核。

```bash
[root@FriendlyARM /mnt]# insmod test_module.ko
test_module: Unknown symbol calculate_add
test_module: Unknown symbol calculate_sub
insmod: cannot insert 'test_module.ko': unknown symbol in module or invalid parameter
```

可以看出模块 test_module 没有加载成功，提示说找不到一些符号。

```bash
[root@FriendlyARM /mnt]# insmod calculate.ko
calculate module init ok!
[root@FriendlyARM /mnt]# insmod test_module.ko
[calculate_add][6] is running
[mod_init][9] 10 - 5 = 15
[calculate_sub][13] is running
[mod_init][10] 10 - 5 = 5
test module init ok!
```

加载依赖的 calculate 模块之后，test_module 正常加载了。
