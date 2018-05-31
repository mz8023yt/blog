---
title: '[ODM] TP 相关 debug 实例'
date: 2018-05-31 09:14:55
tags:
  - touch
  - qcom
categories: ODM
---

### 前言

TP 这块出现问题，条件允许的话，一定要第一时间拿到不良机，使用不良机复现问题，在线调试。这里说的在线调试是指通过 adb 实时打印 log。  
实时打印 kernel log 可以通过下面几条命令：

    dmesg -w | grep -E <keyword>
    logcat -b kernel | grep -E <keyword>

实时打印 kernel log 可以通过下面几条命令：

    logcat | grep -E <keyword>

上面使用的是 grep -E 选项，因此 keyword 可以使用正则表达式进行过滤。

### 实例1：手势失效

工厂出现 200pcs 手势功能失效，无法唤醒手势或打开手势对应的 app，从工厂协调 2pcs 不良机，分析过程如下：

1. 确认上层手势开关设置是否生效  
   在设置界面中设置手势的开关，看底层 TP 手势 proc 节点的 log 有没有打印  

       [ 3834.582866] [Paul][HXTP][himax_GESTURE_write][2966] gesture flag: 01111111

   确认到有相关 log 打印，从上述 log 确认到上层设置手势开关生效了  

2. 确认 TP 睡眠时有没有成功进入手势模式  
   按 power key 灭屏，看 suspend log 确认是否进入手势模式  

       [ 3858.094688] [HXTP] [himax] himax_chip_common_suspend: SMART_WAKEUP enable, reject suspend

   从上述 log 确认到进入了手势模式  

3. 确认是否有手势键值上报  
   进入手势模式了，尝试双击看看有没有 event 事件上报。  
   多次画手势动作，均没有看到手势上报对应的 log，确认到没有事件上报  

4. 确认熄屏手势 TP 是否产生中断  
   查看 /proc/interrupts 节点，看看有没有检测到 TP 的中断  
   熄屏状态下画手势，TP 中断计数没有增加，确认到没有中断产生  
   怀疑 TP 没有识别到手势，因此没有触发中断。  

5. 此问题需要联系 FAE 进一步分析，分析是否是 TP 固件侧的问题  
   最后确认到是固件侧没有识别到手势，没有触发手势中断  
   TP 供应商优化固件后，将 TP 扫描方式从自容扫描修改为互容扫描，提高手势识别率解决了此问题  

### 实例2：固件升级失败

供应商发布 0x8 版本的 TP 固件需要合入临时版本给测试验证，出现升级失败的问题。分析过程如下：

1. 0x7 版本三合一刷 0x8 版本固件 --> 升级失败  
   将 0x8 版本固件合入软件中，刷机第一次开机，log 中正确读取了 IC 和 版本中的固件版本号，但是升级函数返回值出错，升级失败  

       [    5.153623] [wjwind] current name = st7703 720p video mode dsi panel
       [    5.159843] [wjwind] IC fw =0xe502 conf = 0x7 Firmware: fw = 0xe502, conf = 0x8
       [    7.207020] [HXTP] irq_enable_count = 0[HXTP][ERROR] i_update_FW: TP upgrade error, upgrade_times = 1
       [    7.216221] [HXTP][ERROR] i_update_FW: TP upgrade error, upgrade_times = 2
       [    7.429449] [HXTP][ERROR] i_update_FW: TP upgrade error, upgrade_times = 3

2. 0x7 版本三合一刷 0x8 版本固件 --> 升级失败  
   重新启动，看看第二次重启是否 ok，报错同上  

3. 0x6 版本三合一刷 0x8 版本固件 --> 升级失败  
   将三合一通过 bat 脚本强制刷到 0x6 版本固件，使用 0x8 固件的软件版本开机，log 中正确读取了 IC 和 版本中的固件版本号，升级还是失败，报错同上  

       [    5.413623] [wjwind] current name = st7703 720p video mode dsi panel
       [    5.419842] [wjwind] IC fw =0xe502 conf = 0x6 Firmware: fw = 0xe502, conf = 0x8
       [    7.418033] [HXTP] irq_enable_count = 0[HXTP][ERROR] i_update_FW: TP upgrade error, upgrade_times = 1
       [    7.418035] [HXTP][ERROR] i_update_FW: TP upgrade error, upgrade_times = 2
       [    7.418037] [HXTP][ERROR] i_update_FW: TP upgrade error, upgrade_times = 3

4. 0x6 版本三合一刷 0x7 版本固件 --> 升级成功
   使用上一步刷的 0x6 版本的三合一，刷 0x7 固件的软件版本，开机，升级流程正常

       [    4.768697] [wjwind] current LCM = st7703 720p video mode dsi panel
       [    5.199845] [wjwind] IC fw =0xe502 conf = 0x6 Firmware: fw = 0xe502, conf = 0x7
       [   20.428222] [HXTP] i_update_FW: TP upgrade OK

5. 怀疑无法升级至 0x8 版本，是 0x8 版本固件存在问题  
   通过 log 发现，log 中读取的固件长度相比于正常固件的大小偏小，最后确认到是因为我使用的是 Foxmail 邮件客户端，收取固件的时候，部分数据丢失。供应商发出的是 165KB 大小固件文件，我通过 Foxmail 接收到的固件只有 125KB，将固件压缩后重发，Foxmail 接收到后并解压得到 165KB 的固件，使用完整的固件升级成功

### 实例3：手势失效

部件出现两台手势失效的机器，参考实例1的分析过程，实例3分析过程如下：

1. 确认到上层手势开关设置生效  
2. 确认到 TP 睡眠时有进入手势模式  
3. 确认到没有手势事件上报  
4. 确认到有中断计数，TP 产生了中断  
5. 查看 TP 中断中的 log，发现异常 log 如下：  

       [ 3858.094688] [HXTP][HIMAX TP MSG] checksum fail : checksum_cal: 0x49C
       [ 3858.636112] [HXTP][HIMAX TP MSG] checksum fail : checksum_cal: 0x7F8
       [ 3858.706841] [HXTP][HIMAX TP MSG] checksum fail : checksum_cal: 0x4C4

   查看代码发现，这里 checksum 函数校验的是 TP i2c 回传的数据 buf，在手势模式下会去校验 buf 的格式是否满足手势键值的格式。若发现不符合手势数据 buf 格式，则会打印 log  

6. 约供应商一起分析此问题，复现问题的状态下，通过 proc/android_touch/register 节点查看 8F 状态寄存器  
   确认到 8F 寄存器的值为 0xA0，正常情况进入手势的话，8F 寄存器的值应该是 0x20

   这里介绍下，Himax 的 TP 有三种工作状态，分别是：normal mode([8F] = 0x00)，gesture mode([8F] = 0x20)，hsen mode([8F] = 0x40)。括号中是对应的 8F 寄存器的值，8F 寄存器是工作状态寄存器，其中位5是手势标识位，位6是手套标识位，其他位是保留位，没有使用。  
   TP 在 suspend 函数中会先读取 8F 寄存器中的值，然后根据驱动中的手势使能标识和手套使能标识修改 8F 寄存器的位5和位6，将修改后的值写回 8F 寄存器中。

   在读写 8F 寄存器的位置添加 log 后复现问题，确认到：  
   正常情况下，TP suspend 读取 tp 的 8F 寄存器是读取到 0x00，将手势标识位位5置位后，得到 0x20，写回 TP 中，TP 进入手势模式。  
   异常情况1，读取到的值是 0x80，将位5置位后，得到 0xA0，写回 TP 中，TP 工作状态异常。  
   异常情况2，读取到的值是 0x00，将位5置位后，得到 0x20，写回 TP 中，TP 值没有生效，还是 0x00 。TP 工作状态异常。  
   在上述两种异常情况下，TP 固件侧都未处于手势模式，因此回传的 buf 自然校验不过，就出现了 checksum fail 的情况。  

   问题定位到是 TP 状态寄存器值异常，和 FAE 确认到，驱动在 suspend 中，针对手势和手套有做两次分别读写 8F 的寄存器的动作，在 himax_set_SMWP_enable 手势操作完 8F 寄存器后，此时固件也会去操作 8F 寄存器，获取 TP 的最新状态，固件操作 8F 可能会和 himax_set_HSEN_enable 同时操作到 8F 寄存器，产生冲突，导致值出现异常。  
   供应商有做实验，强制高频率让驱动和固件同时操作 8F 寄存器，看是否高概率出现 8F 寄存器的值不稳定，确认到确实存在这样的情况。最后拟定解决方案为：将手势和手套单独读写修改标识位的方式，修改为一起判断标识位，只做一次读写。具体修改 diff 如下：

   ```
   diff --git a/drivers/input/touchscreen/HX8527/himax_debug.c b/drivers/input/touchscreen/HX8527/himax_debug.c
   index 4f0ea5b..ed3e9a8 100755
   --- a/drivers/input/touchscreen/HX8527/himax_debug.c
   +++ b/drivers/input/touchscreen/HX8527/himax_debug.c
   @@ -2811,7 +2811,8 @@ static ssize_t himax_HSEN_write(struct file *file, const char *buff,
        else
            return -EINVAL;
    
   -    himax_set_HSEN_enable(ts->client, ts->HSEN_enable, ts->suspended);
   +    // himax_set_HSEN_enable(ts->client, ts->HSEN_enable, ts->suspended);
   +    himax_set_work_status(ts->client, ts->SMWP_enable, ts->HSEN_enable, ts->suspended);
    
        I("%s: HSEN_enable = %d.\n", __func__, ts->HSEN_enable);
    
   @@ -2878,7 +2879,8 @@ static ssize_t himax_SMWP_write(struct file *file, const char *buff,
        else
            return -EINVAL;
    
   -    himax_set_SMWP_enable(ts->client, ts->SMWP_enable, ts->suspended);
   +    // himax_set_SMWP_enable(ts->client, ts->SMWP_enable, ts->suspended);
   +    himax_set_work_status(ts->client, ts->SMWP_enable, ts->HSEN_enable, ts->suspended);
    
        HX_SMWP_EN = ts->SMWP_enable;
        paul("SMART_WAKEUP_enable = %d", HX_SMWP_EN);
   diff --git a/drivers/input/touchscreen/HX8527/himax_ic.c b/drivers/input/touchscreen/HX8527/himax_ic.c
   index 6611e90..b26ce10 100755
   --- a/drivers/input/touchscreen/HX8527/himax_ic.c
   +++ b/drivers/input/touchscreen/HX8527/himax_ic.c
   @@ -901,6 +901,25 @@ test_retry:
        return pf_value;
    }
    
   +void himax_set_work_status(struct i2c_client *client, uint8_t SMWP_enable, uint8_t HSEN_enable, bool suspended)
   +{
   +    uint8_t buf[4];
   +    i2c_himax_read(client, 0x8F, buf, 1, DEFAULT_RETRY_CNT);
   +
   +    if(SMWP_enable == 1 && suspended)
   +        buf[0] = 0x20 ;
   +    else
   +        buf[0] = 0x00;
   +
   +    if(HSEN_enable == 1 && !suspended)
   +        buf[0] = 0x40;
   +    else
   +        buf[0] &= 0xBF;
   +
   +    if ( i2c_himax_write(client, 0x8F,buf, 1, DEFAULT_RETRY_CNT) < 0)
   +        E("%s i2c write fail.\n",__func__);
   +}
   +
    void himax_set_HSEN_enable(struct i2c_client *client, uint8_t HSEN_enable, bool suspended)
    {
        uint8_t buf[4];
   @@ -2268,12 +2293,16 @@ void himax_resend_cmd_func(bool suspended)
    #endif
        }
    
   -#ifdef HX_SMART_WAKEUP
   -    himax_set_SMWP_enable(ts->client,ts->SMWP_enable,suspended);
   -#endif
   -#ifdef HX_HIGH_SENSE
   -    himax_set_HSEN_enable(ts->client,ts->HSEN_enable,suspended);
   -#endif
   +// #ifdef HX_SMART_WAKEUP
   +//     himax_set_SMWP_enable(ts->client,ts->SMWP_enable,suspended);
   +// #endif
   +// #ifdef HX_HIGH_SENSE
   +//     himax_set_HSEN_enable(ts->client,ts->HSEN_enable,suspended);
   +// #endif
   +
   +    himax_set_work_status(ts->client, ts->SMWP_enable, ts->HSEN_enable, suspended);
   +
    #ifdef HX_USB_DETECT_GLOBAL
        himax_cable_detect_func(true);
    #endif
   diff --git a/drivers/input/touchscreen/HX8527/himax_ic.h b/drivers/input/touchscreen/HX8527/himax_ic.h
   index 21361f1..4e0c286 100755
   --- a/drivers/input/touchscreen/HX8527/himax_ic.h
   +++ b/drivers/input/touchscreen/HX8527/himax_ic.h
   @@ -72,6 +72,7 @@ enum fw_image_type
    int himax_hand_shaking(struct i2c_client *client);
    void himax_set_SMWP_enable(struct i2c_client *client,uint8_t SMWP_enable, bool suspended);
    void himax_set_HSEN_enable(struct i2c_client *client,uint8_t HSEN_enable, bool suspended);
   +void himax_set_work_status(struct i2c_client *client, uint8_t SMWP_enable, uint8_t HSEN_enable, bool suspended);
    void himax_usb_detect_set(struct i2c_client *client,uint8_t *cable_config);
    int himax_determin_diag_rawdata(int diag_command);
    int himax_determin_diag_storage(int diag_command);
   ```

