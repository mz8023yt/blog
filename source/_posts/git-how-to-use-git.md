---
title: '[Git] Git 的安装及配置'
date: 2018-01-23 21:57:54
tags:
  - github
categories: Git
---

## 一. 安装配置 git

### 1. Ubuntu 下安装 git

```bash
user@ubuntu:~$ sudo apt-get install git
```

### 2. 配置邮箱和用户名

```bash
user@ubuntu:~$ git config --global user.name mz8023yt
user@ubuntu:~$ git config --global user.email mz8023yt@163.com
```

### 3. 配置命令别名

```bash
user@ubuntu:~$ git config --global alias.st status
user@ubuntu:~$ git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an> %Creset' --abbrev-commit"
```

### 4. 生成 ssh 秘钥对

```bash
user@ubuntu:~$ ssh-keygen -t rsa -C mz8023yt@163.com
```


### 5. 将 shh 公钥添加到代码托管平台

```bash
user@ubuntu:~$ cat ~/.ssh/id_rsa.pub
```

登录 github、coding、oschina，添加 shh 公钥。将 cat 打印出来的 id_rsa.pub 公钥添加到托管平台账户中。
