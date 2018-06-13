---
title: '[Python] 安装 python3'
date: 2018-06-13 22:26:14
tags:
  - python
categories: Python
---

### 一 ubuntu 下安装 Python3

1. 使用 apt-get 安装 python3

       user@vmware:~$ sudo apt-get install python3.5

2. 验证是否安装成功

       user@vmware:~$ python
       Python 2.7.12 (default, Nov 20 2017, 18:23:56) 
       [GCC 5.4.0 20160609] on linux2
       Type "help", "copyright", "credits" or "license" for more information.
       >>> 

   很奇怪，明明安装的是 python3.5，但是执行时调用的却是 python2.7

3. 找一下版本不对的原因  
   搜索下 python 命令看看
   
       user@vmware:~$ cd /
       user@vmware:/$ sudo find -name "python"
       ./var/lib/python
       ./usr/bin/python
       ./usr/lib/libreoffice/share/Scripts/python
       ./usr/share/gdb/python
       ./usr/share/gcc-5/python
       ./usr/share/lintian/overrides/python
       ./usr/share/doc/python
       ./usr/share/bash-completion/completions/python
       ./usr/share/python
       ./usr/share/librevenge/python
       ./etc/apparmor.d/abstractions/python
       ./etc/python

   上面的搜索结果就 `./usr/bin/python` 有点像可执行文件，看一眼

       user@vmware:/$ ls -l /usr/bin/python
       lrwxrwxrwx 1 root root 18 6月  13 22:34 /usr/bin/python -> /usr/bin/python2.7

   确认到 python 命令其实是指向同级目录下 python2.7 的符号连接，看看 `/usr/bin/` 目录下的 python 命令集合

       user@vmware:/$ ls -l /usr/bin/python*
       lrwxrwxrwx 1 root root      18 6月  13 22:34 /usr/bin/python -> /usr/bin/python2.7
       lrwxrwxrwx 1 root root       9 4月  27 23:20 /usr/bin/python2 -> python2.7
       -rwxr-xr-x 1 root root 3542008 11月 24  2017 /usr/bin/python2.7
       lrwxrwxrwx 1 root root       9 4月  27 23:20 /usr/bin/python3 -> python3.5
       -rwxr-xr-x 2 root root 4464400 11月 29  2017 /usr/bin/python3.5
       -rwxr-xr-x 2 root root 4464400 11月 29  2017 /usr/bin/python3.5m
       lrwxrwxrwx 1 root root      10 4月  27 23:20 /usr/bin/python3m -> python3.5m

   发现，还有 python3.5，因此只要将 python 链接到 python3.5 就可以了

4. 修改 python 链接  

       user@vmware:/$ sudo rm -rf /usr/bin/python
       user@vmware:/$ sudo ln -s /usr/bin/python3.5 /usr/bin/python
       user@vmware:/$ ls -l /usr/bin/python
       lrwxrwxrwx 1 root root 18 6月  13 22:46 /usr/bin/python -> /usr/bin/python3.5

5. 再次验证是否安装成功

       user@vmware:/$ python
       Python 3.5.2 (default, Nov 23 2017, 16:37:01) 
       [GCC 5.4.0 20160609] on linux
       Type "help", "copyright", "credits" or "license" for more information.
       >>> 
