---
title: '[ODM] TP 相关 debug 流程'
date: 2018-04-11 14:00:43
tags:
  - touch
  - qcom
categories: ODM
---

## 一 问题分类

### 1.1 TP 失效分析步骤

#### 1.1.1 确认是否是 TP 的问题

使用 otg 接鼠标看看能否正常点击？otg 鼠标可以的话，用 otg 鼠标打开指针位置，再触摸，看看有没有指针。这步操作可以确认是 TP 的问题，还是 UI 响应的问题。

#### 1.1.2 确认驱动有没有注册上

在没有重启手机(不重启手机是怕重启后不良现象丢失)保留着不良现象的前提下，查看 TP 相关的 proc、sys 节点是否创建成功？有节点说明驱动正常注册上了，没有则驱动没有注册上。  
如果不良现象重启还能复现的话(重启动作最后才能做，先做好其他的验证)，可以抓取开机 log 确认 i2c 驱动是否注册上了，这样最为准确。

#### 1.1.3 确认有没有中断

如果驱动注册上了，还是失效的话，需要查看 /proc/interrupts 节点，看看有没有检测到 TP 的中断。

```
C:\Users\wangbing>adb shell cat /proc/interrupts | findstr "hxcommon"
 72:       3496          0          0          0   msmgpio  65  hxcommon
```

这里的 "hxcommon" 是中断名称，3496 是中断计数，cat 查看之后，触摸几下，再次执行上面的命令。看看中断计数是否有增加，有增加说明有中断产生。

#### 1.1.4 确认有没有报点

如果中断有捕捉到，但还是失效的话，再看看有没有报点信息。  
部分 TP 支持动态的开关 debug log，开启 debug log 示例如下：

```
C:\Users\wangbing>adb shell
ASUS_X00P_1:/ # cd proc/android_touch/
ASUS_X00P_1:/proc/android_touch # echo 3 > debug_level
ASUS_X00P_1:/proc/android_touch # cat debug_level
3
```

开启 debug log 再看看有没有报点 log 打印。

```
C:\Users\wangbing>adb shell dmesg -c > 0 && adb shell dmesg -w | findstr "HX83102"
[53300.503826] [HX83102] Finger 1=> X:347, Y:734 W:82, Z:82, F:1, N:0
```

#### 1.1.5 确认 IC 是否异常

有些时候，可能是某一次 TP IC 在唤醒屏幕的时候 reset 过程出现问题，或者是莫名奇妙的出现问题，这谁也说不准。  
可以按 power key 多亮灭屏几次(会重新给 TP 上下电，做 reset IC 的动作)，看触摸是否可以恢复正常。

#### 1.1.6 确认是否是单体不良

做个交叉实现验证下：
1. 找一台正常手机，接不良机器的屏，看是否还是不良。
2. 将正常机器的屏，接上不良机主板，看不良是否跟着屏走。


