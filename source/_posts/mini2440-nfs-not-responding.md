---
title: '[Mini2440] nfs 无响应问题解决'
date: 2018-07-08 08:07:29
tags:
  - mini2440
  - nfs
categories: Mini2440
---

### 操作环境

win7 电脑上装有 vmware 虚拟机，虚拟机中运行 ubuntu 16.04LTS 系统，ubuntu 启动 nfs 服务供开发板挂载网络文件系统，以便和 ubuntu 进行文件传输。  
开发板使用的是 mini2440 开发板，刷的是韦东山老师的 u-boot-1.1.6 + linux-2.6.22.6 系统。  
网卡驱动用的是韦东山老师光盘中提供的适用于 mini2440 的网卡驱动。

### 遇到问题

由于搬家，重新买了个小米路由器，使用 mini2440 挂载虚拟机启动的 nfs 网络文件系统出现问题。
问题现象是：可以成功挂载，但是拷贝文件的时候会出错，操作过程如下：

先设置下设置开发板 ip 地址

    ifconfig eth0 192.168.31.100

ping 一下服务器，能 ping 通，说明局域网配置没有问题

    # ping 192.168.31.66
    PING 192.168.31.66 (192.168.31.66): 56 data bytes
    64 bytes from 192.168.31.66: seq=0 ttl=64 time=0.963 ms
    64 bytes from 192.168.31.66: seq=1 ttl=64 time=0.707 ms

挂载 nfs 文件系统试试，能成功挂载

    # mount -t nfs 192.168.31.66:/home/user/board /mnt -o nolock
    # ls -l /mnt/
    -rwxrwxr-x    1 1000     1000         5543 Jul  1  2018 app
    -rw-rw-r--    1 1000     1000        53809 Jul  1  2018 early_module.ko

尝试拷贝文件，小文件可以正常拷贝，但是稍微大一点的文件就不行，报错如下：

    # cp /mnt/app  /
    # cp /mnt/early_module.ko  /
    nfs: server 192.168.31.66 not responding, still trying

此问题 100% 必现，影响工作学习，需要赶紧解决。

### 怀疑对象

主要怀疑这几块的问题：

1. vmware 启动的 nfs 服务出现问题
2. 小米路由器的问题
3. 开发板网太卡驱动的问题

### 一一验证

验证过程如下：

1. 重启 vmware 中的 nfs 服务，验证3次，问题仍存在  
   ubuntu 重启 nfs 服务命令如下：

       user@vmware:~$ sudo /etc/init.d/nfs-kernel-server restart
       [ ok ] Restarting nfs-kernel-server (via systemctl): nfs-kernel-server.service.sudo /etc/init.d/nfs-kernel-server restart

2. 使用另外一台装有 ubuntu 的笔记本挂载，可以成功挂载并访问，基本排除 nfs 服务问题

       user@pavilion:~$ ping 192.168.31.66
       PING 192.168.31.66 (192.168.31.66) 56(84) bytes of data.
       64 bytes from 192.168.31.66: icmp_seq=1 ttl=64 time=1.10 ms
       64 bytes from 192.168.31.66: icmp_seq=2 ttl=64 time=0.826 ms
       user@pavilion:~$ sudo mount -t nfs 192.168.31.66:/home/user/board /mnt/ -o nolock
       user@pavilion:~$ ls -l /mnt/
       -rwxrwxr-x 1 user user  5543 7月   1 21:18 app
       -rw-rw-r-- 1 user user 53809 7月   1 21:18 early_module.ko
       user@pavilion:~$ sudo cp /mnt/app ./
       user@pavilion:~$ sudo cp /mnt/early_module.ko ./
       user@pavilion:~$ ls -l 
       -rwxr-xr-x 1 root root  5543 7月   6 22:13 app
       -rw-rw-r-- 1 user user 53809 7月   6 22:13 early_module.ko

    可以正常挂载并访问，这就奇怪了，ubuntu 主机可以，但是开发板不行，这两者的差异在哪？  
    难道是网卡驱动的问题，文件一大就问题就体现出来了？

3. 重新烧录友善之臂官方提供的内核，验证3次，问题仍存在，排除网卡问题  
   参考我[之前总结的博客](http://www.maziot.com/2018/01/12/mini2440-build-the-dev-env/)重新烧录内核后，验证

       [root@FriendlyARM /]# ifconfig eth0 192.168.31.100
       [root@FriendlyARM /]# ping 192.168.31.66
       PING 192.168.31.66 (192.168.31.66): 56 data bytes
       64 bytes from 192.168.31.66: seq=0 ttl=64 time=4.495 ms
       64 bytes from 192.168.31.66: seq=1 ttl=64 time=0.780 ms
       [root@FriendlyARM /]# mount -t nfs 192.168.31.66:/home/user/board /mnt -o nolock
       [root@FriendlyARM /]# ls -l /mnt/
       -rwxrwxr-x    1 1000     1000         5543 Jul  1  2018 app
       -rw-rw-r--    1 1000     1000        53809 Jul  1  2018 early_module.ko
       [root@FriendlyARM /]# cp mnt/app ./
       [root@FriendlyARM /]# cp mnt/early_module.ko ./
       nfs: server 192.168.31.66 not responding, still trying

    使用友善之臂官方提供的网卡也有同样的问题，那就说明不是网卡这一块的问题。  
    难道是小米路由器不认识 mini2440 开发板，对它做了点限制，我不知道到？  
    赶紧闲鱼二手买了个路由器，验证下看看。

4. 换 TP-LINK 路由器验证，问题仍然存在，排除路由器问题

       [root@FriendlyARM /]# ifconfig eth0 192.168.1.230
       [root@FriendlyARM /]# ping 192.168.1.101
       PING 192.168.1.101 (192.168.1.101): 56 data bytes
       64 bytes from 192.168.1.101: seq=0 ttl=64 time=4.495 ms
       64 bytes from 192.168.1.101: seq=1 ttl=64 time=0.780 ms
       [root@FriendlyARM /]# mount -t nfs 192.168.1.101:/home/user/board /mnt -o nolock
       [root@FriendlyARM /]# ls -l /mnt/
       -rwxrwxr-x    1 1000     1000         5543 Jul  1  2018 app
       -rw-rw-r--    1 1000     1000        53809 Jul  1  2018 early_module.ko
       [root@FriendlyARM /]# cp mnt/app ./
       [root@FriendlyARM /]# cp mnt/early_module.ko ./
       nfs: server 192.168.1.101 not responding, still trying

    换了路由器，也还是一样，说明不是路由器的问题。  
    那究竟是什么问题？纠结。

无奈，求助百度，验证多种方案，最后得到的解决方案是: 在 mount 时追加 `-otcp` 选项。  
即使用下面命令在开发板上挂载可以有效的解决此问题。

    mount -t nfs 192.168.31.66:/home/user/board /mnt -o nolock -otcp

### 参考博客

<https://blog.csdn.net/g1036583997/article/details/44492759>
