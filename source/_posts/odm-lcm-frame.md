---
title: '[ODM] 高通 LCD 框架分析'
date: 2018-04-11 17:11:23
tags:
  - touch
  - driver
  - kernel
---

## 一. lk 阶段 lcd 框架

### 1.1 lk 阶段启动流程概览

lk 阶段框架不妨从 lk 阶段的 c 代码开始分析，第一个执行的 c 语言函数是 main.c 文件中的 kmain 函数。

```
// file: src/bootable/bootloader/lk/kernel/main.c
void kmain(void)
        thread_t *thr;
        // get us into some sort of thread context
        thread_init_early();
        // early arch stuff
        arch_early_init();
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
        thr = thread_create("bootstrap2", &bootstrap2, NULL, DEFAULT_PRIORITY, 
        		DEFAULT_STACK_SIZE);
        if (!thr)
        {
                panic("failed to create thread bootstrap2\n");
        }
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

```
// file: src/bootable/bootloader/lk/kernel/main.c
void kmain(void)
        thr = thread_create("bootstrap2", &bootstrap2, NULL, 16, 8192);
                static int bootstrap2(void *arg)
                        apps_init();
```

### 1.2 lk 阶段 app 启动流程

看看 apps_init() 都干了些什么？

```
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

从上面的代码可以看出，每一个 app 都使用 app_descriptor 描述，都具有一个 init 函数，一个 entry 函数和一个 flags 标识。
在 apps_init() 中会去遍历一个 app 的列表，分别调用他们的 init 函数，对 app 进行初始化。初始化结束后，还会根据 flags 标识位决定是否调用 start_app(app) 函数创建一个线程运行 app 的 entry 函数。
  
那么问题来了，这里的 app 的列表是怎么来的？
上面 app 是从 \__apps_start 开始取出 app 元素，一直取到 \__apps_end 才结束，那么 \__apps_start 和 \__apps_end 在哪里定义？
代码中居然找不到，总有定义的地方吧，grep 搜一下，原来在链接脚本中。

```
[wangbing@ubuntu: lk]$ grep -rsn "__apps_start"
arch/arm/trustzone-system-onesegment.ld:51:             __apps_start = .;
arch/arm/system-twosegment.ld:47:                       __apps_start = .;
arch/arm/trustzone-test-system-onesegment.ld:52:        __apps_start = .;
arch/arm/system-onesegment.ld:47:                       __apps_start = .;
```

打开链接脚本看看，可以发现四个脚本描述 \__apps_start 和 \__apps_end 都一样，都是表示 .apps 端的起始结束地址。

```
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

```
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

在 app.h 有两个创建 app 的宏，分别是 APP_START 和 APP_END。搜索一下，看看别人是怎么用他们创建 app 的。

```
---- APP_START Matches (7 in 7 files) ----
aboot.c (bootloader\lk\app\aboot) line 5400 : APP_START(aboot)
app.h (bootloader\lk\include) line 45 : #define APP_START(appname) struct app_descriptor _app_##appname __SECTION(".apps") = { .name = #appname,
clock_tests.c (bootloader\lk\app\clocktests) line 119 : APP_START(clocktests)
pci_tests.c (bootloader\lk\app\pcitests) line 247 : APP_START(pcitests)
shell.c (bootloader\lk\app\shell) line 37 : APP_START(shell)
string_tests.c (bootloader\lk\app\stringtests) line 283 : APP_START(stringtests)
tests.c (bootloader\lk\app\tests) line 42 : APP_START(tests)
```

就以第一个 aboot.c 为例吧，看看是怎么创建的。

```
// file: bootable/bootloader/lk/app/aboot/aboot.c
APP_START(aboot)
	.init = aboot_init,
APP_END
```

将宏定义展开，没错，就是这里指定该代码存放在 .apps 段。

```
struct app_descriptor _app_aboot __SECTION(".apps") = { .name = aboot,
        .init = aboot_init,
};
```

总结一下：
通过 APP_START 和 APP_END 两个宏创建一个 app，这种方式创建的 app 会被链接到 .apps 段。  
在 lk 启动流程中，最后会遍历 .apps 段中所有的 app，分别执行其中的 init 和 entry 函数。  
这里就清楚了, app 对应的 app_descriptor 结构必须放在 apps 段中才可以被启动。
