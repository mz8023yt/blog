---
title: '[Ubuntu] 使用 VMware 虚拟机安装 Ubuntu 系统'
date: 2018-02-25 23:48:53
tags:
  - ubuntu
  - vmware
categories: Ubuntu
---

## 一. 下载安装 VMware

### 1.1 VMware 介绍

VMware 就是我们俗称的虚拟机，通过这个软件我们可以模拟出一台或者多台 PC 机，就好像我们买了很多台电脑一样。  
我们可以在这些虚拟出来的 PC 机上安装我们的操作系统，可以安装 windows、ubuntu、fedora 等操作系统。然后在运行这个装好了操作系统的 PC 机，将这些主机给虚拟出来，就拥有很多台的 PC 主机。

### 1.2 下载 VMware 软件

百度搜索 VMware 关键字，在百度软件中心可以获取到 VMware 安装包，这里下载的是 12.5.7 版本的 VMware。

![图片1](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-01.png)

下载后将会得到一个 exe 的安装包，如下图。

![图片2](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-02.png)

### 1.3 安装 VMware 软件

双击执行刚刚下载的可执行文件，进入安装向导界面。

![图片3](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-03.png)

同意许可协议，不同意不让你安装。

![图片4](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-04.png)

选择软件的安装位置，我一般习惯性将 C 改成 D，保持子路径不变。

![图片5](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-05.png)

选择是否启动自动检查更新和反馈数据，这两个勾一般我都是不勾的，你们自己看心情。

![图片6](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-06.png)

选择是否创建相应的快捷方式。

![图片7](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-07.png)

配置好了，要开始安装了。

![图片8](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-08.png)

正在安装界面。

![图片9](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-09.png)

安装好了，可以选择直接退出，或者输入产品密钥。这里为了方便日后使用，一并激活一下这个软件。

![图片10](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-10.png)

百度搜索 "vmware workstation 12 密钥" 可以看到一大串的密钥，随便找一个用都是可以用的。

![图片11](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-11.png)

输入许可证。

![图片12](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-12.png)

至此，VMware 软件安装完成。

![图片13](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/1-13.png)




## 二. 使用 VMware 创建一台虚拟机

### 2.1 使用 VMware 创建一台虚拟机

使用 VMware 创建一台虚拟机，这个动作就好比，我们新买了一台电脑，有主板、硬盘、网卡、内存等具体的设备，是一套硬件系统，只不过这些东西都是虚拟出来的。  
买了电脑之后，我们就可以装自己想要运行的操作系统了，又拥有了一台电脑，是不是很开心。

### 2.2 新建虚拟机具体步骤

运行 VMware 软件，点击新建虚拟机。

![图片1](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/2-01.png)

随后进入虚拟机配置向导。这里选择虚拟机的配置类型，一般都是典型安装，高级要配置的东西太多了。

![图片2](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/2-02.png)

选择是否同时安装操作系统，这个时候同步安装操作系统将会采取简易安装，建议稍后在手动安装操作系统。

![图片3](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/2-03.png)

选择好这台虚拟机将会安装哪种操作系统，注意区分 32/64 位。

![图片4](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/2-04.png)

配置虚拟机名称以及虚拟机在物理磁盘上保存的路径。

![图片5](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/2-05.png)

设置虚拟机的硬盘大小。

![图片6](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/2-06.png)

点击完成，新的虚拟机将按照之前的配置创建。

![图片7](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/2-07.png)

至此，一台新的虚拟机创建成功，创建成功左侧导航栏会有相应的虚拟机选项。

![图片8](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/2-08.png)




## 三. 获取 Ubuntu 16.04 LTS

### 3.1 从官网获取最新的 Ubuntu

百度搜索 ubuntu 关键字，进入 ubuntu 官网。

![图片1](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/3-01.png)

Ubuntu 为了方便广大中国用户，推出了中文的官方网站。由于在 Ubuntu 官网下载 ubuntu 网速太慢，需要 VPN 才能流畅下载，因此，这里我们进入中文官网，方便下载。

![图片2](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/3-02.png)

进入中文官网，点击下载。

![图片3](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/3-03.png)

下载 64 位版本，或者 32 位版本都行，看你心情。

![图片4](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/3-04.png)

点击对应下载按钮将会弹出下载对话框，开始下载就好。我用的是火狐浏览器，其他浏览器可能直接开始下载了。

![图片5](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/3-05.png)

下载好之后得到 ubuntu iso 镜像包。

![图片6](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/3-06.png)




## 四. 虚拟机安装 ubuntu 操作系统

### 4.1 使用虚拟机安装 Ubuntu 16.04

运行 VMware 软件，点击编辑虚拟机设置，可以修改虚拟机的硬件配置。

![图片1](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-01.png)

在运行虚拟机之前，将下载好的 Ubuntu 镜像加载到光驱中。这样开机便可以从光驱中运行 Ubuntu 安装程序。

![图片2](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-02.png)

配置好了光驱，开启虚拟机，开始安装 Ubuntu。

![图片3](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-03.png)

开机后出现 Ubuntu logo 界面，说明已经进入了 Ubuntu 安装程序。

![图片4](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-04.png)

左侧选择好 Ubuntu 的语言，点击右方 Install Ubuntu 按钮开始安装。

![图片5](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-05.png)

这里选择的是安装时是否同步下载更新，还有就是是否安装第三方媒体库，我一般不选，因为这样会增加安装时间。复选框都不勾选，然后点击 Continue。

![图片6](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-06.png)

选择磁盘如何处理，可以擦除整个磁盘然后装 Ubuntu，也可以自己自定义分区，在指定的分区中安装 Ubuntu。由于使用虚拟机安装，就不自定义分区了，直接全擦后安装 Ubuntu，这样方便。如果是安装 Windows 和 Ubuntu 双系统的话，建议还是自己自定义分区。

![图片7](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-07.png)

这是一个二次确认的对话框，确认要全擦？然后哪些分区将被创建会列举出来。

![图片8](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-08.png)

选择位置。

![图片9](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-09.png)

选择键盘布局，我们一般都是用的美式布局的键盘，默认就可以了。

![图片10](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-10.png)

配置好用户名和主机名以及登录密码。

![图片11](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-11.png)

开始安装了，耐心等待一会。

![图片12](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-12.png)

安装完毕，提示要重启，这个时候直接关闭虚拟机，直接关机，不要重启。

![图片13](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-13.png)

进入虚拟机配置界面，将光驱的配置修改回来，不修改回来的话将无法进入 Ubuntu 系统。

![图片14](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-14.png)

至此 Ubuntu 16.04 LTS 成功安装

![图片15](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/4-15.png)




## 五. 虚拟机安装 VM Tools

### 5.1 VMware Tools 有什么用？

安装 VMware Tools 后，最简单直观的体验就是可以从物理主机直接往虚拟机里面拖文件。  
我们学习嵌入式大都是在 Windows 下阅读、编辑源代码，而编译的过程则是放在 Ubuntu 虚拟机中的。这必然涉及到文件的传输，安装 VMware Tools 之后将极大的方便虚拟机和主机之间的文件传输。

### 5.2 安装 VMware Tools

开始安装 VMware Tools

![图片1](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/5-01.png)

点击之后 Ubuntu 将会加载一个光盘，其中 tar.gz 格式的文件就是 VMware Tools 对应的安装文件。

![图片2](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/5-02.png)

Ubuntu 中使用 Ctrl + Alt + T 调起终端，依次输入以下命令开始安装 VMware Tools。

![图片3](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/5-03.png)

期间或有很多的安装提示选项，除了第一个选项问你是否要安装此程序，默认是 no，安装时需要输入 y 或 yes 以外，其他的安装选项默认即可，安装成功后截图如下。

![图片4](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/5-04.png)

### 5.3 配置虚拟机共享文件夹

在开启虚拟机之前，先按照下图配置虚拟机共享文件夹。通过配置共享文件夹，ubuntu 可以通过访问 /mnt/hgfs/Share 目录访问 windows 下的 E:\Share 目录。其中，/mnt/hgfs/Share 中的最后一级目录 Share 是下图中配置的名称，E:\Share 中的 Share 是下图中配置的主机路径。

![图片5](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/5-05.png)

例如我在 /mnt/hgfs/Share 目录下创建了一个 demo 文件。

![图片6](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/5-06.png)

然后打开 Windows 下的 E:\Share 目录下查看，发现确实有了 demo 文件。

![图片7](https://raw.githubusercontent.com/mz8023yt/blog/master/image/ubuntu/ubuntu-install-ubuntu-by-vmware/5-07.png)

