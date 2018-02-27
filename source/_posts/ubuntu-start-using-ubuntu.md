---
title: '[Ubuntu] 开始使用 Ubuntu'
date: 2018-02-25 23:20:43
tags:
---

## 一. 系统配置

### 1.1 设置 root 用户密码

```
sudo passwd root
```

### 1.2 安装 32 位兼容库

经常会遇到 gcc 安装好了，环境变量配置好了，但是就是 arm-none-eabi-gcc -v 显示找不到命令，其实就是在 64 位的 ubuntu 下没有装 32 位兼容库，导致 32 的 arm-none-eabi-gcc 无法正常运行。

```
sudo apt-get install lib32ncurses5
sudo apt-get install lib32z1
```

### 1.3 安装 ncurses 基本库

ncurses 是字符终端下屏幕控制的基本库。可能很多新开发的程序都不使用了，不过如果要编译一些老程序，还经常遇得到。编译 kernel 和 u-boot 时 make menuconfig 命令就需要这个库。

```
sudo apt-get install libncurses5-dev
```

### 1.4 修改家目录文件夹名

倘若安装 ubuntu 的时候选择的语言是中文，则家目录下的文件夹名都是中文名称，这样在控制台 cd 切入子目录下要输入中文，很不方便。要是是英文的话就方便多了，但是我们直接将家目录下的文件家改为英文名后，之前的桌面、下载、文档等目录就全部都变成了家目录。其他的目录都还好，但是桌面变成了家目录，导致家目录下的所有文件都暴露在桌面上，很不开心。那怎么手动去指定桌面、文档、下载等文件夹的路径呢？

需要修改一个配置文件，该配置文件路径为：

```
user@vmware:~$ vim ~/.config/user-dirs.dirs
```

修改此文件中的 XDG_xxx_DIR 对应的目录便可以指定桌面、文档、下载等文件夹的路径。

这里有一段注释，简单翻译一下：  
此文件由 xdg-user-dirs-update 编写，如果你想要增加或者改变一下家目录下的目录结构，只需编辑你感兴趣的那一行。所有本地更改将在下次运行(重启)时生效。每一行的格式是 XDG_xxx_DIR ="$ HOME/yyy" 相对路径，或着 XDG_xxx_DIR = "/yyy" 绝对路径，不支持其他格式。

所以这里这样就搞定了，但是要生效的话需要重启。

## 二. 软件安装

### 2.1 apt-get 安装常用软件

#### 更新软件源

```
sudo apt-get update
```

#### 文本编辑器 Vim

```
sudo apt-get install vim-nox
```

#### pdf 阅读器 okular

```
sudo apt-get install okular
```

#### 截图工具 shutter

```
sudo apt-get install shutter
```

#### 视频播放器 VLC

```
sudo apt-get install vlc
```

#### 版本控制工具 git

```
sudo apt-get install git
```

### 2.2 deb 格式的常用软件

#### 搜狗输入法

```Bash
sudo dpkg -i sogoupinyin_2.1.0.0086_amd64.deb
sudo apt-get install -f
sudo dpkg -i sogoupinyin_2.1.0.0086_amd64.deb
```

设置 -> 文本输入 -> 拼音 -> 设置齿轮按钮 -> 删除其他输入法，只保留搜狗输入法和美式键盘，可以将搜狗输入法配置为默认输入法。

#### 文本编辑器 Atom

```Bash
sudo dpkg -i atom-amd64.deb
sudo apt-get install -f
sudo dpkg -i atom-amd64.deb
```

#### 音乐播放器网易云音乐

```Bash
sudo dpkg -i netease-cloud-music_1.0.0-2_amd64_ubuntu16.04.deb
sudo apt-get install -f
sudo dpkg -i netease-cloud-music_1.0.0-2_amd64_ubuntu16.04.deb
```

### 2.3 tar 格式常用软件

#### 邮箱客户端 Thunderbird

下载解压，直接运行即可。配置时需要注意一点，163登录不是直接用邮箱密码，而是使用授权密码。

```Bash
sudo tar -jxf thunderbird-52.4.0.tar.bz2 -C /opt/
cd /opt/thunderbird/
./thunderbird
```

## 三. 常用服务

### 3.1 安装 ssh 服务

#### ssh 介绍

SSH 为 Secure Shell 的缩写，由 IETF 的网络小组（Network Working Group）所制定；SSH 为建立在应用层基础上的安全协议。SSH 是目前较可靠，专为远程登录会话和其他网络服务提供安全性的协议。

一般我们在 windows 主机上使用远程登陆工具(Xshell等)登陆 linux 主机都是使用的 ssh 协议。

#### 安装 ssh 服务器

```
user@vmware:~$ sudo apt-get install openssh-server
```

#### 测试 ssh 服务器

配置好 windows 和 ubuntu 在同一个局域网中，在 windows 下安装 xshell 远程登录工具，远程登录到 ubuntu。看是否能够成功登陆。

### 3.2 安装 nfs 服务器

#### nfs 介绍

NFS 即网络文件系统(Network File-System)，可以通过网络让不同机器、不同系统之间可以实现文件共享。通过 NFS，可以访问远程共享目录,就像访问本地磁盘一样。在 ubuntu 主机上安装 nfs 服务器，开发板便可以通过网络访问 ubuntu 主机上的共享的文件。

#### 安装 nfs 服务器

```
user@vmware:~$ sudo apt-get install nfs-kernel-server       # 安装 NFS 服务器端
user@vmware:~$ sudo apt-get install nfs-common              # 安装 NFS 客户端
```

#### 配置 nfs 共享目录

安装完 NFS 服务器后，需要指定共享的 NFS 目录，其方法是在 "/etc/exports" 文件里面设置对应的目录及相应的访问权限，每一行对应一个设置。

配置 /home/user/board/ 目录为 nfs 共享的目录，需要修改 "/etc/exports" 文件，添加一行

```
/home/user/board/ *(rw,sync,no_root_squash) 
```

#### 建立 nfs 共享文件夹

修改完成后,保存并退出 /etc/exports 文件。然后新建 /home/user/board 目录,并为该目录设置最宽松的权限:

```
user@vmware:~$ sudo mkdir -p /home/user/board
user@vmware:~$ sudo chmod -R 777 /home/user/board
user@vmware:~$ sudo chown –R nobody /home/user/board
```

#### 启动 nfs 服务器

```
user@vmware:~$ sudo /etc/init.d/nfs-kernel-server start   # 开启 nfs 服务器
user@vmware:~$ sudo /etc/init.d/nfs-kernel-server restart # 重启 nfs 服务器
```

#### 测试 nfs 服务器

开发板接好网线，保证开发板和虚拟机在同一个局域网下，执行以下命令，挂载 /home/user/board 目录到开发板的 /mnt 目录下。

```
[root@FriendlyARM /]# sudo mount -t nfs 192.168.1.110:/home/user/board /mnt -o nolock
```

其中 192.168.1.100 是 Ubuntu 虚拟机 ip 地址，/home/user/board 是虚拟机 nfs 服务器共享的目录。

### 3.3 安装 samba 服务器

#### samba 介绍

Samba是在Linux和UNIX系统上实现SMB协议的一个免费软件，由服务器及客户端程序构成。
SMB（Server Messages Block，信息服务块）是一种在局域网上共享文件和打印机的一种通信协议，它为局域网内的不同计算机之间提供文件及打印机等资源的共享服务。

#### 安装 samba 服务器

```
user@vmware:~$ sudo apt-get install samba
```

#### 配置 samba 选项

```
user@vmware:~$ sudo vim /etc/samba/smb.conf
```

配置文件追加：

```
[Ubuntu]                         # windows 映射网络位置时显示的文件夹名
   comment = ubuntu share        # 提示信息，不重要，随便写个字符串就好了
   path = /home/user/workspace/  # 用于共享的虚拟机文件夹路径
   writable = yes                # windows 映射后是否可写
   browseable = yes              # windows 映射后是否可浏览
```

#### 设置 samba 用户

```
user@vmware:~$ sudo smbpasswd -a user
```

#### 重启 samba 服务器

```
user@vmware:~$ sudo /etc/init.d/smbd restart
user@vmware:~$ sudo /etc/init.d/nmbd restart
```

#### 映射 samba 网络位置
