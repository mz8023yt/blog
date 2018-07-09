---
title: '[ODM] 新增 PhoneInfoTest 命令'
date: 2018-04-28 10:29:18
tags:
  - qcom
categories: Experience
---

### 问题来源

客户新增 90PN 料号文件的读写需求，需要实现两个部分：

1. 工厂产线生产时写入料号信息(已实现)。
2. 售后网点售后时写入料号信息。

### 背景知识

客户售后网点使用 PhoneInfoTest 应用程序读写手机信息，目前 PhoneInfoTest 支持的命令如下：

```
## 整机 SSN eg. D8OKCT101847
adb shell /data/data/PhoneInfoTest 0 1 [SSN]
adb shell /data/data/PhoneInfoTest 0 0

## IMEI eg. 358463051106605 358463051106613
adb shell /data/data/PhoneInfoTest 1 1 [IMEI1]
adb shell /data/data/PhoneInfoTest 1 2 [IMEI2]
adb shell /data/data/PhoneInfoTest 1 0 [IMEI1]
adb shell getprop persist.radio.device.imei
adb shell getprop persist.radio.device.imei2

## BT eg. 5CFF356DB1D9
adb shell /data/data/PhoneInfoTest 2 1 [BT Mac]
adb shell /data/data/PhoneInfoTest 2 0
adb shell getprop ro.btmac

## WiFi eg. 5CFF358DB1D9
adb shell /data/data/PhoneInfoTest 3 1 [Wifi Mac]
adb shell /data/data/PhoneInfoTest 3 0
adb shell getprop ro.wifimac

## 主板 ISN eg. E234C671106011
adb shell /data/data/PhoneInfoTest 4 1 [ISN]
adb shell /data/data/PhoneInfoTest 4 0
adb shell getprop ro.isn

## Country Code eg. CN、WW
adb shell /data/data/PhoneInfoTest 6 1 [Country Code]
adb shell /data/data/PhoneInfoTest 6 0
adb shell getprop ro.config.versatility

## Color ID eg. 1A
adb shell /data/data/PhoneInfoTest 7 1 [Color ID Code]
adb shell /data/data/PhoneInfoTest 7 0
adb shell getprop ro.config.idcode

## Customer ID eg. XIAOMI
adb shell /data/data/PhoneInfoTest 8 1 [Customer ID Code]
adb shell /data/data/PhoneInfoTest 8 0
adb shell getprop ro.config.CID

## Packing Code eg. WW
adb shell /data/data/PhoneInfoTest 9 1 [Packing Code]
adb shell /data/data/PhoneInfoTest 9 0
adb shell getprop ro.config.revenuecountry

## MEID eg. 99000841001456
adb shell /data/data/PhoneInfoTest 11 1 [MEID]
adb shell /data/data/PhoneInfoTest 11 0
adb shell getprop persist.radio.device.meid

## SIMCODE
adb shell /data/data/PhoneInfoTest 12 1 [SIMCODE]
adb shell /data/data/PhoneInfoTest 12 0

## Model eg. ASUS_X00GD
adb shell getprop ro.product.model

## Device eg. ASUS_X00GD_1
adb shell getprop ro.product.device

## Image Version eg. CSC_ZC521TL_14.00.1701.10_CN_20170104
adb shell getprop ro.build.version.incremental
adb shell getprop ro.build.csc.version
adb shell getprop ro.build.display.id
```
 
备注：PhoneInfoTest 程序通过将 `argv[1]` 和 `argv[2]` 组合起来的 `argv[1]*10 + argv[2]` 的值，去查表，找到真正的命令，再去执行命令对应的 case 分支，从而实现不同命令做不同的操作的需求，具体请分析代码。

### 功能实现

新增 PhoneInfoTest 13 指令实现读写 90PN 文件。

```
diff --git a/custom_files/vendor/qcom/proprietary/<Project>/diag/PhoneInfoTest/PhoneInfoTest.cpp b/custom_files/vendor/qcom/proprietary/<Project>/diag/PhoneInfoTest/PhoneInfoTest.cpp
index 83328a2..5fd87d6 100755
--- a/custom_files/vendor/qcom/proprietary/<Project>/diag/PhoneInfoTest/PhoneInfoTest.cpp
+++ b/custom_files/vendor/qcom/proprietary/<Project>/diag/PhoneInfoTest/PhoneInfoTest.cpp
@@ -355,6 +355,9 @@ void diag_deinit()
 
 #define PROINFO_BUFFER_PATH_BACKUP "/dev/block/bootdevice/by-name/proinfo"
 
+// mz8023yt@163.com 20180424 begin >>> [1/7] add write 90PN interface
+#define FACTORY_90PN_PATH "/factory/90PN"
+// mz8023yt@163.com 20180424 end   <<< [1/7] add write 90PN interface
 
 int proinfo_read(unsigned char *buffer, int len)
 {
@@ -433,7 +436,12 @@ int main(int argc, char** argv)
 
        struct AP_Product_Info *p_AP_Product_Info;
        struct AP_Product_Info *p_AP_Product_backup_Info;

+// mz8023yt@163.com 20180424 begin >>> [2/7] add write 90PN interface
+       unsigned char pn_buf[1024];
+       int pn_fd = 0;
+       int pn_ret = -1;
+// mz8023yt@163.com 20180424 end   <<< [2/7] add write 90PN interface

        unsigned char imei[9] = {0x08,0x0A};
        int i = 0;
@@ -685,9 +693,41 @@ int main(int argc, char** argv)
                                        diag_deinit();
                                }       
                                break;

+// mz8023yt@163.com 20180424 begin >>> [3/7] add write 90PN interface
+               case WRITE_90PN:
+                       pn_fd = open(FACTORY_90PN_PATH, O_RDWR | O_SYNC | O_CREAT, 0660);
+                       if (pn_fd < 0) {
+                               printf("can't open file! error = %d\n", pn_fd);
+                               return -1;
+                       }
+                       memset(&pn_buf, 0, sizeof(pn_buf));
+                       memcpy(&pn_buf, argv[3], strlen(argv[3]));
+                       pn_ret = write(pn_fd, pn_buf, 1024);
+                       fsync(pn_fd);
+                       if(pn_ret == strlen(1024)) {
+                               printf("0: succeed\n");
+                       } else {
+                               printf("1: failure\n");
+                       }
+                       close(pn_fd);
+                       break;
+
+               case READ_90PN:
+                       pn_fd = open(FACTORY_90PN_PATH, O_RDWR | O_SYNC | O_CREAT, 0660);
+                       if (pn_fd < 0) {
+                               printf("can't open file! error = %d\n", pn_fd);
+                               return -1;
+                       }
+                       memset(&pn_buf, 0, sizeof(pn_buf));
+                       read(pn_fd, pn_buf, 1024);
+                       close(pn_fd);
+                       printf("%s\n", pn_buf);
+                       break;
+// mz8023yt@163.com 20180424 end   <<< [3/7] add write 90PN interface

                 default:
                        printf("cmd %d don't support\n", cmd);
                        break;
        }

diff --git a/custom_files/vendor/qcom/proprietary/<Project>/diag/PhoneInfoTest/PhoneInfoTest.h b/custom_files/vendor/qcom/proprietary/<Project>/diag/PhoneInfoTest/PhoneInfoTest.h
index 1abe502..d5f5c48 100755
--- a/custom_files/vendor/qcom/proprietary/<Project>/diag/PhoneInfoTest/PhoneInfoTest.h
+++ b/custom_files/vendor/qcom/proprietary/<Project>/diag/PhoneInfoTest/PhoneInfoTest.h
@@ -52,6 +52,11 @@ typedef enum {
        WRITE_CUSTOMID,
        WRITE_PACKINGCODE,
        WRITE_SIMCODE,
+// mz8023yt@163.com 20180424 begin >>> [4/7] add write 90PN interface
+       WRITE_90PN,
+// mz8023yt@163.com 20180424 end   <<< [4/7] add write 90PN interface
        WRITE_END,
        READ_SSN ,
        READ_MEID,      
@@ -65,6 +70,11 @@ typedef enum {
        READ_CUSTOMID,
        READ_PACKINGCODE,
        READ_SIMCODE,
+// mz8023yt@163.com 20180424 begin >>> [5/7] add write 90PN interface
+       READ_90PN,
+// mz8023yt@163.com 20180424 end   <<< [5/7] add write 90PN interface
        READ_END,
        
        AT_CMD, 
@@ -123,6 +133,11 @@ static unsigned int adbcmd_code[40][2]={
        WRITE_CUSTOMID,81,
        WRITE_PACKINGCODE,91,
        WRITE_SIMCODE,121,
+// mz8023yt@163.com 20180424 begin >>> [6/7] add write 90PN interface
+       WRITE_90PN, 131,
+// mz8023yt@163.com 20180424 end   <<< [6/7] add write 90PN interface
        READ_SSN ,00,
        READ_MEID,110,  
        READ_IMEI1,10,
@@ -134,6 +149,11 @@ static unsigned int adbcmd_code[40][2]={
        READ_CUSTOMID,80,
        READ_PACKINGCODE,90,
        READ_SIMCODE,120,
+// mz8023yt@163.com 20180424 begin >>> [7/7] add write 90PN interface
+       READ_90PN, 130,
+// mz8023yt@163.com 20180424 end   <<< [7/7] add write 90PN interface
        AT_CMD,880,
        CUSTOM_CMD, 770,
        ADB_CMD_END,990
```

### 编译验证

单编模块，编译更快，编译完成之后，push 到手机中执行看效果，最后将 90PN 文件 push 出来，确认是否真的有写到。

```
## 配置 Android 工程
[wangbing@ubuntu: ~/src] source build/envsetup.h
[wangbing@ubuntu: ~/src] lunch <Project>

## 单编
[wangbing@ubuntu: ~/src]$ rm -rf out/target/product/<Project>/vendor/bin/PhoneInfoTest ~/
[wangbing@ubuntu: ~/src]$ mmm vendor/qcom/proprietary/diag/PhoneInfoTest/

## 导入手机
[wangbing@ubuntu: ~/src]$ cp out/target/product/<Project>/vendor/bin/PhoneInfoTest ~/
[wangbing@ubuntu: ~/src]$ adb push ~/PhoneInfoTest /vendor/bin

## 验证
[wangbing@ubuntu: ~/src]$ adb shell /data/data/PhoneInfoTest 13 1 90AZ01K1-S00050
0: succeed
[wangbing@ubuntu: ~/src]$ adb shell /data/data/PhoneInfoTest 13 0
90AZ01K1-S00050

## 导出到桌面，使用文本工具查看对应16进制字符再次确认
C:\Users\wangbing>adb pull /factory/90PN Desktop
```

### 产线工具

前面有提到工厂产线生产时写入料号信息已经实现，这里也简单概述下，产线写 90PN 的相关实现逻辑。产线流水线有一个专门的写号站位，会将手机的很多信息统一写到手机的 NVRAM 中。这很多的信息中自然也包括 90PN 信息。而产线的工具实际上是调用高通开放的 API 接口实现和手机通信的，这个 API 接口其实就是 QXDM 工具中发送 cmd 是同样的东西。因此在验证阶段，我们用 QXDM 发送对应的 cmd 就可以实现产线工具同样的写 90PN 的效果。

写入 90PN 对应的 QXDM 指令是：`Send_data 0x4b 0xc9 0xcc 0xff 12 <data>`，其中 data 有格式要求，data 的格式是，"一次发一个字符的 ASCII 码的十进制数，用空格隔开"。

倘若我们要往 90PN 中写入 `Hello` 字符，我们应该这么干：
1. 查一下 ASCII 码表，看看 `Hello` 中的 5 个字符的十进制格式的 ASCII 码是多少？
   查了下，分别是：72 101 108 108 111
2. 接下里使用 QXDM 发送 `Send_data 0x4b 0xc9 0xcc 0xff 12 72 101 108 108 111`。
3. 使用 `adb shell /data/data/PhoneInfoTest 13 0` 查看是否写入成功。

### ASCII 码表

[百度百科 ASCII 码表](https://baike.baidu.com/item/ASCII/309296?fr=aladdin)
