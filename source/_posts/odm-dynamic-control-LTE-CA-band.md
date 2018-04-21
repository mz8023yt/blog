---
title: '[ODM] 动态开启关闭 LTE CA band'
date: 2018-02-26 19:01:47
tags:
  - modem
  - AT
  - mtk
categories: ODM
---

### 问题描述

客户在定义项目的时候，之前单软多硬仅仅做了三个 modem 模块。  
但是后来客户又要求要求在 TW 的基础上删除 LTE band，再克制化一个 modem。  

### 解决思路

**(1)** 再克制化一个 modem，增加一个单软多硬的 modem

参考 MTK online [FAQ19715] How to disable/enable CA for LTE-A。  
可以修改 modem 端直接关闭 CA。但是需要多一个 modem 版本配置，需要通过单软多硬实现。  
但是倘若使用单软多硬再多加一个配置的话，需要较大的管控力度。

**(2)** 使用 AT command 在 AP 端动态的开关 CA band

可以通过 AT+ECASW 来开关 CA 功能，该 AT command 设定后会写入 NVRAM，但是需要重启或是 EFUN=0 -> 1 之后使用的参数会生效。  
命令格式如下：

```bash
AT+ECASW – LTE CA switch (Proprietary Command)
Description     To turn on/off LTE CA.
Command Format
Command         Response
+ECASW=<mode>
+ECASW?         +ECASW: <mode>
+ECASW=?        +ECASW: (list of supported <mode>s)
Field
<mode>: integer
                0 – turn off LTE CA
                1 – turn on LTE CA
```

### 思路验证

联想到客户售后指令就是通过 AT 和 NV 交互获取 NV 的信息。因此想增加一个 PhoneInfoTest 的子命令实现 CA 的关闭和开启。  
不料却发现，根本不需要这么干，PhoneInfoTest 里面的 command 24 就是 AT command 的接口，使用下面格式可以直接执行 AT command，但是要求要是 eng 的版本才有权限。

```bash
C:\Users\wangbing>adb shell /data/data/PhoneInfoTest 88 0 "AT command"
```

客户售后指令 PhoneInfoTest 里面支持 AT command，使用 eng 版本的手机，进入 adb 执行以下命令关闭 LTE CA。

```bash
C:\Users\wangbing>adb shell
android:/ # /data/data/PhoneInfoTest 88 0 AT+ECASW=0
android:/ # /data/data/PhoneInfoTest 88 0 AT+ECASW=0 debug
<<adb>> command: 24,adbcmd:880,atoi(cmd1)=88,atoi(cmd1)=0
<<adb>> ATsend 387
<<adb>> AT: AT+ECASW=0
<<adb>> connectTarget 156,connc_socket=1
<<adb>> single modem mode and try to connect socket 1.
<<adb>> sendDataToRild(AT+ECASW=0) 121
<<adb>> rx...
<<adb>> rx... got len: 6 rx:
OK
<<adb>>
OK
```

接下来就是给射频验证了，看看使用 AT command 关闭 CA 是否生效。  
射频给出答复，关闭 CA 生效了。
