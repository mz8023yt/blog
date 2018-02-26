---
title: '[U-Boot] 如何获取最新的 U-Boot'
date: 2018-01-24 22:43:46
tags:
  - u-boot
  - bootloader
categories: U-Boot
---

## 一. U-Boot 简介

### 1.1 U-Boot 是什么？

U-Boot，全称 Universal Boot Loader，是遵循 GPL 条款的开源项目。U-Boot 是一种普遍用于嵌入式系统中的 BootLoader。

### 1.2 BootLoader 又是什么？

Bootloader 的定义：Bootloader 是在操作系统运行之前执行的一小段程序，通过这一小段程序，我们可以初始化硬件设备、建立内存空间的映射表，从而建立适当的系统软硬件环境，为最终调用操作系统内核做好准备。意思就是说如果我们要想让一个操作系统在我们的板子上运转起来，我们就必须首先对我们的板子进行一些基本配置和初始化，然后才可以将操作系统引导进来运行。  
BootLoader 的主要运行任务就是将内核映象从硬盘上读到 RAM 中，然后跳转到内核的入口点去运行，即开始启动操作系统。

### 1.3 Windows 电脑和嵌入式设备启动流程对比

PC 机上电启动流程：上电 -> BIOS -> Windows -> 识别 C盘、D盘  
嵌入式设备上电启动流程：上电 -> BootLoader -> Linux Kernel -> 挂载根文件系统

## 二. 获取最新的 U-Boot

### 2.1 访问 U-Boot 官网

访问 U-Boot 官网：http://www.denx.de/wiki/U-Boot/WebHome

![图片1](https://raw.githubusercontent.com/mz8023yt/blog/master/image/u-boot/u-boot-get-latest-u-boot/01.png)

### 2.2 点击 Source Code，进入源码界面。

这里提示说要获取 U-Boot 的发布版本可以通过 FTP 服务器。

![图片2](https://raw.githubusercontent.com/mz8023yt/blog/master/image/u-boot//u-boot-get-latest-u-boot/02.png)

### 2.3 点击 FTP Server 进入 FTP 服务器文件列表界面

这里有各个版本的 U-Boot，当然也就包括最新版。

![图片3](https://raw.githubusercontent.com/mz8023yt/blog/master/image/u-boot//u-boot-get-latest-u-boot/03.png)

其实可以直接访问 [FTP Server](ftp://ftp.denx.de/pub/u-boot/) 获取各个版本的 U-Boot。
