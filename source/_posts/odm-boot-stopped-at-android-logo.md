---
title: '[ODM] 开机卡死在 android logo 界面'
date: 2018-02-9 19:25:48
tags:
  - boot
  - system
  - mtk
categories: Experience
---

### 问题描述

最近做的一个 MTK 平台的项目客退一台手机，不良现象为: 开机卡死在 android logo 界面，无法正常进入系统。

### 背景知识

android 手机开机一般都有三帧 boot logo，第一帧是 "Power by android"，第二帧一般是各个手机品牌商的品牌 logo，第三帧就是定制的开机动画了。  
此问题是没有跳转到第二帧，卡死在第一帧，所以关键看第一帧到第二帧期间哪里出了问题。

贴出同事总结的 Android logo 加载流程:

```c
[第一张 logo 显示在 lk 启动流程中介绍]
[第二张 logo 初始化显示位置]
alps\device\mediatek\mt6755\init.mt6755.rc
# Update the second boot logo
service bootlogoupdater /vendor/bin/boot_logo_updater
        class core
        oneshot
    
// vendor\mediatek\proprietary\external\boot_logo_updater\boot_logo_updater.c
main(void)
        // 获得启动模式，即启动的原因，并设置相应的属性节点值
        int ret = update_boot_reason();
                fd = open(BOOT_MODE_PATH, O_RDWR);                      // /sys/class/BOOT/BOOT/boot/boot_mode
                s = read(fd, (void *)&boot_mode, sizeof(boot_mode));
                property_get(BOOT_PACKAGE_SYS_PROPERTY, propVal, "0");  // sys.boot.reason"
                property_set(BOOT_REASON_SYS_PROPERTY, propVal);        // persist.sys.bootpackage
        
        // 设置背光亮度
        set_int_value(LCD_BACKLIGHT_PATH, 120);                         // /sys/class/leds/lcd-backlight/brightness
                write_to_file(path, buf, strlen(buf));    
                        int fd = open(path, O_RDWR);
                        int count = write(fd, buf, size);
                        close(fd);
        
        // 都不在文件中了：查看 Android.mk 
        // LOCAL_C_INCLUDES += $(LOCAL_PATH)/../libshowlogo/
        // 最终此文件中找到了对应的函数实现，代码路径为：
        // vendor\mediatek\proprietary\external\libshowlogo
        set_draw_mode(DRAW_ANIM_MODE_FB);
                //Charging_animation.cpp
                set_draw_mode(int draw_mode)
                        // 设置显示模式全局变量，有两种显示模式
                        // #define DRAW_ANIM_MODE_FB       1
                        // #define DRAW_ANIM_MODE_SURFACE  0
                        draw_anim_mode = draw_mode;
        
        // 获得对应平台的 fstab 文件，获得 logo.img 挂载位置，然后读取 logo 
        // 然后初始化显示设备，有两种显示方式：
        //   1. 直接在 fb 内核中显示
        //   2. 在 surface 中显示 
        anim_init();
                //Charging_animation.cpp            
                anim_init()
                        // 获得对应平台的 fstab 文件，然后获得其挂载分区 /logo 
                        // 读取对应的 logo 镜像文件
                        anim_logo_init();
                                int ret = property_get("ro.hardware", propbuf, "");
                                snprintf(fstab_filename, sizeof(fstab_filename), FSTAB_PREFIX"%s", propbuf);
                                fstab = fs_mgr_read_fstab(fstab_filename);
                                rec = fs_mgr_get_entry_for_mount_point(fstab, LOGO_MNT_POINT);
                                fd = open(rec->blk_device, O_RDONLY);
                                logo_addr = (unsigned int*)malloc(LOGO_BUFFER_SIZE);
                                // (1) skip the image header
                                len = read(fd, logo_addr, 512);
                                // get the image
                                len = read(fd, logo_addr, LOGO_BUFFER_SIZE - 512);
                                close(fd);
    
                        // 1. 在 fb 中显示 logo                
                        if (draw_anim_mode == (DRAW_ANIM_MODE_FB)) {
                                anim_fb_init();
                                    // 打开 fb0, 
                                    fb_fd = open(FB_NODE_PATH, O_RDWR);
                                    
                                    // 获得屏相关的信息参数
                                    ioctl(fb_fd, FBIOGET_VSCREENINFO, &vinfo);
                                    ioctl(fb_fd, FBIOGET_FSCREENINFO, &finfo);
                                    
                                    // 将显示屏映射到进程的地址空间中
                                    fb_size  = finfo.line_length * vinfo.yres;
                                    dec_logo_addr = (unsigned int*) malloc(fb_size);
                                    lk_fb_addr =(unsigned int*)mmap(0, fb_size*3, PROT_READ | PROT_WRITE, MAP_SHARED, fb_fd, 0);
                                    charging_fb_addr = (unsigned int*)((unsigned int)lk_fb_addr + fb_size);
                                    kernel_fb_addr = (unsigned int*)((unsigned int)charging_fb_addr + fb_size);
                                    fb_addr = lk_fb_addr;
                                    
                                    // 设置本地的屏的相关参数
                                    phical_screen.bits_per_pixel = vinfo.bits_per_pixel;
                                    phical_screen.fill_dst_bits = vinfo.bits_per_pixel;
                                    phical_screen.red_offset = vinfo.red.offset;
                                    phical_screen.blue_offset = vinfo.blue.offset;
                                    phical_screen.width = vinfo.xres;
                                    phical_screen.height = vinfo.yres;
                                    phical_screen.allignWidth = finfo.line_length/(vinfo.bits_per_pixel/8);
                                    phical_screen.needAllign = 1;
                                    phical_screen.need180Adjust = 1;
                                    phical_screen.fb_size = fb_size;
                                    
                                    // 设置旋转参数
                                    if(0 == strncmp(MTK_LCM_PHYSICAL_ROTATION, "270", 3))
                                        。。。
                                        phical_screen.rotation = 270;
                                    
                        // 2. 在 surface 中显示 logo 初始化
                        } else {
                            anim_surface_init();
                                    surfaceControl = client->createSurface(String8("charging-surface"), dinfo_width,  dinfo_height, PIXEL_FORMAT_BGRA_8888);
                                    SurfaceComposerClient::openGlobalTransaction();
                                    surfaceControl->setLayer(2000000);
                                    SurfaceComposerClient::closeGlobalTransaction();
                                    
                                    // data structure to access surface content
                                    surface = surfaceControl->getSurface();
                                    phical_screen.width = dinfo.w;
                                    phical_screen.height = dinfo.h;
                                    
                                    // for we adjust the roration avove, so no need to consider rotation
                                    phical_screen.rotation = 0;
                                    phical_screen.need180Adjust = 0;
                                                                    
                                    ANativeWindow* window = surface.get();
                                    status_t result = native_window_api_connect(window, NATIVE_WINDOW_API_CPU);
                                    err = window->dequeueBuffer(window, &buf, &fenceFd);
                                    sp<Fence> mFence(new Fence(fenceFd));
                                    mFence->wait(Fence::TIMEOUT_NEVER);
                                    gb = new GraphicBuffer(buf, false);
                                    phical_screen.needAllign = 1;
                                    phical_screen.allignWidth = gb->getStride();
                                    window->cancelBuffer(window, buf, fenceFd);
                                    // use PIXEL_FORMAT_RGBA_8888 for surface flinger
                                    phical_screen.bits_per_pixel = 32;
                                    phical_screen.fill_dst_bits = 32;
                                    fb_size  = dinfo.w * dinfo.h* 4;
                                    dec_logo_addr = malloc(fb_size);
                                    phical_screen.fb_size = fb_size;
                                                                
                        }
                    
        // 显示 logo, 其实就是
        //      1. 从 logo.img 中读出对应的 logo 
        //      2. 将 logo 填充到显示缓冲区中即可
        show_kernel_logo();
                anim_show_logo(kernel_logo_position);       // 此参数为 kernel logo 在 logo.img 中的 index  ，static int kernel_logo_position = KERNEL_LOGO_INDEX ;
                        if (draw_anim_mode == (DRAW_ANIM_MODE_FB)) {
                            anim_set_buffer_address(index);
                            fill_animation_logo(index, fb_addr, dec_logo_addr, logo_addr,phical_screen);        // 显示图片
                            anim_fb_disp_update();
                         } else {
                            ARect tmpRect;
                            tmpRect.left = 0;
                            tmpRect.top = 0;
                            tmpRect.right = phical_screen.width;
                            tmpRect.bottom = phical_screen.height;

                            status_t  lockResult = surface->lock(&outBuffer, &tmpRect);

                            SLOGD("[libshowlogo: %s %d]outBuffer.bits = %d\n",__FUNCTION__,__LINE__, (int)outBuffer.bits);
                            SLOGD("[libshowlogo: %s %d]surface->lock return =  0x%08x,  %d\n",__FUNCTION__,__LINE__,lockResult,lockResult);
                            if (0 == lockResult)
                            {
                                fill_animation_logo(index, (void *)outBuffer.bits, dec_logo_addr, logo_addr,phical_screen);// 显示图片
                                surface->unlockAndPost();
                            }
                         }
        // 相关资源的释放        
        anim_deinit();   
                anim_logo_deinit();
                        free_fstab();
                        free(logo_addr);
                        logo_addr = NULL;
                        free(dec_logo_addr);
                if (draw_anim_mode == (DRAW_ANIM_MODE_FB)) {
                    anim_fb_deinit();
                        munmap(lk_fb_addr, fb_size*3);
                        close(fb_fd);
                } else {
                    anim_surface_deinit();
                            surface = surfaceControl->getSurface();
                            ANativeWindow* window = surface.get();
                            native_window_api_disconnect(window,NATIVE_WINDOW_API_CPU);
                            surfaceControl->clear();
                            client->dispose();
                }
```

### 分析思路

(1) 先回读镜像，回读了 pl lk boot(但不知道手机的软件版本)，通过和正式版本的镜像对比，发现回读的镜像和 WW B18 一致，确认到手机软件版本为 WW B18。
(2) 和 WW B18 的镜像对比，发现 pl lk boot 镜像和版本中的镜像一致，没有损坏。
(3) 焊接串口，抓取串口 log，通过 fastboot 在 user 版本下抓取完整的开机 uart log。

```
C:\Users\wangbing>fastboot oem p2u on
C:\Users\wangbing>fastboot continue
```

(4) 对比不良机和正常机开机串口 log，发现，不良机在 init 进程中找不到 boot_logo_updater，没有正常启动 boot_logo_updater 程序(此服务用于加载 kernel 阶段的 boot logo，对应开机的客户品牌 logo 图片)。

```
[    5.119985] <5>.(5)[1:init]init: Starting service 'healthd'...
[    5.121544] <5>.(5)[1:init]init: cannot find '/vendor/bin/boot_logo_updater' (No such file or directory), disabling 'bootlogoupdater'
[    5.123102] <5>.(5)[1:init]init: cannot find '/vendor/bin/wmt_loader' (No such file or directory), disabling 'wmt_loader'
[    5.124484] <5>.(5)[1:init]init: cannot find '/vendor/bin/wmt_launcher' (No such file or directory), disabling 'wmt_launcher'
[    5.125082] <5>.(4)[314:healthd]binder: 314:314 binder_context_mgr_node is NULL
[    5.125087] <5>.(4)[314:healthd]binder: 314:314 transaction failed 29189, size 0-0
[    5.125094] <5>.(4)[314'(ndor/bin/ccci_mdinit' (No such file or directory), disabling 'ccci_mdinit'
[    5.133184] <5>.(5)[1:init]init: cannot find '/vendor/bin/ccci_fsd' (No such file or directory), disabling 'ccci3_fsd'
[    5.134526] <5>.(5)[1:init]init: cannot find '/vendor/bin/ccci_mdinit' (No such file or directory), disabling 'ccci3_mdinit'
[    5.135973] <5>.(5)[1:init]init: cannot find '/vendor/bin/ccci_rpcd' (No such file or directory), disabling 'ccci_rpcd'
[    5.137324] <5>.(5)[1:init]init: cannot find '/vendor/bin/terservice' (No such file or directory), disabling 'terservice'
```

(5) boot_logo_updater 服务编译版本后会打包在 /system/vendor/bin 目录下，而 /vendor/bin/ 是通过软链接方式链接到 /system/vendor/bin 目录的。

```
C:\Users\wangbing>adb shell
android:/ # ls -l vendor
lrwxrwxrwx 1 root root 14 1970-01-01 08:00 vendor -> /system/vendor
```

(6) boot_logo_update 程序打不开，有两个怀疑点，要么是 /system/vendor/bin 目录下的执行文件有问题，要么是从 /vendor/ 到 /system/vendor/ 的软链接没有生效。
(7) 试图连接下 adb，想确认下软连接有没有生效，很遗憾，无法连接。
(8) 回读不良机的 system 镜像，使用 ext2explore.exe 解压确认，发现不良原因确实是 boot_logo_updater 损坏，导致无法加载第二帧 logo。

### 分析结论

客退机 system 分区损坏，导致无法加载 boot_logo_updater 程序引导第二帧 boot logo；同时 init 进程无法成功执行，导致无法开机。
