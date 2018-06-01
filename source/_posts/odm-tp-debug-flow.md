---
title: '[ODM] TP 相关 debug 流程'
date: 2018-04-11 14:00:43
tags:
  - touch
  - qcom
categories: ODM
---

## 一 常见问题

### 1.1 触摸失效

1. 确认是否是 TP 的问题
   使用 otg 接鼠标看看能否正常点击？otg 鼠标可以的话，用 otg 鼠标打开指针位置，再触摸，看看有没有指针。这步操作可以确认是 TP 的问题，还是 UI 响应的问题。

2. 确认驱动有没有注册上
   在没有重启手机(不重启手机是怕重启后不良现象丢失)保留着不良现象的前提下，查看 TP 相关的 proc、sys 节点是否创建成功？有节点说明驱动正常注册上了，没有则驱动没有注册上。如果不良现象重启还能复现的话(重启动作最后才能做，先做好其他的验证)，可以抓取开机 log 确认 i2c 驱动是否注册上了，这样最为准确。

3. 确认有没有中断
   如果驱动注册上了，还是失效的话，需要查看 /proc/interrupts 节点，看看有没有检测到 TP 的中断。
   ```
   C:\Users\wangbing>adb shell cat /proc/interrupts
              CPU0       CPU1       CPU2       CPU3
     3:     103643      54431      40430      34763       GIC  20  arch_timer
     5:      76002      33111      25707      15604       GIC  39  arch_mem_timer
     6:         37          0          0          0       GIC 240
    72:       3496          0          0          0   msmgpio  65  hxcommon
   ```
   这里的 "hxcommon" 是中断名称，3496 是中断计数，cat 查看之后，触摸几下，再次执行上面的命令。看看中断计数是否有增加，有增加说明有中断产生。

4. 确认有没有报点
   如果中断有捕捉到，但还是失效的话，再看看有没有报点信息。部分 TP 支持动态的开关 debug log，开启 debug log 示例如下：
   ```
   C:\Users\wangbing>adb shell
   android:/ # cd proc/android_touch/
   android:/proc/android_touch # echo 3 > debug_level
   android:/proc/android_touch # cat debug_level
   3
   ```
   开启 debug log 再看看有没有报点 log 打印。
   ```
   C:\Users\wangbing>adb shell dmesg -c > 0 && adb shell dmesg -w | findstr "HX83102"
   [53300.503826] [HX83102] Finger 1=> X:347, Y:734 W:82, Z:82, F:1, N:0
   ```

5. 查看 IC 工作状态  
   部分 TP 预留有查看寄存器信息的 proc 节点，具体的使用方法根据各家的 proc 节点实现而定，详细的使用方法需要仔细阅读节点读写函数。
   ```
   C:\Users\wangbing>adb shell
   android:/ # cd proc/android_touch/
   android:/proc/android_touch # echo "r:x8F" > register
   android:/proc/android_touch # cat register
   command:  00,00,00,8F
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   
   android:/proc/android_touch # echo "r:x8F" > register
   android:/proc/android_touch # cat register
   command:  00,00,00,8F
   0x20 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
   ```
   这里看 cat 8F 寄存器得到的第一次字节，在亮屏报点模式下，8F 寄存器的值是 0x00，在熄屏手势模式下，8F 寄存器的值是 0x20，和 spec 描述相符合，说明工作状态正常。

6. 确认 IC 是否异常
   有些时候，可能是某一次 TP IC 在唤醒屏幕的时候 reset 过程出现问题，或者是莫名奇妙的出现问题，这谁也说不准。可以按 power key 多亮灭屏几次(会重新给 TP 上下电，做 reset IC 的动作)，看触摸是否可以恢复正常。

7. 确认是否是单体不良
   做个交叉实现验证下：找一台正常手机，接不良机器的屏，看是否还是不良。将正常机器的屏，接上不良机主板，看不良是否跟着屏走。

### 1.2 熄屏手势失效

1. 确认上层手势开关设置是否生效
   我们知道手势的开关控制是驱动在 proc 目录下创建对应的控制节点，上层通过写这些节点将手势的开关信息传递到驱动中。手势失效的情况，应该先 cat 这些 proc 节点，看看手势的开关控制信息是否真正的传递到了驱动中。一般驱动中都会在读写手势的 fops 中加点 log，最好是通过 log 确认，这样最为准确。

2. 确认 TP 睡眠时有没有成功进入手势模式
   一般供应商在 tp suspend 函数中进入手势模式的位置都有对应的 debug 的 log，这里直接通过 log 确认就好。没有的话一定要自己加一句。  
   部分 TP 支持查看状态寄存器，这种确认方式最为有效。

3. 确认是否有手势键值上报
   如果确实开了手势，并且 TP 已经进入了手势模式，看看有没有手势键值上报，一般上报手势键值的位置都要加好对应的 log，通过 log 确认下有没有手势键值上报，如果调试的时候忘记了加 log，可以使用 getevent 看看 TP 对应的 input 设备有没有上报 input 事件，这样也可以确认。如果有键值上报，说明上层没有响应，上层的问题，由于有做口袋模式，那顺便看看距离传感器功能是不是正常的。没有上报，还是驱动这块的问题，再接着分析。

4. 确认熄屏手势是否产生中断
   如果进入了手势模式，但没有键值上报，那就要看看 /proc/interrupts 节点，看看有没有检测到 TP 的中断，没有检测到，估计就是 TP IC 单体不良，交给供应商分析就好。如果检测到中断，还是没有上报，估计就是驱动中哪里出了问题，要打开 TP debug log 开关，好好分析 log 了，同样建议约供应商一起看 log 分析。

## 二 常用 TP debug 手段

### 2.1 adb shell input

input 命令是用来向设备发送模拟操作的命令，不同的 Android 版本支持的选项有所不同，Android 8.0 系统的 input 说明如下：
```
C:\Users\wangbing>adb shell input
Usage: input [<source>] <command> [<arg>...]

The sources are:
      dpad
      keyboard
      mouse
      touchpad
      gamepad
      touchnavigation
      joystick
      touchscreen
      stylus
      trackball

The commands and default sources are:
      text <string> (Default: touchscreen)
      keyevent [--longpress] <key code number or name> ... (Default: keyboard)
      tap <x> <y> (Default: touchscreen)
      swipe <x1> <y1> <x2> <y2> [duration(ms)] (Default: touchscreen)
      draganddrop <x1> <y1> <x2> <y2> [duration(ms)] (Default: touchscreen)
      press (Default: trackball)
      roll <dx> <dy> (Default: trackball)
```

使用 input 命令，可以实现唤醒屏幕，上滑解锁，点击屏幕，输入字符等操作。

```
// 唤醒屏幕
C:\Users\wangbing>adb shell input keyevent "KEYCODE_POWER"

// 上滑，从 (360 900) 滑动到 (360 300)
C:\Users\wangbing>adb shell input swipe 360 900 360 300

// 左滑
C:\Users\wangbing>adb shell input swipe 200 720 600 720

// 触摸
C:\Users\wangbing>adb shell input tap 340 360

// 向获得焦点的 EditText 控件输入内容
C:\Users\wangbing>adb shell input text "hello,world"
```

将上面的命令组合起来后依次执行对应的效果如下：

![效果图](https://raw.githubusercontent.com/mz8023yt/blog.material/master/odm/gif/odm-tp-input-cmd.gif)

        C:\Users\wangbing>adb shell input keyevent "KEYCODE_POWER" && adb shell input swipe 360 900 360 300 && adb shell input swipe 200 720 600 720 && adb shell input tap 340 360 && adb shell input text "hello,world"

### 2.2 adb shell getevent

```
C:\Users\wangbing>adb shell getevent -help
Usage: getevent [-t] [-n] [-s switchmask] [-S] [-v [mask]] [-d] [-p] [-i] [-l] [-q] [-c count] [-r] [device]
    -t: show time stamps
    -n: don't print newlines
    -s: print switch states for given bits
    -S: print all switch states
    -v: verbosity mask (errs=1, dev=2, name=4, info=8, vers=16, pos. events=32, props=64)
    -d: show HID descriptor, if available
    -p: show possible events (errs, dev, name, pos. events)
    -i: show all device info and possible events
    -l: label event types and names in plain text
    -q: quiet (clear verbosity mask)
    -c: print given number of events then exit
    -r: print rate events are received
```

### 2.3 adb shell sendevent

```
C:\Users\wangbing>adb shell sendevent --help
usage: sendevent DEVICE TYPE CODE VALUE

Sends a Linux input event.
```
