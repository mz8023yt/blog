---
title: '[ODM] 客退按电源键无法熄屏'
date: 2018-03-18 11:58:34
tags:
  - otg
  - mtk
categories: Experience
---

### 问题描述

最近做的一个项目客退一台手机，不良现象为：

1. 手机亮屏状态下，按 power key 无法熄屏。
2. 亮屏状态下，长按 power key 功能正常。
3. 手机接上 usb 或者 otg 线后，按 power key 可以正常熄屏。

### 解决思路

很明显，差异点在 usb 上，查看 usb 相关的 log，发现 otg 插拔的 log 不断的在打印，说明 otg 检测这块有问题。  

```
<4>[78943.472855]  (4)[12294:kworker/4:4] report the otg cable in  int event
<4>[78943.506079]  (4)[12294:kworker/4:4] report the otg cable out int event
<4>[78943.586288]  (4)[12294:kworker/4:4] report the otg cable in  int event
<4>[78943.613871]  (4)[12294:kworker/4:4] report the otg cable out int event
```

这里虽然只贴出了4句，其实打了4000句都不止，一直在打印。

使用万用表量取开机状态下的 usb 各个引脚的电压，正常机器和异常机器各个引脚的电压值如下表：

| USB Pin |  Normal  | Normal OTG | Normal charge | Normal PC |   Bad    | Bad usb charge |
| ------- | :------: | :--------: | :-----------: | :-------: | :------: | :------------: |
| VBUS    | 0V       | 5V         | 5V            | 5V        | 0.7-2.3V | 5V             |
| DM      | 0V       | 0V         | 0V            | 0V        | 0V       | 0V             |
| DP      | 0V       | 0V         | 0V            | 0V        | 0.35V    | 0V             |
| ID      | 1.8V     | 0V         | 1.8V          | 1.8V      | 0.35V    | 0V             |
| GND     | 0V       | 0V         | 0V            | 0V        | 0V       | 0V             |

这里的 VBUS 是在 0.7~2.3V 之间不断的跳变，而 USB host 会根据 VBUS 的边沿动作产生中断。

很明显，usb pin 硬件上出现了异常，看起来像 DP 和 ID 脚短接导致。  
用万用表量了一下，果然是 DP 和 ID 脚短接了。  
使用一台正常机器，短接 ID 和 DP 引脚，复现出不良现象。

### 背景知识

#### OTG 插拔亮屏需求

此项目有做 OTG 设备插拔时亮屏(唤醒屏幕)的需求。  
即，熄屏状态下插入或者拔出 OTG 设备，系统检测到 USB 设备则会唤醒屏幕。

#### OTG 设备检测原理

该设备支持OTG，下面说下设备的发现过程：

##### 作为从设备插入PC端口时：

1. 系统检测到VBUS上的XEINT28上升沿触发中断，因为PC端会有一个5V从VBUS给过来，进入中断处理函数进一步确认ID脚状态，ID脚为低则状态错误，ID脚为高表示设备应该切换到从设备模式
2. 通知usb gadget使能vbus，按照device模式使能PHY。gadget在probe时注册了一个SPI软中断IRQ_USB_HSOTG，用于响应数据接收
3. 开启usb clk，使能PHY，此时外部5V电源供给系统XuotgVBUS，gadget收到IRQ_USB_HSOTG中断要求重启OTG core
4. USB DP（高速设备为DP，低速设备为DM）上产生一个高电平脉冲，此时PC识别到一个USB设备插入，windows会提示用户
5. 后续就是SETUP，GET DISCRIPTOR的过程

##### 作为主设备发现设备插入时：
1. 系统检测到ID脚上XEINT29下降沿触发中断（实际是插入的usb公口第四脚直接连接到第五脚地上面），进入中断处理，切换到主设备模式
2. 关中断，使能DC5V给VBUS上电，唤醒ehci与ohci
3. usb core在内核初始化时注册了一个名为khubd的内核线程，由khubd监控port event。（实际过程我理解是从设别由VUBS供电后，会在DP或DM上产生一个高电平脉冲
ehci在接收到脉冲信号后识别到设备插入，仅仅是理解，这一点未验证）
3. khubd获取port，speed后交给ehci，接下来就是usb的SETUP，GET DISCRIPTOR过程

#### MTK 回复

USB设备插入拔出的动作，分Host、Device两种情况：  

1. 手机作为device，通过USB线连接到Host（比如PC），此时VBUS电压由PC提供，产生中断给手机，然后手机与PC端进行USB握手交互，
2. 手机作为Host，通过USB OTG线连接到USB设备（比如u盘、鼠标），此时外设端将USB_ID信号接地，产生中断给手机，然后手机端通过Boost芯片产生5V VBUS电压给USB设备，并进行USB握手交互。


### 分析结论

因此，此问题不是 power key 的问题，而是因为 DP 和 ID 短接，导致不断的触发 OTG 的插拔动作，OTG 不断的唤醒屏幕，导致屏幕无法熄屏。  
这也就解释了，为什么手机接上 usb 或者 otg 线后，按 power key 就可以正常熄屏。  
看表格中最后一列，接上 USB 设备后，USB 各个pin的电平是稳定的，不会触发OTG检测机制。

