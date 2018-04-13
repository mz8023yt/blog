---
title: '[ODM] 高通 LCD 框架分析'
date: 2018-04-11 17:11:23
tags:
  - lcd
  - qcom
categories: ODM
---

## 一. lk 阶段 lcd 框架

### 1.1 lk 阶段启动流程概览

lk 阶段框架不妨从 lk 阶段的 c 代码开始分析，第一个执行的 c 语言函数是 main.c 文件中的 kmain 函数。

```c
// file: src/bootable/bootloader/lk/kernel/main.c
void kmain(void)
        // get us into some sort of thread context
        thread_init_early();

        // early arch stuff
        arch_early_init()；

        // do any super early platform initialization
        platform_early_init();
                board_init();
                platform_clock_init();
                qgic_init();
                qtimer_init();
                scm_init();

        // do any super early target initialization
        target_early_init();
        bs_set_timestamp(BS_BL_START);

        // deal with any static constructors
        call_constructors();

        // bring up the kernel heap
        heap_init();
        __stack_chk_guard_setup();

        // initialize the threading system
        thread_init();

        // initialize the dpc system
        dpc_init();

        // initialize kernel timers
        timer_init();

        // create a thread to complete system initialization
        thr = thread_create("bootstrap2", &bootstrap2, NULL, DEFAULT_PRIORITY, DEFAULT_STACK_SIZE);
        thread_resume(thr);

        // enable interrupts
        exit_critical_section();

        // become the idle thread
        thread_become_idle();

static int bootstrap2(void *arg)
        arch_init();
        platform_init();
        target_init();
                target_keystatus();
                target_sdc_init();
                vib_timed_turn_on(VIBRATE_TIME);
        apps_init();
```

简化一些与屏启动不相关的流程后的得到的函数调用关系为：

```c
// file: src/bootable/bootloader/lk/kernel/main.c
void kmain(void)
        |-- thread_create("bootstrap2", &bootstrap2, NULL, 16, 8192);
        <=> static int bootstrap2(void *arg)
                |-- apps_init();
```

这里可以看出，lk 在做完平台端必要的初始化后，会去启动所谓的 app，那么 app 是什么东西？如何定义？又在哪里被执行？将在下一小节中介绍。

### 1.2 lk 阶段 app 启动流程

先看看 apps_init() 都干了些什么？

```c
// file: src/bootable/bootloader/lk/app/app.c
#include <debug.h>
#include <app.h>
#include <kernel/thread.h>

extern const struct app_descriptor __apps_start;
extern const struct app_descriptor __apps_end;

static void start_app(const struct app_descriptor *app);

/* one time setup */
void apps_init(void)
{
	const struct app_descriptor *app;

	/* call all the init routines */
	for (app = &__apps_start; app != &__apps_end; app++) {
		if (app->init)
			app->init(app);
	}

	/* start any that want to start on boot */
	for (app = &__apps_start; app != &__apps_end; app++) {
		if (app->entry && (app->flags & APP_FLAG_DONT_START_ON_BOOT) == 0) {
			start_app(app);
		}
	}
}

static int app_thread_entry(void *arg)
{
	const struct app_descriptor *app = (const struct app_descriptor *)arg;

	app->entry(app, NULL);

	return 0;
}

static void start_app(const struct app_descriptor *app)
{
	thread_t *thr;
	printf("starting app %s\n", app->name);

	thr = thread_create(app->name, &app_thread_entry, (void *)app, DEFAULT_PRIORITY, DEFAULT_STACK_SIZE);
	if(!thr)
	{
		return;
	}
	thread_resume(thr);
}
```

从上面的代码可以看出，每一个 app 都使用 app_descriptor 描述，并具有一个 init 函数，一个 entry 函数和一个 flags 标识。
在 apps_init() 中会去遍历一个 app 的列表，分别调用他们的 init 函数，对 app 进行初始化。初始化结束后，还会根据 flags 标识位决定是否调用 start_app(app) 函数创建一个线程运行 app 的 entry 函数。
  
那么问题来了，这里的 app 的列表是怎么来的？
上面 app 是从 \__apps_start 开始取出 app 元素，一直取到 \__apps_end 才结束，那么 \__apps_start 和 \__apps_end 在哪里定义？
代码中居然找不到，总有定义的地方吧，grep 搜一下，原来在链接脚本中。

```bash
[wangbing@ubuntu: lk]$ grep -rsn "__apps_start"
arch/arm/trustzone-system-onesegment.ld:51:      __apps_start = .;
arch/arm/system-twosegment.ld:47:                __apps_start = .;
arch/arm/trustzone-test-system-onesegment.ld:52: __apps_start = .;
arch/arm/system-onesegment.ld:47:                __apps_start = .;
```

打开链接脚本看看，可以发现四个脚本描述 \__apps_start 和 \__apps_end 都一样，都是表示 .apps 端的起始结束地址。

```bash
.rodata : { 
	*(.rodata .rodata.* .gnu.linkonce.r.*) 
	. = ALIGN(4);
	__commands_start = .;
	KEEP (*(.commands))
	__commands_end = .;
	. = ALIGN(4);
	__apps_start = .;               // 起始地址
	KEEP (*(.apps)) 
	__apps_end = .;                 // 结束地址
	. = ALIGN(4); 
	__rodata_end = . ;
}
```

看到这里疑问就解除了，app 列表是在从 .apps 段中取出来的。但是新的问题又来了，lk 驱动中是怎么将 app 对应的 app_descriptor 结构链接到 .apps 段中呢？
那就好好看看 app.c 和 app.h 文件吧，其中 app.c 上面已经贴出来了，下面仅贴出 app.h 文件。

```c
// file: src/bootable/bootloader/lk/app/app.h
#ifndef __APP_H
#define __APP_H

/* app support api */
void apps_init(void); /* one time setup */

/* app entry point */
struct app_descriptor;
typedef void (*app_init)(const struct app_descriptor *);
typedef void (*app_entry)(const struct app_descriptor *, void *args);

/* app startup flags */
#define APP_FLAG_DONT_START_ON_BOOT 0x1

/* each app needs to define one of these to define its startup conditions */
struct app_descriptor {
	const char *name;
	app_init  init;
	app_entry entry;
	unsigned int flags;
};

#define APP_START(appname) struct app_descriptor _app_##appname __SECTION(".apps") = { .name = #appname,
#define APP_END };

#endif
```

在 app.h 的最后几行提供了两个用于创建 app 的宏，分别是 APP_START 和 APP_END。搜索一下，看看别人是怎么用他们创建 app 的。

```
---- APP_START Matches (7 in 7 files) ----
aboot.c (bootloader\lk\app\aboot) line 5400 :             APP_START(aboot)
clock_tests.c (bootloader\lk\app\clocktests) line 119 :   APP_START(clocktests)
pci_tests.c (bootloader\lk\app\pcitests) line 247 :       APP_START(pcitests)
shell.c (bootloader\lk\app\shell) line 37 :               APP_START(shell)
string_tests.c (bootloader\lk\app\stringtests) line 283 : APP_START(stringtests)
tests.c (bootloader\lk\app\tests) line 42 :               APP_START(tests)
```

就以第一个 aboot.c 为例吧，看看它是怎么创建的。

```c
// file: bootable/bootloader/lk/app/aboot/aboot.c
APP_START(aboot)
	.init = aboot_init,
APP_END
```

将宏定义展开，没错，就是这里指定该结构存放在 .apps 段。

```
struct app_descriptor _app_aboot __SECTION(".apps") = { .name = aboot,
        .init = aboot_init,
};
```

总结一下：
通过 APP_START 和 APP_END 两个宏创建一个 app，这种方式创建的 app 会被链接到 .apps 段。  
在 lk 启动流程中，最后会遍历 .apps 段中所有的 app，分别执行其中的 init 和 entry 函数。  
这里就清楚了, app 对应的 app_descriptor 结构必须放在 apps 段中才可以被启动。

### 1.3 lk 阶段 lcd 兼容流程

先介绍下 lcd 的 id 寄存器：MIPI 组织规定各屏厂生产的 lcd ic 的 id 信息必须记录在 A1 寄存器(RDDDB: Read DDB start)中，A1 寄存器中除了第一个字节的供应商 id 是由 MIPI 组织分配的，其他字节供应商可以自定义，可以记录 ic id、ic version code 什么的。这个随便找一块 lcd 的 datasheet 看一下就明白了。
再简单介绍下 lcd 的兼容原理：原理其实很简单，其实就是代码中已经包含有多块 lcd 驱动程序，在开机过程中，会去读出实际接在手机上的 lcd 模组的 id，将读出的 id 和各个屏驱动中的 id 做对比，如果某一个 lcd 驱动的 id 和读出的 id 一样，则驱动匹配成功，就使用这个匹配上的驱动程序操作 lcd。

基于上面原理性的介绍，我们也就大概明确了稍后分析 lcd 兼容这部分代码需要理清的主线逻辑了，这里列举一下：
1. 在哪里读取手机上实际接的 lcd 的 id？
2. 在哪里遍历所有的 lcd 驱动，并和实际读取的 id 做比对？

接下就分析代码中，lk 阶段最后是启动各个 app，简单看了各个 app 的流程，最后发现 lk 的探测兼容部分是在 aboot 中实现的。

```bash
// file: bootable/bootloader/lk/app/aboot/aboot.c
APP_START(aboot)
	.init = aboot_init,
APP_END

void aboot_init(const struct app_descriptor *app)
        |-- target_display_init(device.display_panel);
                |
                |   >>> 1 <<<
                |-- do {
                |       |-- memcpy(oem.panel, (panel_name_my + panel_loop), 64);
                |       |-- gcdb_display_init(oem.panel, MDP_REV_50, (void *)MIPI_FB_ADDR);
                |               |-- oem_panel_select(panel_name, &panelstruct, &(panel.panel_info), &dsi_video_mode_phy_db);
                |               |       |
                |               |       |   >>> 2 <<<
                |               |       |-- panel_override_id = panel_name_to_id(supp_panels, ARRAY_SIZE(supp_panels), panel_name);
                |               |       |-- panel_id = panel_override_id;
                |               |       |-- init_panel_data(panelstruct, pinfo, phy_db);
                |               |               |
                |               |               |   >>> 3 <<<
                |               |               |-- switch (panel_id)
                |               |
                |               |-- msm_display_init(&panel);
                |                       |-- msm_display_config();
                |                               |-- mdss_dsi_config(panel);
                |                                       |
                |                                       |   >>> 4 <<<
                |                                       |-- mdss_dsi_panel_initialize(mipi, mipi->broadcast);
                |
                |-- } while (++panel_loop <= oem_panel_max_auto_detect_panels());

```

### 1.4 lk 阶段 lcd 移植流程

在分析 lk 阶段的 lcd 框架之前，不妨先看看 lk 阶段 porting lcd 需要修改哪些地方。

### 1.5 lk 阶段 lcd 框架分析

对于 i2c、spi、usb 总线下的设备，写驱动程序的时候，涉及到两部分，分别是总线驱动和设备驱动。以 i2c 为例，总线驱动和设备驱动各自有对应的分工，总线驱动负责控制 i2c 控制器(片上外设)，将 i2c 控制器寄存器相关的控制逻辑封装成符合 i2c 规范的 start、stop、ask、sendbyte、readbyte 等接口函数。至于设备驱动，只需要根据设备的 datasheet 并按照总线的规范调用总线提供的接口函数，就可以实现和设备的通信。
和我们熟悉的 i2c、spi、usb 一样，mipi dsi 协议也是一样的，也分为总线驱动和设备驱动。我们分析框架，对于总线驱动部分只需要找到对应的接口函数即可，暂不去深入分析 mipi 总线驱动，重点看看点亮一块屏，设备驱动端上电时序，如下 init code，以及让 lcd 显示一帧图片需要如何操作。