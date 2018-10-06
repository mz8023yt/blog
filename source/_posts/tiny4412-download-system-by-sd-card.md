---
title: '[Tiny4412] 使用 SD 卡给 Tiny4412 烧写系统'
date: 2018-10-06 14:16:58
tags:
  - tiny4412
  - FrindlyARM
categories: Tiny4412
---

### 需要准备的东西

这里我列出我通过 SD 卡给 Tiny4412 烧写系统涉及到的资源

- 电脑一台，电脑安装 Windows 系统，并通过 vmware 安装有 Ubuntu 虚拟机
- Tiny4412 开发板一台，包括显示屏和电源线
- 大于 4G 的 SD 卡一张，包括小卡转大卡的 SD adapter
- 网线一根，路由器一个
- 友善之臂光盘两张，没有实际光盘可以在 百度云 中直接下载光盘镜像

### 刷机步骤

1. 对 SD 卡分区，并烧写引导程序到 SD 卡中  
   分区工具位置：`光盘A盘\tools\SD-Flasher-1328\SD-Flasher.exe`  
   注意：工具软件需要使用管理员身份运行  

   ![操作步骤](https://raw.githubusercontent.com/mz8023yt/blog.material/master/tiny4412/png/tiny4412-download-system-by-sd-card-02.png)

2. 将光盘 B 盘中的 image 目录拷贝到 SD 卡中  
   这个 image 目录中就是系统镜像文件，进入 image 目录可以看到有三个子目录，分别是 Android、Linux、Ubuntu 相关的系统镜像文件。除了这三个文件夹，还有一个通用的引导程序和一个配置文件。

3. 修改配置文件  
   配置文件前几行为：
   
       CheckOneButton=No
       Action = Install
       OS = Android

       LowFormat = Yes
       VerifyNandWrite = No

       LCD-Mode = No
       CheckCRC32=No

       StatusType = Beeper | LED

   这里我们可通过修改 OS 的值为 Android、Linux、Ubuntu 来选择要烧写哪个系统

4. 使用 SD 卡烧写  
   将启动开关拨到 SD 卡启动，插入制作好的 SD 卡，上电，可以在显示屏上看到进度条，同时蜂鸣器会有提示音。

5. 使用 EMMC 启动系统
   进度条完成后，此时系统已经烧写成功了，掉电，将启动开关拨到 EMMC 启动，上电，可以进入系统。

### 通过 nfs 网络文件系统在 Ubuntu 和 Tiny4412 之间传输文件

先查看下 ubuntu 的 ip 地址

    user@vmware:~$ ifconfig
    ens33     Link encap:Ethernet  HWaddr 00:0c:29:d6:8f:01  
              inet addr:192.168.31.178  Bcast:192.168.31.255  Mask:255.255.255.0
              inet6 addr: fe80::c7e0:ff91:111f:c9cf/64 Scope:Link
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:840535 errors:0 dropped:0 overruns:0 frame:0
              TX packets:1385918 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000 
              RX bytes:82241401 (82.2 MB)  TX bytes:2467078326 (2.4 GB)

    lo        Link encap:Local Loopback  
              inet addr:127.0.0.1  Mask:255.0.0.0
              inet6 addr: ::1/128 Scope:Host
              UP LOOPBACK RUNNING  MTU:65536  Metric:1
              RX packets:5138 errors:0 dropped:0 overruns:0 frame:0
              TX packets:5138 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000 
              RX bytes:416944 (416.9 KB)  TX bytes:416944 (416.9 KB)

再查看下开发板的 ip 地址，这里我的板子自动获取了 ip，如果没有自动获取的话，需要使用 `ifconfig eth0 <ip>` 手动设置板子的 ip 地址

    [root@FriendlyARM /]# ifconfig
    eth0      Link encap:Ethernet  HWaddr 1C:6F:65:34:51:7E  
              inet addr:192.168.31.153  Bcast:192.168.31.255  Mask:255.255.255.0
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:5363 errors:0 dropped:2 overruns:0 frame:0
              TX packets:43 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000 
              RX bytes:250704 (244.8 KiB)  TX bytes:3103 (3.0 KiB)

    lo        Link encap:Local Loopback  
              inet addr:127.0.0.1  Mask:255.0.0.0
              UP LOOPBACK RUNNING  MTU:16436  Metric:1
              RX packets:0 errors:0 dropped:0 overruns:0 frame:0
              TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0 
              RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

在板子上 ping 一下 Ubuntu，保证板子和 Ubuntu 在同一个局域网中，相互之前可以 ping 通
            
    [root@FriendlyARM /]# ping 192.168.31.178
    PING 192.168.31.178 (192.168.31.178): 56 data bytes
    64 bytes from 192.168.31.178: seq=0 ttl=64 time=1.064 ms
    64 bytes from 192.168.31.178: seq=1 ttl=64 time=0.980 ms
    64 bytes from 192.168.31.178: seq=2 ttl=64 time=0.902 ms
    ^C
    --- 192.168.31.178 ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max = 0.902/0.982/1.064 ms

在 Ubuntu 中配置好要和板子共享的 nfs 共享目录，这里我设置的是 `/home/user/board` 目录作为共享目录。具体的设置步骤我就不细说了。  
板子上挂载 nfs 网络文件系统，实现和 Ubuntu 之间通信，这样，在 ubuntu 上编译的驱动和应用程序就可以直接传到板子上运行了。

    [root@FriendlyARM /]# mount -t nfs 192.168.31.178:/home/user/board /mnt -o nolock
    [root@FriendlyARM /]# cd /mnt/
    [root@FriendlyARM /mnt]# ls
    sixth_app     sixth_drv.ko


