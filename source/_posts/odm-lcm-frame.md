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

```bash
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

```c
struct app_descriptor _app_aboot __SECTION(".apps") = { .name = aboot,
        .init = aboot_init,
};
```

总结一下：
通过 APP_START 和 APP_END 两个宏创建一个 app，这种方式创建的 app 会被链接到 .apps 段。  
在 lk 启动流程中，最后会遍历 .apps 段中所有的 app，分别执行其中的 init 和 entry 函数。  
这里就清楚了, app 对应的 app_descriptor 结构必须放在 apps 段中才可以被启动。

### 1.3 lk 阶段 lcd 兼容流程

#### lcd id 寄存器介绍

先介绍下 lcd 的 id 寄存器：MIPI 组织规定各屏厂生产的 lcd ic 的 id 信息必须记录在 A1 寄存器(RDDDB: Read DDB start)中，A1 寄存器中除了第一个字节的供应商 id 是由 MIPI 组织分配的，其他字节供应商可以自定义，可以记录 ic id、ic version code 什么的。这个随便找一块 lcd 的 datasheet 看一下就明白了。
再简单介绍下 lcd 的兼容原理：原理其实很简单，其实就是代码中已经包含有多块 lcd 驱动程序，在开机过程中，会去读出实际接在手机上的 lcd 模组的 id，将读出的 id 和各个屏驱动中的 id 做对比，如果某一个 lcd 驱动的 id 和读出的 id 一样，则驱动匹配成功，就使用这个匹配上的驱动程序操作 lcd。

基于上面原理性的介绍，我们也就大概明确了稍后分析 lcd 兼容这部分代码需要理清的主线逻辑了，这里列举一下：
1. 在哪里读取手机上实际接的 lcd 的 id？
2. 在哪里遍历所有的 lcd 驱动，并和实际读取的 id 做比对？

#### 兼容框架分析

接下就是分析代码了，上一小节分析到 lk 阶段最后是启动各个 app，简单看了各个 app 的流程，最后发现 lk 的探测兼容部分是在 aboot 中实现的。

```c
// file: bootable/bootloader/lk/app/aboot/aboot.c
APP_START(aboot)
	.init = aboot_init,
APP_END

void aboot_init(const struct app_descriptor *app)
        |-- target_display_init(device.display_panel);
                |
                |-- do {
                |       |   // (1) 依次遍历代码中的兼容的 lcd 驱动做初始化操作
                |       |-- memcpy(oem.panel, (panel_name_my + panel_loop), 64);
                |       |-- gcdb_display_init(oem.panel, MDP_REV_50, (void *)MIPI_FB_ADDR);
                |       |       |-- panel_name = oem.panel;
                |       |       |-- oem_panel_select(panel_name, &panelstruct, &(panel.panel_info), &dsi_video_mode_phy_db);
                |       |       |       |
                |       |       |       |   // (2) 根据 panel_name 解析 supp_panels 数组，得到 panel 的编号
                |       |       |       |-- panel_override_id = panel_name_to_id(supp_panels, ARRAY_SIZE(supp_panels), panel_name);
                |       |       |       |-- panel_id = panel_override_id;
                |       |       |       |-- init_panel_data(panelstruct, pinfo, phy_db);
                |       |       |               |
                |       |       |               |   // (3) 根据解析出来 panel 编号，绑定对应的 lcd 驱动函数
                |       |       |               |-- switch (panel_id) {
                |       |       |               |       case ST7703_HSD_PANEL:
                |       |       |               |               ... ...
                |       |       |               |       case ST7703_BOE_PANEL:
                |       |       |               |               ... ...
                |       |       |               |       case HX83102B_HSD_PANEL:
                |       |       |               |               ... ...
                |       |       |               |               panelstruct->paneldata = &HX83102_B_720p_hsd_video_panel_data;
                |       |       |               |               pinfo->mipi.panel_compare_id_read_cmds = HX83102_B_720p_video_compare_id_page_command;
                |       |       |               |               pinfo->mipi.panel_compare_id_page_cmds = HX83102_B_720p_video_compare_id_read_command;
                |       |       |               |               pinfo->mipi.compare_id = HX83102_B_720P_VIDEO_COMPARE_ID;
                |       |       |               |               pinfo->mipi.signature = HX83102_B_720P_HSD_VIDEO_SIGNATURE;
                |       |       |               |               ... ...
                |       |       |               |               break;
                |       |       |               |-- }
                |       |       |
                |       |       |-- msm_display_init(&panel);
                |       |               |-- msm_display_config();
                |       |                       |-- mdss_dsi_config(panel);
                |       |                               |-- mdss_dsi_panel_initialize(mipi, mipi->broadcast);
                |       |                                       |-- mdss_dsi_read_panel_signature(mipi);
                |       |                                               |
                |       |                                               |   // (4) 使用绑定的 lcd 驱动函数读取手机实际接的 lcd 的 id
                |       |                                               |-- chip_id = oem_panel_compare_chip_id(mipi);
                |       |                                               |       |-- mdss_dsi_cmds_tx(mipi, mipi->panel_compare_id_page_cmds, 1, 0);
                |       |                                               |       |-- mdss_dsi_cmds_tx(mipi, mipi->panel_compare_id_read_cmds, 1, 0);
                |       |                                               |       |-- mdss_dsi_cmds_rx(mipi, &lp, 1, 1);
                |       |                                               |       |-- return (ntohl(*lp) >> 16) & 0xFF;
                |       |                                               |
                |       |                                               |   // (5) 将读取到的 id 和当前的驱动对应的 id 做比对，匹配返回 0，不匹配返回 1
                |       |                                               |-- if(chip == mipi->compare_id)
                |       |                                               |       return 0;
                |       |                                               |-- else
                |       |                                                       return 1;
                |       |
                |       |   // (1) 如果 id 匹配成功(ret = 0)或者所有屏驱动都遍历了，但没有一块屏匹配，则跳出循环，不再遍历
                |       |-- if (!ret || ret == ERR_NOT_SUPPORTED) {
		}               break;
                |
                |   // (1) 如果没有走到上面 break 的位置，则接着遍历下一个兼容的 lcd 驱动
                |-- } while (++panel_loop <= oem_panel_max_auto_detect_panels());
```

上面贴出来的框图仅仅是将 lcd 兼容函数调用关系贴出来了，一些地方描述不是很详细，下面介绍一下上图中做有标号的地方。

#### 标号1. 依次遍历代码中的兼容的 lcd 驱动做初始化操作。

```c
// file: src/bootable/bootloader/lk/target/msm8952/target_display.c

// 这里记录了目前代码中兼容的 lcd 的 panel name，看数组的第二维大小，得出每个 panel name 占用 64 字节的大小
static char panel_name_my[5][64] = {
        {"st7703_hsd_55_720p_video"},
        {"st7703_boe_55_720p_video"},
        {"hx83102b_hsd_55_720p_video"},
        {"ili9881p_panda_55_720p_video"},
        {"ili9881c_hsd_55_720p_video"},
};

void target_display_init(const char *panel_name) {
        ... ...
        do {
                target_force_cont_splash_disable(false);

                // 每个 panel name 占用 64 字节的大小，这里依次取出数组中的 panel name 
                memcpy(oem.panel, (panel_name_my + panel_loop), 64);

                // 将当前取出的 panel name 记录在 oem.panel 中，传递下去
                ret = gcdb_display_init(oem.panel, MDP_REV_50, (void *)MIPI_FB_ADDR);

                // 如果 id 匹配成功(ret = 0)或者所有屏驱动都遍历了，但没有一块屏匹配，则跳出循环，不再遍历
                if (!ret || ret == ERR_NOT_SUPPORTED) {
                        break;
                } else {
                        target_force_cont_splash_disable(true);
                        msm_display_off();
                }

        // 如果没有走到上面 break 的位置，则接着遍历下一个兼容的 lcd 驱动
        // 循环的次数为代码中兼容的屏的个数
	} while (++panel_loop <= oem_panel_max_auto_detect_panels());
	... ...
}


// file: src/bootable/bootloader/lk/target/msm8952/oem_panel.c

// 代码中兼容的屏的个数
#define DISPLAY_MAX_PANEL_DETECTION 5

uint32_t oem_panel_max_auto_detect_panels()
{
        // 原本是这段代码，需要根据 target_panel_auto_detect_enabled 函数判断当前处理器平台是否开启了兼容功能
        // return target_panel_auto_detect_enabled() ? DISPLAY_MAX_PANEL_DETECTION : 0;

        // 现在是直接修改为使能兼容状态，返回兼容的屏的个数
        return DISPLAY_MAX_PANEL_DETECTION;
}
```

#### 标号2. 根据 panel_name 解析 supp_panels 数组，得到 panel 的编号。

```c
// file: src/bootable/bootloader/lk/dev/gcdb/display/panel_display.h

struct panel_list {
        char name[MAX_PANEL_ID_LEN];
        uint32_t id;
};


// file: src/bootable/bootloader/lk/target/msm8952/oem_panel.c

enum {
        ST7703_HSD_PANEL,
        ST7703_BOE_PANEL,
        HX83102B_HSD_PANEL,
        ILI9881P_PANDA_PANEL,
        ILI9881C_HSD_PANEL,
        UNKNOWN_PANEL
};

/*
 * The list of panels that are supported on this target.
 * Any panel in this list can be selected using fastboot oem command.
 */
static struct panel_list supp_panels[] = {
        {"st7703_hsd_55_720p_video",     ST7703_HSD_PANEL},
        {"st7703_boe_55_720p_video",     ST7703_BOE_PANEL},
        {"hx83102b_hsd_55_720p_video",   HX83102B_HSD_PANEL},
        {"ili9881p_panda_55_720p_video", ILI9881P_PANDA_PANEL},
        {"ili9881c_hsd_55_720p_video",   ILI9881C_HSD_PANEL},
};


// file: src/bootable/bootloader/lk/dev/gcdb/display/panel_display.c

int32_t panel_name_to_id(struct panel_list supp_panels[], uint32_t supp_panels_size, const char *panel_name)
{
        uint32_t i;
        int32_t panel_id = ERR_NOT_FOUND;
        
        if (!panel_name) {
                dprintf(CRITICAL, "Invalid panel name\n");
                return panel_id;
        }

        // 根据 panel_name 解析 supp_panels 数组，得到 panel 的编号
        for (i = 0; i < supp_panels_size; i++) {
                if (!strncmp(panel_name, supp_panels[i].name, MAX_PANEL_ID_LEN)) {
                        panel_id = supp_panels[i].id;
                        break;
                }
        }

        return panel_id;
}
```

#### 标号3. 根据解析出来 panel 编号，绑定对应的 lcd 驱动函数

```c
// file: src/bootable/bootloader/lk/target/msm8952/oem_panel.c

// 需要包含屏供应商提供头文件
#include "include/panel_st7703_co55swr8_video.h"
#include "include/panel_st7703_boe55_video.h"
#include "include/dsi_hsd_HX83102_B_720p_video.h"
#include "include/panel_ili9881p_panda5_video.h"
#include "include/dsi_panel_ili9881c_hsd_huashi_video.h"

static int init_panel_data(struct panel_struct *panelstruct, struct msm_panel_info *pinfo, struct mdss_dsi_phy_ctrl *phy_db)
{
        int pan_type = PANEL_TYPE_DSI;
        struct oem_panel_data *oem_data = mdss_dsi_get_oem_data_ptr();

        switch (panel_id)
        {
        case ST7703_HSD_PANEL:
                ... ...
        case ST7703_BOE_PANEL:
                ... ...
        // 等号右边的变量都来自 dsi_hsd_HX83102_B_720p_video.h 头文件中
        case HX83102B_HSD_PANEL:
                panelstruct->paneldata = &HX83102_B_720p_hsd_video_panel_data;
                panelstruct->paneldata->panel_with_enable_gpio = 1;
                panelstruct->panelres = &HX83102_B_720p_hsd_video_panel_res;
                panelstruct->color = &HX83102_B_720p_hsd_video_color;
                panelstruct->videopanel = &HX83102_B_720p_hsd_video_video_panel;
                panelstruct->commandpanel = &HX83102_B_720p_hsd_video_command_panel;
                panelstruct->state = &HX83102_B_720p_hsd_video_state;
                panelstruct->laneconfig = &HX83102_B_720p_hsd_video_lane_config;

                // 读取屏 chip id 的 mipi cmd
                pinfo->mipi.panel_compare_id_read_cmds = HX83102_B_720p_video_compare_id_page_command;
                pinfo->mipi.panel_compare_id_page_cmds = HX83102_B_720p_video_compare_id_read_command;

                // 屏驱动文件中记录的屏 chip id
                pinfo->mipi.compare_id = HX83102_B_720P_VIDEO_COMPARE_ID;

                panelstruct->paneltiminginfo = &HX83102_B_720p_hsd_video_timing_info;
                panelstruct->panelresetseq = &HX83102_B_720p_hsd_video_reset_seq;
                panelstruct->backlightinfo = &HX83102_B_720p_hsd_video_backlight;
                pinfo->mipi.panel_on_cmds = HX83102_B_720p_hsd_video_on_command;
                pinfo->mipi.num_of_panel_on_cmds = HX83102_B_720P_HSD_VIDEO_ON_COMMAND;
                pinfo->mipi.panel_off_cmds = HX83102_B_720p_hsd_video_off_command;
                pinfo->mipi.num_of_panel_off_cmds = HX83102_B_720P_HSD_VIDEO_OFF_COMMAND;
                memcpy(phy_db->timing, HX83102_B_720p_hsd_video_timings, TIMING_SIZE);
                pinfo->mipi.signature = HX83102_B_720P_HSD_VIDEO_SIGNATURE;
                break;
        ... ...
        case UNKNOWN_PANEL:
        default:
                memset(panelstruct, 0, sizeof(struct panel_struct));
                memset(pinfo->mipi.panel_on_cmds, 0, sizeof(struct mipi_dsi_cmd));
                pinfo->mipi.num_of_panel_on_cmds = 0;
                memset(pinfo->mipi.panel_off_cmds, 0, sizeof(struct mipi_dsi_cmd));
                pinfo->mipi.num_of_panel_off_cmds = 0;
                memset(phy_db->timing, 0, TIMING_SIZE);
                pan_type = PANEL_TYPE_UNKNOWN;
                break;
	}

        return pan_type;
}
```

#### 标号4标号5. 探测位置

标号4和标号5的位置比较简单，不涉及其他没有贴出来的变量或函数，就不再去详细介绍。

### 1.4 lk 阶段 lcd 移植流程

经过上面兼容框架的分析，现在接着思考下在 lk 阶段 porting 一块新的 lcd 需要修改哪些地方。

#### 修改点1. src/bootable/bootloader/lk/dev/gcdb/display/include/

将屏供应商提供的屏参数头文件，如 dsi_hsd_HX83102_B_720p_video.h 文件添加到 src/bootable/bootloader/lk/dev/gcdb/display/include/ 目录下。

#### 修改点2. src/bootable/bootloader/lk/target/msm8952/oem_panel.c

```
  #include "include/panel_st7703_co55swr8_video.h"
  #include "include/panel_st7703_boe55_video.h"
+ #include "include/dsi_hsd_HX83102_B_720p_video.h"
  #include "include/panel_ili9881p_panda5_video.h"
  #include "include/dsi_panel_ili9881c_hsd_huashi_video.h"
  
  enum {
          ST7703_HSD_PANEL,
          ST7703_BOE_PANEL,
+         HX83102B_HSD_PANEL,
          ILI9881P_PANDA_PANEL,
          ILI9881C_HSD_PANEL,
          UNKNOWN_PANEL
  };
  
  static struct panel_list supp_panels[] = {
          {"st7703_hsd_55_720p_video",     ST7703_HSD_PANEL},
          {"st7703_boe_55_720p_video",     ST7703_BOE_PANEL},
+         {"hx83102b_hsd_55_720p_video",   HX83102B_HSD_PANEL},
          {"ili9881p_panda_55_720p_video", ILI9881P_PANDA_PANEL},
          {"ili9881c_hsd_55_720p_video",   ILI9881C_HSD_PANEL},
  };

switch (panel_id)
        {
+         case HX83102B_HSD_PANEL:
+                 panelstruct->paneldata = &HX83102_B_720p_hsd_video_panel_data;
+                 panelstruct->paneldata->panel_with_enable_gpio = 1;
+                 panelstruct->panelres = &HX83102_B_720p_hsd_video_panel_res;
+                 panelstruct->color = &HX83102_B_720p_hsd_video_color;
+                 panelstruct->videopanel = &HX83102_B_720p_hsd_video_video_panel;
+                 panelstruct->commandpanel = &HX83102_B_720p_hsd_video_command_panel;
+                 panelstruct->state = &HX83102_B_720p_hsd_video_state;
+                 panelstruct->laneconfig = &HX83102_B_720p_hsd_video_lane_config;
+ 
+                 // 读取屏 chip id 的 mipi cmd
+                 pinfo->mipi.panel_compare_id_read_cmds = HX83102_B_720p_video_compare_id_page_command;
+                 pinfo->mipi.panel_compare_id_page_cmds = HX83102_B_720p_video_compare_id_read_command;
+ 
+                 // 屏驱动文件中记录的屏 chip id
+                 pinfo->mipi.compare_id = HX83102_B_720P_VIDEO_COMPARE_ID;
+ 
+                 panelstruct->paneltiminginfo = &HX83102_B_720p_hsd_video_timing_info;
+                 panelstruct->panelresetseq = &HX83102_B_720p_hsd_video_reset_seq;
+                 panelstruct->backlightinfo = &HX83102_B_720p_hsd_video_backlight;
+                 pinfo->mipi.panel_on_cmds = HX83102_B_720p_hsd_video_on_command;
+                 pinfo->mipi.num_of_panel_on_cmds = HX83102_B_720P_HSD_VIDEO_ON_COMMAND;
+                 pinfo->mipi.panel_off_cmds = HX83102_B_720p_hsd_video_off_command;
+                 pinfo->mipi.num_of_panel_off_cmds = HX83102_B_720P_HSD_VIDEO_OFF_COMMAND;
+                 memcpy(phy_db->timing, HX83102_B_720p_hsd_video_timings, TIMING_SIZE);
+                 pinfo->mipi.signature = HX83102_B_720P_HSD_VIDEO_SIGNATURE;
+                 break;
```

修改点1. 

修改点1. 

修改点1. 

修改点1. 

### 1.5 lk 阶段 lcd 框架分析

对于 i2c、spi、usb 总线下的设备，写驱动程序的时候，涉及到两部分，分别是总线驱动和设备驱动。以 i2c 为例，总线驱动和设备驱动各自有对应的分工，总线驱动负责控制 i2c 控制器(片上外设)，将 i2c 控制器寄存器相关的控制逻辑封装成符合 i2c 规范的 start、stop、ask、sendbyte、readbyte 等接口函数。至于设备驱动，只需要根据设备的 datasheet 并按照总线的规范调用总线提供的接口函数，就可以实现和设备的通信。
和我们熟悉的 i2c、spi、usb 一样，mipi dsi 协议也是一样的，也分为总线驱动和设备驱动。我们分析框架，对于总线驱动部分只需要找到对应的接口函数即可，暂不去深入分析 mipi 总线驱动，重点看看点亮一块屏，设备驱动端上电时序，如下 init code，以及让 lcd 显示一帧图片需要如何操作。