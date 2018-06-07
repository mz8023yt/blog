---
title: '[ODM] 高通 LCD 框架分析'
date: 2018-04-11 17:11:23
tags:
  - lcd
  - qcom
categories: ODM
---

## 一. LCD 兼容

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
// file: src/bootable/bootloader/lk/app/aboot/aboot.c

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

### 1.3 lk 阶段 lcd 兼容原理

#### 兼容原理概述

先介绍下 lcd 的 id 寄存器：MIPI 组织规定各屏厂生产的 lcd ic 的 id 信息必须记录在 A1 寄存器(RDDDB: Read DDB start)中，A1 寄存器中除了第一个字节的供应商 id 是由 MIPI 组织分配的，其他字节供应商可以自定义，可以记录 ic id、ic version code 什么的。这个随便找一块 lcd 的 datasheet 看一下就明白了。

再简单介绍下 lcd 的兼容原理：原理其实很简单，其实就是代码中已经包含有多块 lcd 驱动程序，在开机过程中，会去读出实际接在手机上的 lcd 模组的 id，将读出的 id 和各个屏驱动中的 id 做对比，如果某一个 lcd 驱动的 id 和读出的 id 一样，则驱动匹配成功，就使用这个匹配上的驱动程序操作 lcd。

基于上面原理性的介绍，我们也就大概明确了稍后分析 lcd 兼容这部分代码需要理清的主线逻辑了，这里列举一下：
1. 在哪里读取手机上实际接的 lcd 的 id？
2. 在哪里遍历所有的 lcd 驱动，并和实际读取的 id 做比对？

#### 兼容框架分析

接下就是分析代码了，上一小节分析到 lk 阶段最后是启动各个 app，简单看了各个 app 的流程，最后发现 lk 的探测兼容部分是在 aboot 中实现的。

```c
// file: src/bootable/bootloader/lk/app/aboot/aboot.c

APP_START(aboot)
	.init = aboot_init,
APP_END

void aboot_init(const struct app_descriptor *app)
    |- target_display_init(device.display_panel);
        |
        |- do {
        |   |  // (1) 依次遍历代码中的兼容的 lcd 驱动做初始化操作
        |   |- memcpy(oem.panel, (panel_name_my + panel_loop), 64);
        |   |- gcdb_display_init(oem.panel, MDP_REV_50, (void *)MIPI_FB_ADDR);
        |   |   |- panel_name = oem.panel;
        |   |   |- oem_panel_select(panel_name, &panelstruct, &(panel.panel_info), &dsi_video_mode_phy_db);
        |   |   |   |
        |   |   |   |  // (2) 根据 panel_name 解析 supp_panels 数组，得到 panel 的编号
        |   |   |   |- panel_override_id = panel_name_to_id(supp_panels, ARRAY_SIZE(supp_panels), panel_name);
        |   |   |   |- panel_id = panel_override_id;
        |   |   |   |- init_panel_data(panelstruct, pinfo, phy_db);
        |   |   |       |
        |   |   |       |  // (3) 根据解析出来 panel 编号，绑定对应的 lcd 驱动函数
        |   |   |       |- switch (panel_id) {
        |   |   |       |   case ST7703_HSD_PANEL:
        |   |   |       |       ... ...
        |   |   |       |   case ST7703_BOE_PANEL:
        |   |   |       |       ... ...
        |   |   |       |   case HX83102B_HSD_PANEL:
        |   |   |       |       ... ...
        |   |   |       |       pinfo->mipi.panel_compare_id_read_cmds = HX83102_compare_id_page_command;
        |   |   |       |       pinfo->mipi.panel_compare_id_page_cmds = HX83102_compare_id_read_command;
        |   |   |       |       pinfo->mipi.compare_id = HX83102_COMPARE_ID;
        |   |   |       |       pinfo->mipi.signature = HX83102_SIGNATURE;
        |   |   |       |       ... ...
        |   |   |       |       break;
        |   |   |       |- }
        |   |   |
        |   |   |- msm_display_init(&panel);
        |   |           |- msm_display_config();
        |   |               |- mdss_dsi_config(panel);
        |   |                   |- mdss_dsi_panel_initialize(mipi, mipi->broadcast);
        |   |                       |- mdss_dsi_read_panel_signature(mipi);
        |   |                           |
        |   |                           |  // (4) 使用绑定的 lcd 驱动函数读取手机实际接的 lcd 的 id
        |   |                           |- chip_id = oem_panel_compare_chip_id(mipi);
        |   |                           |       |- mdss_dsi_cmds_tx(mipi, mipi->panel_compare_id_page_cmds, 1, 0);
        |   |                           |       |- mdss_dsi_cmds_tx(mipi, mipi->panel_compare_id_read_cmds, 1, 0);
        |   |                           |       |- mdss_dsi_cmds_rx(mipi, &lp, 1, 1);
        |   |                           |       |- return (ntohl(*lp) >> 16) & 0xFF;
        |   |                           |
        |   |                           |  // (5) 将读取到的 id 和当前的驱动对应的 id 做比对，匹配返回 0，不匹配返回 1
        |   |                           |- if(chip_id == mipi->compare_id)
        |   |                           |   return 1;
        |   |                           |- else
        |   |                               return 0;
        |   |
        |   |  // (1) 如果 id 匹配成功(ret = 0)或者所有屏驱动都遍历了，但没有一块屏匹配，则跳出循环，不再遍历
        |   |- if (!ret || ret == ERR_NOT_SUPPORTED)
        |       break;
        |
        |  // (1) 如果没有走到上面 break 的位置，则接着遍历下一个兼容的 lcd 驱动
        |- } while (++panel_loop <= oem_panel_max_auto_detect_panels());
```

上面贴出来的框图仅仅是将 lcd 兼容函数调用关系贴出来了，一些地方描述不是很详细，下面介绍一下上图中做有标号的地方。

#### 标号1. 依次遍历代码中的兼容的 lcd 驱动做初始化操作

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

#### 标号2. 根据 panel_name 解析 supp_panels 数组，得到 panel 的编号

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
#include "include/panel_hx83102b_hsd_video.h"
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
        // 等号右边的变量都来自 panel_hx83102b_hsd_video.h 头文件中
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

### 1.4 kernel 阶段 lcd 兼容原理

#### 兼容原理概述

kernel 阶段的兼容相比 lk 阶段要简单的多，由于在 lk 阶段已经做过了遍历所有屏驱动探测 lcd 的动作，再加上 bootloader 可以通过 cmdline 传参机制传递信息到 kernel 中。这样的话，完全可以通过 cmdline 将 lk 阶段的探测到的屏名称直接传递到 kernel，kernel 拿到传递过来的 panel name 再去加载对应的屏驱动。

#### 兼容框架分析

lk 阶段将屏信息写入 cmdline。

```c
// file: src/bootable/bootloader/lk/app/aboot/aboot.c

char display_panel_buf[MAX_PANEL_BUF_SIZE];

void aboot_init(const struct app_descriptor *app)
        |-- memset(display_panel_buf, '\0', MAX_PANEL_BUF_SIZE);
        |-- boot_linux_from_mmc();
                |-- boot_linux(hdr->kernel_addr, hdr->tags_addr, hdr->cmdline, board_machtype(), hdr->ramdisk_addr, hdr->ramdisk_size);
                        |-- cmdline = hdr->cmdline;
                        |-- final_cmdline = update_cmdline(cmdline);
                        |       |-- target_display_panel_node(display_panel_buf, MAX_PANEL_BUF_SIZE);
                        |       |-- pbuf = display_panel_buf
                        |       |       |-- gcdb_display_cmdline_arg(pbuf, buf_size);
                        |       |               |
                        |       |               |   // 将屏供应商提供的头文件中的 panel config 保存到 pbuf 中
                        |       |               |-- panel_node = panelstruct.paneldata->panel_node_id;
                        |       |               |-- strlcpy(pbuf, panel_node, buf_size);
                        |       |
                        |       |   // 设置拷贝的目的地址为 cmdline_final(cmdline)
                        |       |-- dst = cmdline_final;
                        |       |
                        |       |   // 设置拷贝的源地址为 display_panel_buf(panel config)
                        |       |-- src = display_panel_buf;
                        |       |-- if (have_cmdline)
                        |       |        --dst;
                        |       |
                        |       |   // 将 panel config 追加到 cmdline
                        |       |-- while ((*dst++ = *src++));
                        |       |--return cmdline_final;
                        |
                        |-- update_device_tree((void *)tags, (const char *)final_cmdline, ramdisk, ramdisk_size);
                                |-- fdt_appendprop_string(fdt, offset, (const char*)"bootargs", (const void*)cmdline);
                                ... ... 待补充
```

kernel 阶段解析 cmdline 获取到 lk 传递过来的屏信息。

```c
// file: src/kernel/msm-3.18/drivers/video/msm/mdss/mdss_dsi.c

module_init(mdss_dsi_ctrl_driver_init);

static int __init mdss_dsi_ctrl_driver_init(void)
        mdss_dsi_ctrl_register_driver();
                platform_driver_register(&mdss_dsi_ctrl_driver);

static struct platform_driver mdss_dsi_ctrl_driver = {
        .probe = mdss_dsi_ctrl_probe,
        .remove = mdss_dsi_ctrl_remove,
        .shutdown = NULL,
        .driver = {
                .name = "mdss_dsi_ctrl",
                .of_match_table = mdss_dsi_ctrl_dt_match,
        },
};

// 匹配的 dts 节点在这里指定 "qcom,mdss-dsi-ctrl"
static const struct of_device_id mdss_dsi_ctrl_dt_match[] = {
        {.compatible = "qcom,mdss-dsi-ctrl"},
        {}
};

static int mdss_dsi_ctrl_probe(struct platform_device *pdev)
        |
        |-- struct device_node *dsi_pan_node = mdss_dsi_config_panel(pdev, index);
                |
                |   // 取出私有数据
                |-- struct mdss_dsi_ctrl_pdata *ctrl_pdata = platform_get_drvdata(pdev);
                |-- char panel_cfg[MDSS_MAX_PANEL_LEN];
                |
                |   // 通过私有数据中的函数接口获取到屏的配置信息
                |-- mdss_dsi_get_panel_cfg(panel_cfg, ctrl_pdata);
                |       |-- ctrl = ctrl_pdata;
                |       |
                |       |   // 这里通过 panel_intf_type 函数获取到屏的配置信息
                |       |   // 这里获取到的字符串是 0:qcom,dsi_co55swr8_st7703_720p_video:1:none:cfg:single_dsi
                |       |-- pan_cfg = ctrl->mdss_util->panel_intf_type(MDSS_PANEL_INTF_DSI);
                |       |-- strlcpy(panel_cfg, pan_cfg->arg_cfg, sizeof(pan_cfg->arg_cfg));
                |
                |   // 通过 panel_cfg 找到对应屏的 dts 节点, panel_cfg 字符串就是从 cmd line 中解析出来的
                |-- struct device_node *dsi_pan_node = mdss_dsi_find_panel_of_node(pdev, panel_cfg);
                |
                |   // 拿到了对应的 dts 节点就可以去初始化 panel 了
                |-- mdss_dsi_panel_init(dsi_pan_node, ctrl_pdata, ndx);
                |       |
                |       |   // 将 panel nane 取出，写到 device info 中
                |       |-- panel_name = of_get_property(node, "qcom,mdss-dsi-panel-name", NULL);
                |       |-- sprintf(wind_device_info.lcm_module_info.ic_name, "%s", panel_name);
                |       |
                |       |   // 绑定上电和背光控制的操作函数，后面有章节专门介绍
                |       |-- ctrl_pdata->on = mdss_dsi_panel_on;
                |       |-- ctrl_pdata->off = mdss_dsi_panel_off;
                |       |-- ctrl_pdata->panel_data.set_backlight = mdss_dsi_panel_bl_ctrl;
                |
                |-- return dsi_pan_node;
```

lk 阶段在将 panel_config 写进 cmdline，kernel 中解析出出 cmdline 中 panel_config 的 panel_node_id，找到 panel_node_id 同名的 dts 节点。最后将找到的 dts 节点中的 "qcom,mdss-dsi-panel-name" 属性值写进 device info。
总结一下：这样的话就要求 panel_config->panel_node_id 要和屏 dts 节点名要相同。

## 二. LCD 移植

### 2.1 lk 阶段 lcd 移植流程

经过上面兼容框架的分析，现在接着思考下在 lk 阶段 porting 一块新的 lcd 需要修改哪些地方。

#### 修改点1. 屏参数头文件

将屏供应商提供的屏参数头文件，如 panel_hx83102b_hsd_video.h 文件添加到 src/bootable/bootloader/lk/dev/gcdb/display/include/ 目录下。
这个屏参数头文件中，包含有屏的启动参数，屏的配置信息等，在 init_panel_data() 函数中会将使用到这些信息。

#### 修改点2. target_display.c

这里需要在 panel_name_my 中追加新添加的 panel name，这样在遍历 lcd 驱动的时候才会取出新添加的屏驱动做探测的动作。

```c
diff --git a/bootable/bootloader/lk/target/msm8952/target_display.c b/bootable/bootloader/lk/target/msm8952/target_display.c
index a4548d7..16bd60b 100644
--- a/bootable/bootloader/lk/target/msm8952/target_display.c
+++ b/bootable/bootloader/lk/target/msm8952/target_display.c
@@ -59,6 +59,7 @@
 #define TRULY_CMD_PANEL_STRING "1:dsi:0:qcom,mdss_dsi_truly_720p_cmd:1:none:cfg:single_dsi"

 static char panel_name_my[5][64] = {
+       {"hx83102b_hsd_55_720p_video"},
        {"st7703_hsd_55_720p_video"},
        {"st7703_boe_55_720p_video"},
        {"ili9881p_panda_55_720p_video"},
        {"ili9881c_hsd_55_720p_video"},
 };
```

#### 修改点3. oem_panel.c

这里文件修改点比较多，有 5 修改点，分别是：
1. 包含屏参数头文件。
2. 在枚举中追加新兼容的屏的 panel id。
3. 将新兼容屏的 panel name 和 panel id 匹配上，并追加到 supp_panels 数组中。
4. 在 switch (panel_id) 位置绑定屏相关参数。
5. 修改目前兼容的 lcd 的数量。

```c
diff --git a/bootable/bootloader/lk/target/msm8952/oem_panel.c b/bootable/bootloader/lk/target/msm8952/oem_panel.c
index d838a8b..742053b 100644
--- a/bootable/bootloader/lk/target/msm8952/oem_panel.c
+++ b/bootable/bootloader/lk/target/msm8952/oem_panel.c
@@ -66,6 +66,7 @@
 #include "include/panel_st7703_co55swr8_video.h"
 #include "include/panel_st7703_boe55_video.h"
 #include "include/panel_ili9881p_panda5_video.h"
+#include "include/panel_hx83102b_hsd_video.h"

 /*---------------------------------------------------------------------------*/
 /* static panel selection variable                                           */
@@ -74,6 +75,7 @@
 #define ID1_PIN 61

 enum {
+       HX83102B_HSD_PANEL,
        ST7703_HSD_PANEL,
        ST7703_BOE_PANEL,
        ILI9881P_PANDA_PANEL,
@@ -108,6 +110,7 @@ uint32_t panel_regulator_settings[] = {
  * Any panel in this list can be selected using fastboot oem command.
  */
 static struct panel_list supp_panels[] = {
+       {"hx83102b_hsd_55_720p_video",   HX83102B_HSD_PANEL},
        {"st7703_hsd_55_720p_video",     ST7703_HSD_PANEL},
        {"st7703_boe_55_720p_video",     ST7703_BOE_PANEL},
        {"ili9881p_panda_55_720p_video", ILI9881P_PANDA_PANEL},

@@ -191,6 +194,28 @@ static int init_panel_data(struct panel_struct *panelstruct,

        switch (panel_id)
        {
+       case HX83102B_HSD_PANEL:
+               panelstruct->paneldata = &HX83102_B_720p_hsd_video_panel_data;
+               panelstruct->paneldata->panel_with_enable_gpio = 1;
+               panelstruct->panelres = &HX83102_B_720p_hsd_video_panel_res;
+               panelstruct->color = &HX83102_B_720p_hsd_video_color;
+               panelstruct->videopanel = &HX83102_B_720p_hsd_video_video_panel;
+               panelstruct->commandpanel = &HX83102_B_720p_hsd_video_command_panel;
+               panelstruct->state = &HX83102_B_720p_hsd_video_state;
+               panelstruct->laneconfig = &HX83102_B_720p_hsd_video_lane_config;
+               pinfo->mipi.panel_compare_id_read_cmds = HX83102_B_720p_video_compare_id_page_command;
+               pinfo->mipi.panel_compare_id_page_cmds = HX83102_B_720p_video_compare_id_read_command;
+               pinfo->mipi.compare_id = HX83102_B_720P_VIDEO_COMPARE_ID;
+               panelstruct->paneltiminginfo = &HX83102_B_720p_hsd_video_timing_info;
+               panelstruct->panelresetseq = &HX83102_B_720p_hsd_video_reset_seq;
+               panelstruct->backlightinfo = &HX83102_B_720p_hsd_video_backlight;
+               pinfo->mipi.panel_on_cmds = HX83102_B_720p_hsd_video_on_command;
+               pinfo->mipi.num_of_panel_on_cmds = HX83102_B_720P_HSD_VIDEO_ON_COMMAND;
+               pinfo->mipi.panel_off_cmds = HX83102_B_720p_hsd_video_off_command;
+               pinfo->mipi.num_of_panel_off_cmds = HX83102_B_720P_HSD_VIDEO_OFF_COMMAND;
+               memcpy(phy_db->timing, HX83102_B_720p_hsd_video_timings, TIMING_SIZE);
+               pinfo->mipi.signature = HX83102_B_720P_HSD_VIDEO_SIGNATURE;
+               break;
        case ST7703_BOE_PANEL:
                panelstruct->paneldata = &st7703_boe55_720p_video_panel_data;
                panelstruct->paneldata->panel_with_enable_gpio = 1;
@@ -895,7 +920,7 @@ uint32_t oem_panel_compare_chip_id(struct mipi_panel_info *mipi)
        return data;
 }

-#define DISPLAY_MAX_PANEL_DETECTION 3
+#define DISPLAY_MAX_PANEL_DETECTION 4
 static uint32_t auto_pan_loop = 0;

 uint32_t oem_panel_max_auto_detect_panels()
```

### 2.2 kernel 阶段 lcd 移植流程

#### 修改点1. 添加屏相关的 dtsi 文件

咨询 FAE 要到 LCM 相关的 panel dtsi 文件. 添加到 src/kernel/msm-3.18/arch/arm/boot/dts/qcom/ 目录下:
eg: src/kernel/msm-3.18/arch/arm/boot/dts/qcom/dsi-panel-st7703_boe55-video.dtsi
此 dtsi 仅仅是在 mdss_mdp 节点下追加了一个子节点。

```c
&mdss_mdp {
        dsi_boe55_st7703_720p_video: qcom,dsi_boe55_st7703_720p_video {
                qcom,mdss-dsi-panel-name = "dsi_boe55_st7703_720p_video";
                ... ...
        };
};
```

问1: msm8937 是 64 位的处理器, 为什么是在 arch/arm/ 下添加文件, 而不是在 arch/arm64 目录下添加?  
答1: 其实 arch/arm64 是通过软链接方式链接到 arch/arm 目录下，因此不管在哪个目录修改都是可以生效的。

```bash
[wangbing@ubuntu: ~/src/kernel/msm-3.18/arch/arm64/boot/dts]$ ls -l
lrwxrwxrwx 1 wangbing wangbing   26 Apr  3 23:07 qcom -> ../../../arm/boot/dts/qcom
```

#### 修改点2. 包含屏参数 dtsi 文件到 dts 中

在 src/kernel/msm-3.18/arch/arm/boot/dts/qcom/msm8937-mdss-panels.dtsi 包含上一个修改点中添加的 dtsi 文件。目的是为了让追加的 panel 的 dtsi 在编译 dtb 的时候能够被编译到。

```c
#include "dsi-panel-st7703_boe55-video.dtsi"
```

问1: dtb 编译的时候是如何确定哪些文件(哪些 dts 和 dtsi 文件)会被编译呢?  
答1: 编译 dtb 文件也是根据 Makefile 规则编译的，在 src/kernel/msm-3.18/arch/arm/boot/dts/qcom/ 目录的 Makefile 决定哪些 dts 文件会编译成 dtb。编译的时候会编译 Makefile 中指定的 dtb 文件, 而这些 dtb 文件对应的源文件(dts文件)会依赖于相关的 dtsi 文件。编译的时候就会将 dts 以及依赖的 dtsi 一起编译(类似于c文件依赖h文件一样)。

问2: dsi-panel-st7703_boe55-video.dtsi 文件的依赖关系是?  
答2: 一直根据 grep 搜索对应关键字, 就可以找到依赖关系:

```bash
msm8937-mdss-panels.dtsi:34:    #include "dsi-panel-st7703_boe55-video.dtsi"
msm8937-mtp.dtsi:320:           #include "msm8937-mdss-panels.dtsi"
msm8937-pmi8937-mtp.dtsi:15:    #include "msm8937-mtp.dtsi"
msm8937-pmi8937-mtp.dts:17:     #include "msm8937-pmi8937-mtp.dtsi"
Makefile:196:                   msm8937-pmi8937-mtp.dtb \
```

#### 修改点3. 设置默认屏驱动(可选)

```c
msm8937-mtp.dtsi (kernel\msm-3.18\arch\arm64\boot\dts\qcom)
 
 &mdss_dsi0 {
+        qcom,dsi-pref-prim-pan = <&dsi_hsd_ili9881_720p_video>;
         pinctrl-names = "mdss_default", "mdss_sleep";
         pinctrl-0 = <&mdss_dsi_active &mdss_te_active>;
         pinctrl-1 = <&mdss_dsi_suspend &mdss_te_suspend>;
         qcom,platform-te-gpio = <&tlmm 24 0>;
         qcom,platform-enable-gpio = <&tlmm 99 0>;
         qcom,platform-reset-gpio = <&tlmm 60 0>;
         qcom,platform-bklight-en-gpio = <&tlmm 98 0>;
 };
 
+&dsi_hsd_ili9881_720p_video {
+        qcom,panel-supply-entries = <&dsi_panel_pwr_supply>;
+        qcom,mdss-dsi-pan-fps-update = "dfps_immediate_porch_mode_vfp";
+};
```

## 三. LCD 背光

### 3.1 lk 阶段背光驱动

#### 3.1.1 点亮背光位置

```c
// file: src/bootable/bootloader/lk/app/aboot/aboot.c

void aboot_init(const struct app_descriptor *app)
        target_display_init(device.display_panel);
                gcdb_display_init(oem.panel, MDP_REV_50, (void *)MIPI_FB_ADDR);
                        /* 根据 panel 的协议绑定对应的背光控制函数 */
                        if (pan_type == PANEL_TYPE_DSI)
                                panel.bl_func = mdss_dsi_bl_enable;
                        msm_display_init(&panel);
                                pdata->power_func(1, &(panel->panel_info);

static int mdss_dsi_bl_enable(uint8_t enable)
        panel_backlight_ctrl(enable);
                target_backlight_ctrl(panelstruct.backlightinfo, enable);
                        msm8952_wled_backlight_ctrl(enable);
                                qpnp_wled_enable_backlight(enable);
                                        /* #define QPNP_WLED_MAX_BR_LEVEL 1638 */
                                        qpnp_wled_set_level(gwled, QPNP_WLED_MAX_BR_LEVEL);
                                        qpnp_wled_enable(gwled, gwled->ctrl_base, enable);
                                qpnp_ibb_enable(enable);
```

### 3.2 kernel 阶段背光驱动

#### 3.2.1 对上抛出的接口

LCD 背光驱动是通过内核提供的 LED 子系统注册的驱动，因此对应用层抛出的接口创建在 `/sys/class/leds/` 目录下。

        C:\Users\wangbing> adb shell

        # 读取手机当前设置的背光等级
        android:/ # cat /sys/class/leds/lcd-backlight/brightness
        74

        # 设置背光等级为 100/255(手机变得更亮)
        android:/ # echo 100 > /sys/class/leds/lcd-backlight/brightness
        android:/ # cat /sys/class/leds/lcd-backlight/brightness
        100

如何查看 logcat log 确认当前设置的背光值, 可以通过 `DisplayPowerController: Animating brightness:` 关键字确认。

        C:\Users\wangbing>adb shell
        android:/ # logcat -s DisplayPowerController | grep "brightness"
        01-01 01:26:21.588  1206  1246 I DisplayPowerController: Animating brightness: target=102, rate=0, asusAnimator=false
        01-01 01:26:21.588  1206  1246 I DisplayPowerController: Animating brightness: target=102, rate=0, asusAnimator=false
        01-01 01:26:21.591  1206  1246 I DisplayPowerController: Animating brightness: target=102, rate=0, asusAnimator=false

#### 3.2.2 驱动注册流程

```c
// file: src/kernel/msm-3.18/drivers/video/msm/mdss/mdss_fb.c

module_init(mdss_fb_init);

int __init mdss_fb_init(void)
        platform_driver_register(&mdss_fb_driver);

static struct platform_driver mdss_fb_driver = {
        .probe    = mdss_fb_probe,
        .remove   = mdss_fb_remove,
        .suspend  = mdss_fb_suspend,
        .resume   = mdss_fb_resume,
        .shutdown = mdss_fb_shutdown,
        .driver   = {
                .name = "mdss_fb",
                .of_match_table = mdss_fb_dt_match,
                .pm = &mdss_fb_pm_ops,
        },
};

/* LED 设备，设备注册成功后，将会创建 /sys/class/leds/lcd-backlight 目录 */
static struct led_classdev backlight_led = {
        .name           = "lcd-backlight",              /* LED 设备的名字 */
        .brightness     = 127,                          /* 默认亮度为 127 */
        .brightness_set = mdss_fb_set_bl_brightness,    /* 设置亮度的函数 */
        .max_brightness = 255,                          /* 最大亮度为 255 */
};

/* 注册 LED 设备的位置 */
static int mdss_fb_probe(struct platform_device *pdev)
        backlight_led.brightness = mfd->panel_info->brightness_max;
        backlight_led.max_brightness = mfd->panel_info->brightness_max;
        if (led_classdev_register(&pdev->dev, &backlight_led))

/* 设置 LED 亮度的函数 */
static void mdss_fb_set_bl_brightness(struct led_classdev *led_cdev, enum led_brightness value)

        /* 如果传入的值比支持设置的最大值要大，则修改为支持的最大值 */
        if (value > mfd->panel_info->brightness_max)
                value = mfd->panel_info->brightness_max;

        /* 加了个 log 将 value 和 bl_lvl 打印出来，确认到这里的映射关系为: bl_lvl = value / 255 * 4096 */
        /* 说明这里是将上层传递过来的 0~255 的亮度等级转化为实际 PMIC 支持的亮度等级 */
        /* This maps android backlight level 0 to 255 into driver backlight level 0 to bl_max with rounding */
        MDSS_BRIGHT_TO_BL(bl_lvl, value, mfd->panel_info->bl_max, mfd->panel_info->brightness_max);

        /* 设置屏幕背光，最终还是调用 panel 中绑定的函数 */
        mdss_fb_set_backlight(mfd, bl_lvl);
                pdata->set_backlight(pdata, bl_lvl);

/* 搜索看看 set_backlight 在哪里绑定了操作函数 */
---- set_backlight Matches (13 in 5 files) ----
mdss_dsi_panel_init in mdss_dsi_panel.c (msm\mdss) : 	ctrl_pdata->panel_data.set_backlight = mdss_dsi_panel_bl_ctrl;
mdss_edp_device_register in mdss_edp.c (msm\mdss) : 	edp_drv->panel_data.set_backlight = mdss_edp_set_backlight;

/* 实际控制背光的位置在这里 */
static void mdss_dsi_panel_bl_ctrl(struct mdss_panel_data *pdata, u32 bl_level)

        /* 控制 PMI 的 WLED 引脚控制背光 */
        led_trigger_event(bl_led_trigger, bl_level);
```

#### 3.2.3 自问自答

问1: 背光驱动没有实现 brightness_get 接口，为什么却可以通过 cat /sys/class/leds/lcd-backlight/brightness 节点获得亮度值?  
答1: 看了下读 brightness 节点的函数，在读取 /sys/class/leds/lcd-backlight/brightness 节点时。
如果定义了 brightness_get 函数，则将会获取到的亮度值，传递给 backlight_led->brightness，然后将 backlight_led->brightness 值返回；
如果没有定义 brightness_get 函数，则直接返回 backlight_led->brightness 的值。

问2: 现在的疑问就变成了，为什么没有通过 brightness_get 获取亮度值，backlight_led->brightness 也是准确的?  
答2: 看了下写 brightness 节点的函数，在写入亮度值 /sys/class/leds/lcd-backlight/brightness 节点时。
同时将写入的值记录到了 backlight_led->brightness 变量中，因此读取的时候其实获取到的是上一次写入的值。

上述两个疑问涉及到的代码如下：

```c
// file: src/kernel/msm-3.18/drivers/leds/led-class.c

/* 定义了属性节点 */
static DEVICE_ATTR_RW(brightness);

/* 读属性时调用的函数 */
static ssize_t brightness_show(struct device *dev, struct device_attribute *attr, char *buf)
        |-- struct led_classdev *led_cdev = dev_get_drvdata(dev);
        |-- led_update_brightness(led_cdev);
        |       |
        |       |   // 如果定义了 brightness_get 函数，则将获取到的亮度值记录到 led_cdev->brightness
        |       |-- if (led_cdev->brightness_get)
        |                led_cdev->brightness = led_cdev->brightness_get(led_cdev);
        |
        |   // 这里直接将 led_cdev->brightness 返回
        |-- return sprintf(buf, "%u\n", led_cdev->brightness);

/* 写属性时调用的函数 */
static ssize_t brightness_store(struct device *dev, struct device_attribute *attr, const char *buf, size_t size)
        |-- struct led_classdev *led_cdev = dev_get_drvdata(dev);
        |-- ret = kstrtoul(buf, 10, &state);
        |-- __led_set_brightness(led_cdev, state);
                |
                |   // 先将写进的值保存到 led_cdev->brightness 中
                |-- led_cdev->brightness = value;
                |
                |   // 接着调用 brightness_set 函数控制 PMI 调整背光
                |-- led_cdev->brightness_set(led_cdev, value);
```

## 四. LCD 上电

### 4.1 LCD 上电前言

每一块 lcd ic 在 spec 上都会详细的描述它们上电的时序要求。以往工厂生产的时候，经常会出现由于上电时序不对或者是各路电之前的延时过短不符合 spec 规范导致的花屏、读不到 panel id 甚至烧坏 ic 的情况。因此在阅读屏驱动框架的时候，必须要准确的找到上电这块的代码位置。  
目前我们公司大部分的 LCD 都有两路电，一路是给 ic 供电的 IOVDD，另一路是加在玻璃上的偏置电压(VSP、VSN)，同时在 spec 上电时序的描述中，还会多加一路 ic 的 reset 信号，spec 上会给出这三者详细的上电掉电时序要求。  
我们要做的就是去 check 这个位置，看看是不是符合 spec 要求，不符合就将其修改。一般来说修改也仅仅是修改偏置电压，交换 reset 和偏置电压的前后顺序，需改时序间的延时之类的。

### 4.2 lk 阶段开机上电位置

```c
// file: src/bootable/bootloader/lk/app/aboot/aboot.c

void aboot_init(const struct app_descriptor *app)
        target_display_init(device.display_panel);
                gcdb_display_init(oem.panel, MDP_REV_50, (void *)MIPI_FB_ADDR);
                        /* 根据 panel 的协议绑定对应的上电函数 */
                        if (pan_type == PANEL_TYPE_DSI)
                                panel.power_func = mdss_dsi_panel_power;
                        msm_display_init(&panel);
                                pdata->power_func(1, &(panel->panel_info);

static int mdss_dsi_panel_power(uint8_t enable, struct msm_panel_info *pinfo)
        if (enable) {
                target_ldo_ctrl(enable, pinfo);
                mdss_dsi_panel_reset(enable);
        } else {
                mdss_dsi_panel_reset(enable);
                target_ldo_ctrl(enable, pinfo);
        }

int target_ldo_ctrl(uint8_t enable, struct msm_panel_info *pinfo)
        if (enable) {
                regulator_enable(ldo_num);      /* 使能 IOVDD */
                wled_init(pinfo);               /* 配置 VSP VSN 电压电流 */
                qpnp_ibb_enable(true);          /* 使能 VSP VSN */
        } else {
                ... ...
        }
```

### 4.3 kernel 阶段上下电位置

```
mdss_dsi_panel_power_ctrl
        mdss_dsi_panel_power_on
                msm_dss_enable_vreg
```

## 五. LCD 静电

### 5.1 使能 esd check 功能

屏 dtsi 中增加以下属性以开启 esd check 功能：

    &mdss_mdp {
        dsi_co55swr8_st7703_720p_video: qcom,dsi_co55swr8_st7703_720p_video {
            ... ...
            /* 使能 esd check 机制 */
            qcom,esd-check-enabled;

            /* 指定 esd check 的 command，cmd 最后一次字节是 check 的寄存器 */
            qcom,mdss-dsi-panel-status-command = [
                    14 01 00 01 05 00 01 68
                    06 01 00 01 05 00 01 09
                    14 01 00 01 05 00 01 CC
                    14 01 00 01 05 00 01 AF
            ];

            /* 指定 esd check 的命令模式 */
            qcom,mdss-dsi-panel-status-command-state = "dsi_lp_mode";

            /* 指定 esd check 的校验模式 */
            qcom,mdss-dsi-panel-status-check-mode = "reg_read";

            /* 在 command 中指定指定过了 check 的寄存器，这里指定 check 对应寄存器的前几个字节 */
            qcom,mdss-dsi-panel-status-read-length = <1 3 1 1>;

            /* 这个属性的含义还在摸索中 */
            qcom,mdss-dsi-panel-max-error-count = <2>;

            /* 指定 esd check 读取的寄存器的标准值 */
            qcom,mdss-dsi-panel-status-value = <0xC0> , <0x80 0x73 0x04>, <0x0B>,<0xFD>;
            ... ...
        };
    };

### 5.2 追踪 esd check 流程

在 mdss_dsi.c 文件中解析 dts 中屏节点中设置的 esd 相关属性，相关代码流程如下：

    // file: src/kernel/msm-3.18/drivers/video/msm/mdss/mdss_dsi.c
    static int mdss_dsi_ctrl_probe(struct platform_device *pdev)
        mdss_dsi_config_panel(pdev, index);

            /* 根据 panel_cfg 指定的 panel 找到对应的 dts 节点 */
            mdss_dsi_find_panel_of_node(pdev, panel_cfg);

            // file: src/kernel/msm-3.18/drivers/video/msm/mdss/mdss_dsi_panel.c
            mdss_dsi_panel_init(dsi_pan_node, ctrl_pdata, ndx);
                mdss_panel_parse_dt(node, ctrl_pdata);
                    mdss_dsi_parse_panel_features(np, ctrl_pdata);

                        /* 解析 dts 中 esd check 相关的参数 */
                        mdss_dsi_parse_esd_params(np, ctrl);

        dsi_panel_device_register(pdev, dsi_pan_node, ctrl_pdata);

            /* 根据 status_mode 绑定对应的 esd check 的函数 */
            if (ctrl_pdata->status_mode == ESD_REG)
                ctrl_pdata->check_status = mdss_dsi_reg_status_check;

    // file: src/kernel/msm-3.18/drivers/video/msm/mdss/mdss_dsi_status.c
    module_init(mdss_dsi_status_init);
        mdss_dsi_status_init
            INIT_DELAYED_WORK(&pstatus_data->check_status, check_dsi_ctrl_status);
                check_dsi_ctrl_status
                    pdsi_status->mfd->mdp.check_dsi_status(work, interval);
                    > mdss_check_dsi_ctrl_status
                        ctrl_pdata->check_status(ctrl_pdata);
                        > mdss_dsi_reg_status_check

                            /* 读取寄存器状态 */
                            mdss_dsi_read_status(ctrl_pdata);

                            /* 校验寄存器状态 */
                            ctrl_pdata->check_read_status(ctrl_pdata);
                            > mdss_dsi_gen_read_status(ctrl_pdata);
                                mdss_dsi_cmp_panel_reg_v2(ctrl_pdata);

```c
static void mdss_dsi_parse_esd_params(struct device_node *np, struct mdss_dsi_ctrl_pdata *ctrl)
{
    u32 tmp;
    u32 i, status_len, *lenp;
    int rc;
    struct property *data;
    const char *string;
    struct mdss_panel_info *pinfo = &ctrl->panel_data.panel_info;

    /* 获取 esd check 使能的标识位 */
    pinfo->esd_check_enabled = of_property_read_bool(np, "qcom,esd-check-enabled");

    /* 倘若未使能 esd check，此函数直接 return */
    if (!pinfo->esd_check_enabled)
        return;

    ctrl->status_mode = ESD_MAX;

    /* 获取 esd check 的 check mode */
    rc = of_property_read_string(np, "qcom,mdss-dsi-panel-status-check-mode", &string);

    /* 根据获取到的 check mode 设置好 ctrl->status_mode 为对应的宏 */
    if (!rc) {
        if (!strcmp(string, "bta_check")) {
            ctrl->status_mode = ESD_BTA;
        } else if (!strcmp(string, "reg_read")) {
            ctrl->status_mode = ESD_REG;
            ctrl->check_read_status = mdss_dsi_gen_read_status;
        } else if (!strcmp(string, "reg_read_nt35596")) {
            ctrl->status_mode = ESD_REG_NT35596;
            ctrl->status_error_count = 0;
            ctrl->check_read_status = mdss_dsi_nt35596_read_status;
        } else if (!strcmp(string, "te_signal_check")) {
            if (pinfo->mipi.mode == DSI_CMD_MODE) {
                ctrl->status_mode = ESD_TE;
            } else {
                pr_err("TE-ESD not valid for video mode\n");
                goto error;
            }
        } else {
            pr_err("No valid panel-status-check-mode string\n");
            goto error;
        }
    }

    /* 对于 ESD_BTA、ESD_BTA、ESD_MAX 这几种 check mode 不需要获取其他的属性参数 */ 
    if ((ctrl->status_mode == ESD_BTA) || (ctrl->status_mode == ESD_TE) || (ctrl->status_mode == ESD_MAX))
        return;

    /* 对于 ESD_REG 和 ESD_REG_NT35596 这两种 check 寄存器状态的模式，需要获取 check 哪些寄存器 */ 
    mdss_dsi_parse_dcs_cmds(np, &ctrl->status_cmds, "qcom,mdss-dsi-panel-status-command", "qcom,mdss-dsi-panel-status-command-state");

    rc = of_property_read_u32(np, "qcom,mdss-dsi-panel-max-error-count", &tmp);
    ctrl->max_status_error_count = (!rc ? tmp : 0);

    if (!mdss_dsi_parse_esd_status_len(np, "qcom,mdss-dsi-panel-status-read-length", &ctrl->status_cmds_rlen, ctrl->status_cmds.cmd_cnt)) {
        pinfo->esd_check_enabled = false;
        return;
    }

    if (mdss_dsi_parse_esd_status_len(np, "qcom,mdss-dsi-panel-status-valid-params", &ctrl->status_valid_params, ctrl->status_cmds.cmd_cnt)) {
        if (!mdss_dsi_parse_esd_check_valid_params(ctrl))
            goto error1;
    }

    status_len = 0;
    lenp = ctrl->status_valid_params ?: ctrl->status_cmds_rlen;
    for (i = 0; i < ctrl->status_cmds.cmd_cnt; ++i)
        status_len += lenp[i];

    data = of_find_property(np, "qcom,mdss-dsi-panel-status-value", &tmp);
    tmp /= sizeof(u32);
    if (!IS_ERR_OR_NULL(data) && tmp != 0 && (tmp % status_len) == 0) {
        ctrl->groups = tmp / status_len;
    } else {
        pr_err("%s: Error parse panel-status-value\n", __func__);
        goto error1;
    }

    ctrl->status_value = kzalloc(sizeof(u32) * status_len * ctrl->groups, GFP_KERNEL);
    if (!ctrl->status_value)
        goto error1;

    ctrl->return_buf = kcalloc(status_len * ctrl->groups, sizeof(unsigned char), GFP_KERNEL);
    if (!ctrl->return_buf)
        goto error2;

    rc = of_property_read_u32_array(np, "qcom,mdss-dsi-panel-status-value", ctrl->status_value, ctrl->groups * status_len);
    if (rc) {
        pr_debug("%s: Error reading panel status values\n", __func__);
        memset(ctrl->status_value, 0, ctrl->groups * status_len);
    }

    return;

error2:
    kfree(ctrl->status_value);
error1:
    kfree(ctrl->status_valid_params);
    kfree(ctrl->status_cmds_rlen);
error:
    pinfo->esd_check_enabled = false;
}

static int mdss_dsi_read_status(struct mdss_dsi_ctrl_pdata *ctrl)
{
    int i, rc, *lenp;
    int start = 0;
    struct dcs_cmd_req cmdreq;

    rc = 1;
    lenp = ctrl->status_valid_params ?: ctrl->status_cmds_rlen;

    for (i = 0; i < ctrl->status_cmds.cmd_cnt; ++i) {
        memset(&cmdreq, 0, sizeof(cmdreq));
        cmdreq.cmds = ctrl->status_cmds.cmds + i;
        cmdreq.cmds_cnt = 1;
        cmdreq.flags = CMD_REQ_COMMIT | CMD_REQ_RX;
        cmdreq.rlen = ctrl->status_cmds_rlen[i];
        cmdreq.cb = NULL;
        cmdreq.rbuf = ctrl->status_buf.data;

        if (ctrl->status_cmds.link_state == DSI_LP_MODE)
            cmdreq.flags |= CMD_REQ_LP_MODE;
        else if (ctrl->status_cmds.link_state == DSI_HS_MODE)
            cmdreq.flags |= CMD_REQ_HS_MODE;

        rc = mdss_dsi_cmdlist_put(ctrl, &cmdreq);
        if (rc <= 0) {
            pr_err("%s: get status: fail\n", __func__);
            return rc;
        }

        memcpy(ctrl->return_buf + start, ctrl->status_buf.data, lenp[i]);
        start += lenp[i];
    }

    return rc;
}

static bool mdss_dsi_cmp_panel_reg_v2(struct mdss_dsi_ctrl_pdata *ctrl)
{
    int i, j;
    int len = 0, *lenp;
    int group = 0;

    lenp = ctrl->status_valid_params ?: ctrl->status_cmds_rlen;
    
    for (i = 0; i < ctrl->status_cmds.cmd_cnt; i++)
        len += lenp[i];

    /* 这里比对寄存器中读取出来的值和 dts 中理论上寄存器中的值 */
    for (j = 0; j < ctrl->groups; ++j)
    {
        for (i = 0; i < len; ++i)
        {
            printk("lcd esd return_buf[%d] = 0x%x  status_value = 0x%x %s %d\n", i, ctrl->return_buf[i], ctrl->status_value[group + i],  __func__, __LINE__);
            if (ctrl->return_buf[i] != ctrl->status_value[group + i])
                break;
        }
        if (i == len)
            return true;
        group += len;
    }
    return false;
}
```

## 六. LCD 框架

### 6.1 lk 阶段 lcd 框架分析

对于 i2c、spi、usb 总线下的设备，写驱动程序的时候，涉及到两部分，分别是总线驱动和设备驱动。以 i2c 为例，总线驱动和设备驱动各自有对应的分工，总线驱动负责控制 i2c 控制器(片上外设)，将 i2c 控制器寄存器相关的控制逻辑封装成符合 i2c 规范的 start、stop、ask、sendbyte、readbyte 等接口函数。至于设备驱动，只需要根据设备的 datasheet 并按照总线的规范调用总线提供的接口函数，就可以实现和设备的通信。
和我们熟悉的 i2c、spi、usb 一样，mipi dsi 协议也是一样的，也分为总线驱动和设备驱动。我们分析框架，对于总线驱动部分只需要找到对应的接口函数即可，暂不去深入分析 mipi 总线驱动，重点看看点亮一块屏，设备驱动端上电时序，如下 init code，以及让 lcd 显示一帧图片需要如何操作。

未完待续。
