---
title: '[ODM] TP 手势唤醒屏幕亮起来慢'
date: 2018-04-20 10:53:17
tags:
  - touch
  - gesture
  - qcom
categories: Experience
---

### 问题描述

客户测试人员报出 bug，使用 TP 手势唤醒手机，屏幕亮起来慢，而同平台的对比机不存在此问题。

### 解决思路

由于整个屏幕唤醒的流程非常复杂，这里先将其简单划分为三块，(1)TP 识别手势上报手势键值 -> (2)上层处理 TP 上报的手势键值 -> (3)上层根据键值唤醒屏幕或者调起对应的 apk。  
不妨先做相关的对比试验看看具体慢是慢在哪一块？是 TP 识别手势耗时，还是上层处理手势值耗时，还是屏幕唤醒流程相比于对比机就是耗时？  
做实验对比 power key 唤醒和双击手势唤醒的时间，看看时间是否差不多。时间怎么看，需要找到亮屏瞬间对应的 log，手势上报键值瞬间对应的 log，以及 power key 按下瞬间对应的 log。使用这些 log 前面的间戳就可以计算出之间的耗时。那对应的 log 怎么找？可以这么干，使用 dmesg -w 或者 logcat -b kernel 实时打印 kernel log 出来，看看按 power key、熄屏双击以及亮屏瞬间都有哪些 log 打印，再从这些打印的 log 中去找当前动作对应的关键 log。  
最后发现亮屏瞬间 kernel 中没有 log，通过 kernel log 仅找到手势上报键值和按 power key 对应的 log。kernel 中没有亮屏对应 log 的话，那没有办法了，只能通过 logcat 抓取亮屏瞬间对应的 main log，最后找到的各个动作的对应关系为：

```bash
# power key
[  804.903664] [wind-tick][kpdpwr] start add timer
[  805.053863] [wind-tick][kpdpwr] start del timer

# input keycode
[  225.105052] [Paul][TP][himax_wake_check_func][674] ret_event = 128 enable = 1
[  225.105060] [Paul][TP][himax_wake_check_func][692] gesture: double click

# screen on
PowerSaverService: mIsScreenOn = true
```

由于是性能问题，计算时间的话不能采用 eng 版本做实验，必须使用 user 版本。在 user 版本可以通过这条命令同时抓取 kernel log 和 main log。

```bash
C:\Users\wangbing>adb shell
android$ logcat -b all | grep -E 'Paul|kpdpwr|mIsScreenOn'
```

对于手势和电源键分别抓取了 10 组数据，截取其中 3 组 log 如下：

```bash
## input keycode
1-01 20:51:44.957     0     0 E [Paul][TP][himax_wake_check_func][692] gesture: double click
1-01 20:51:45.676  2503  2503 D PowerSaverService: mIsScreenOn = true

1-01 20:51:54.395     0     0 E [Paul][TP][himax_wake_check_func][692] gesture: double click
1-01 20:51:55.124  2503  2503 D PowerSaverService: mIsScreenOn = true

1-01 20:52:19.116     0     0 E [Paul][TP][himax_wake_check_func][692] gesture: double click
1-01 20:52:19.848  2503  2503 D PowerSaverService: mIsScreenOn = true

## power key
01-01 20:44:12.082     0     0 W [wind-tick][kpdpwr] start add timer
01-01 20:44:12.255  2503  2503 D PowerSaverService: mIsScreenOn = true

01-01 20:44:16.933     0     0 W [wind-tick][kpdpwr] start add timer
01-01 20:44:17.153  2503  2503 D PowerSaverService: mIsScreenOn = true

01-01 20:44:21.748     0     0 W [wind-tick][kpdpwr] start add timer
01-01 20:44:21.955  2503  2503 D PowerSaverService: mIsScreenOn = true
```

通过 10 组数据最后计算出来的平均值，发现手势唤醒需要 700ms 左右，而 power key 唤醒只需要 200ms 左右。  
说明: "(3)上层根据键值唤醒屏幕" 这块耗时在 200ms 左右，多出来 500ms 在 "(1)TP 识别手势上报手势键值 -> (2)上层处理 TP 上报的手势键值" 这一段。然而对于手势唤醒 log 计算的时间 700ms 中并没有包含 "(1)TP 识别手势上报手势键值" 的时间，这个 700ms 是从手势上报键值的位置开始计算的，说明多出来的 500ms 仅仅是耗费在 "(2)上层处理 TP 上报的手势键值" 这块。

此问题，转给上层同事分析。

### 背景知识

口袋模式是指在 TP 上报键值后，上层也接收到了，即将响应手势键值的时候，会去读取 psensor 的值，如果 psensor 的值过大，说明此时 psensor 是被遮挡的状态，则认为是在口袋中的 TP 误触行为，不去响应手势；否则正常响应。

### 分析结论

最后定位问题耗时是在 TP 手势的口袋模式读取 psensor 数据的阶段。这里上层在读取 psensor 数据的时候，延时了 500ms 等待 psensor 数据稳定，导致手势唤醒慢。  
同步有做试验将这 500ms 延时去除，重新计算时间为 200ms 左右，唤醒时间正常，确认唤醒慢就是 500ms 延时导致。
