---
title: '[Mini2440] 内核调试手段之 printk'
date: 2018-01-23 11:51:10
tags:
  - debug
categories: Mini2440
---

## 一. 内核打印函数 printk 介绍

### 1.1 前言

内核提供了 printk 函数在内核运行时打印信息，类似于 C 语言中的 printf 函数。

使用方式：

```c
printk(<日志级别> "打印内容");
```

什么是日志级别? 日志级别表示这句 log 的严重等级，总共有 8 个级别，分别是 0-7，数字越小级别越高。kernel 中可以设置屏蔽一些低级别的 log 不打印出来。  
printk 的日志级别定义在 linux-2.6.32.2/inlcude/linux/kernel.h 中：

```c
#define KERN_EMERG      "<0>"   /* system is unusable               */ 
#define KERN_ALERT      "<1>"   /* action must be taken immediately */ 
#define KERN_CRIT       "<2>"   /* critical conditions              */ 
#define KERN_ERR        "<3>"   /* error conditions                 */ 
#define KERN_WARNING    "<4>"   /* warning conditions               */ 
#define KERN_NOTICE     "<5>"   /* normal but significant condition */ 
#define KERN_INFO       "<6>"   /* informational                    */ 
#define KERN_DEBUG      "<7>"   /* debug-level messages             */ 
```

简单解释一下：

 - 紧急事件消息，系统崩溃前的提示，表示系统不可用
 - 报告事件，表示必须立刻采取措施
 - 临界条件，通常涉及严重的硬件或软件失败
 - 错误条件，驱动程序通常用此等级报告硬件错误
 - 警告条件，对可能出现的问题进行警告
 - 正常又重要的的信息，用于提醒
 - 提示信息，打印运行时的提示信息
 - 调试信息，调试级别的消息

Kernel 中可以修改 /proc/sys/kernel/printk 节点修改 printk 打印相关的配置。

```bash
[root@FriendlyARM /]# cat /proc/sys/kernel/printk
7    4    1    7
```

其实这四个值是在kernel/printk.c 中被定义的，这四个数值分别表示：

```c
int console_printk[4] = {
        DEFAULT_CONSOLE_LOGLEVEL,       /* console_loglevel         7 */
        DEFAULT_MESSAGE_LOGLEVEL,       /* default_message_loglevel 4 */
        MINIMUM_CONSOLE_LOGLEVEL,       /* minimum_console_loglevel 1 */
        DEFAULT_CONSOLE_LOGLEVEL,       /* default_console_loglevel 7 */
};
```

简单解释一下：

 - 当前控制台日志级别：优先级高于该值的消息将被打印至控制台
 - 默认的消息日志级别：将用该优先级来打印没有优先级的消息
 - 最高的控制台日志级别：控制台日志级别可被设置的最小值(最高优先级)
 - 默认的控制台日志级别：控制台日志级别的缺省值

当 printk 的消息日志级别小于当前控制台日志级别 DEFAULT_CONSOLE_LOGLEVEL 时，才会被打印出来。

### 1.2 示例模块

#### 文件一：print_level.c

```c
#include <linux/init.h>
#include <linux/module.h>

static int __init mod_init(void)
{
        printk(KERN_EMERG   "hello! KERN_EMERG   = %s\n", KERN_EMERG  );
        printk(KERN_ALERT   "hello! KERN_ALERT   = %s\n", KERN_ALERT  );
        printk(KERN_CRIT    "hello! KERN_CRIT    = %s\n", KERN_CRIT   );
        printk(KERN_ERR     "hello! KERN_ERR     = %s\n", KERN_ERR    );
        printk(KERN_WARNING "hello! KERN_WARNING = %s\n", KERN_WARNING);
        printk(KERN_NOTICE  "hello! KERN_NOTICE  = %s\n", KERN_NOTICE );
        printk(KERN_INFO    "hello! KERN_INFO    = %s\n", KERN_INFO   );
        printk(KERN_DEBUG   "hello! KERN_DEBUG   = %s\n", KERN_DEBUG  );
        return 0;
}

static void __exit mod_exit(void)
{
        printk(KERN_EMERG   "goodbye! KERN_EMERG   = %s\n", KERN_EMERG  );
        printk(KERN_ALERT   "goodbye! KERN_ALERT   = %s\n", KERN_ALERT  );
        printk(KERN_CRIT    "goodbye! KERN_CRIT    = %s\n", KERN_CRIT   );
        printk(KERN_ERR     "goodbye! KERN_ERR     = %s\n", KERN_ERR    );
        printk(KERN_WARNING "goodbye! KERN_WARNING = %s\n", KERN_WARNING);
        printk(KERN_NOTICE  "goodbye! KERN_NOTICE  = %s\n", KERN_NOTICE );
        printk(KERN_INFO    "goodbye! KERN_INFO    = %s\n", KERN_INFO   );
        printk(KERN_DEBUG   "goodbye! KERN_DEBUG   = %s\n", KERN_DEBUG  );
}

module_init(mod_init);
module_exit(mod_exit);

MODULE_LICENSE("GPL");
```

#### 文件二：Makefile

```bash
obj-m += print_level.o

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

### 1.3 加载模块

加载卸载模块打印的信息：

```bash
[root@FriendlyARM /mnt]# insmod print_level.ko 
hello! KERN_EMERG   = <0>
hello! KERN_ALERT   = <1>
hello! KERN_CRIT    = <2>
hello! KERN_ERR     = <3>
hello! KERN_WARNING = <4>
hello! KERN_NOTICE  = <5>
hello! KERN_INFO    = <6>
[root@FriendlyARM /mnt]# rmmod print_level
goodbye! KERN_EMERG   = <0>
goodbye! KERN_ALERT   = <1>
goodbye! KERN_CRIT    = <2>
goodbye! KERN_ERR     = <3>
goodbye! KERN_WARNING = <4>
goodbye! KERN_NOTICE  = <5>
goodbye! KERN_INFO    = <6>
```

修改控制台日志级别：

```bash
[root@FriendlyARM /mnt]# cat /proc/sys/kernel/printk
7    4    1    7
[root@FriendlyARM /mnt]# echo 4 4 1 7 > /proc/sys/kernel/printk
```

修改后加载卸载模块打印的信息：

```bash
[root@FriendlyARM /mnt]# dmesg -c > 0              # 清空缓冲区 
[root@FriendlyARM /mnt]# insmod print_level.ko 
hello! KERN_EMERG   = <0>
hello! KERN_ALERT   = <1>
hello! KERN_CRIT    = <2>
hello! KERN_ERR     = <3>
[root@FriendlyARM /mnt]# rmmod print_level    
goodbye! KERN_EMERG   = <0>
goodbye! KERN_ALERT   = <1>
goodbye! KERN_CRIT    = <2>
goodbye! KERN_ERR     = <3>
```

可以看出，只打印了 0-3 级别的 log 信息，完整的 log 信息可以用 dmesg 查看。  
dmesg 命令被用于检查和控制内核的环形缓冲区。kernel 会将运行时的打印信息存储在 ring buffer 中。没有及时查看到的信息，可利用 dmesg 来查看。

```bash
[root@FriendlyARM /mnt]# dmesg 
hello! KERN_EMERG   = <0>
hello! KERN_ALERT   = <1>
hello! KERN_CRIT    = <2>
hello! KERN_ERR     = <3>
hello! KERN_WARNING = <4>
hello! KERN_NOTICE  = <5>
hello! KERN_INFO    = <6>
hello! KERN_DEBUG   = <7>
goodbye! KERN_EMERG   = <0>
goodbye! KERN_ALERT   = <1>
goodbye! KERN_CRIT    = <2>
goodbye! KERN_ERR     = <3>
goodbye! KERN_WARNING = <4>
goodbye! KERN_NOTICE  = <5>
goodbye! KERN_INFO    = <6>
goodbye! KERN_DEBUG   = <7>
```

## 二. 封装 printk 打造一个专属的打印函数

### 2.1 前言

每次调试的时候都要加行号，加函数名调试，实在是太麻烦了，为什么不自己封装一个专属的 print log 的函数呢?

```c
#define log(fmt, arg...) printk(KERN_INFO "[Paul][%s][%d] "fmt"\n", __func__, __LINE__, ##arg);
```

其中 \_\_func\_\_ \_\_LINE\_\_ 分别表示当前函数名和行号。

### 2.2 示例模块

#### 文件一：log_module.c

```c
#include <linux/init.h>
#include <linux/module.h>

/* simple log */
#define log(fmt, arg...) printk(KERN_INFO "[Paul][%s][%d] "fmt"\n", __func__, __LINE__, ##arg);
#define err(fmt, arg...) printk(KERN_ERR "[Paul][%s][%d] ERROR!!! "fmt"\n", __func__, __LINE__, ##arg);

/* detail log */
#define dbg(fmt, arg...) printk(KERN_INFO "%s:\n[Paul][%s][%d] "fmt"\n", __FILE__, __func__, __LINE__, ##arg);

static int __init mod_init(void)
{
        log("hello everyone");
        log("my name is %s", "Paul");
        log("my age is %d", 24);
        log("using hex is 0x%02x", 24);
        return 0;
}

static void __exit mod_exit(void)
{
        dbg("goodbye, everyone");
}

module_init(mod_init);
module_exit(mod_exit);

MODULE_LICENSE("GPL");
```

#### 文件二：Makefile

```bash
obj-m += log_module.o

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

加载模块，看看打印效果。

```bash
[root@FriendlyARM /mnt]# insmod log_module.ko 
[Paul][mod_init][14] hello everyone
[Paul][mod_init][15] my name is Paul
[Paul][mod_init][16] my age is 24
[Paul][mod_init][17] using hex is 0x18

[root@FriendlyARM /mnt]# rmmod log_module
/home/user/workspace/mini2440/linux_driver/02_printk/02_log/log_module.c:
[Paul][mod_exit][23] goodbye, everyone 
```



