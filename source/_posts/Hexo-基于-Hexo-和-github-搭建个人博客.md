---
title: '[Hexo] 基于 Hexo 和 github 搭建个人博客'
date: 2018-01-21 18:02:44
tags:
  - github
  - git
categories: hexo
---

## 一. 前言

## 二. 配置博客环境
### 2.1 nodejs
#### 2.1.1 node 介绍
安装 nodejs 的目的是为了使用 hexo 组件，关于 node 和 hexo 是什么东西，我一个做嵌入式开发的小伙子并不懂，具体还请各位自行访问其官网或者百度了解。

#### 2.1.2 下载 nodejs
百度搜索 nodejs 或者直接点击访问 <https://nodejs.org/zh-cn/> nodejs 官网。  
下载最新版本的版本的 nodejs，由于我使用的是 Ubuntu 平台，下载后在 ~/download 目录下有对应的 linux 版本 nodejs 压缩包。
```
user@lenovo:~/download$ ls
node-v8.9.4-linux-x64.tar.xz
```

#### 2.1.3 解压 nodejs
这里我是直接在 ~/ 目录下创建了一个 ~/opt/ 目录，用来安装 nodejs 软件。  
```
user@lenovo:~$ mkdir opt
user@lenovo:~$ tar Jxf download/node-v8.9.4-linux-x64.tar.xz -C opt/
```
为什么不直接将 nodejs 安装在 /opt 目录下呢？  
其实我也是逼不得已，我之前确实是有将 nodejs 解压到 /opt 目录下，并成功配置好环境变量后，查看 nodejs 版本也是 ok 的。但是执行 npm install -g hexo 的时候，提示说没有权限，然而加上 sudo，还是不行，最后只能在自己 ~/ 目录下创建一个安装软件的 /opt 目录了。

#### 2.1.4 配置环境变量
上述解压步骤执行完之后，nodejs 基本上就可以说是已经安装了，我们直接在 /home/user/opt/node-v8.9.4-linux-x64/bin 目录下敲 node 相关命令是可以执行的。但是在终端的其他的位置则无法使用 node 命令，原因其实很简单，我们在 shell 中执行的每一条命令都有其对应的可执行文件，理理论上要执行这些文件都需要切换到可执行文件对应的目录或者通过可执行文件的全路径指定。  
可是这样实在是太麻烦了，linux 博大精深，不可能没有应对机制，shell 有一个机制，在 shell 中解析命令的时候，会先在当前目录下找命令的可执行文件，如果当前目录下找不到的话，则会根据 PATH 这个环境指定的目录灾区找可执行文件。  
这就豁然开朗了，也就明nodejs白了为什么我们要配置环境变量了，为的是不管在哪一个目录下都可以直接使用 node 命令，而不需要指定 node 全路径。
```
user@lenovo:~$ echo "export PATH=$PATH:/home/user/opt/node-v8.9.4-linux-x64/bin" >> .bashrc
user@lenovo:~$ . .bashrc
```

#### 2.1.5 验证 nodejs 是否配置成功
在任意目录下，执行任意一条 node 命令即可验证配置是否生效。  
就用最简单的 node 查看版本号的命令吧。
```
user@lenovo:~$ node -v
v8.9.4
```

### 2.2 git & github
#### 2.2.1 git 和 github 介绍
git 是一个版本控制工具，github 是一个代码托管平台。  
版本控制是干什么的呢？详细介绍请移步[廖雪峰的git教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/001373962845513aefd77a99f4145f0a2c7a7ca057e7570000)  
托管平台又是什么东西呢？我们将其类比为百度网盘这样的云端备份就可以了。  
我们本地的 code 使用 git 来管理，同时我们将 code 拷贝一份到 github 上，做备份用，只不过这个备份的副本，其他人也可以修改，这其实就是 git 的协作功能。

#### 2.2.2 安装 git
直接使用 Ubuntu 的软件包管理器安装即可。
```
user@lenovo:~$ sudo apt-get install git
```
备注：  
apt-get 是高级包装工具(Advanced Packaging Tools)是 Debian 及其衍生发行版(eg.Ubuntu)的软件包管理器。APT可以自动下载，配置，安装二进制或者源代码格式的软件包，因此简化了 Unix 系统上管理软件的过程，apt-get 命令一般需要 root 权限执行，所以一般跟着 sudo 命令。

#### 2.2.3 配置 git
因为 git 是分布式版本控制系统，所以，每个机器都必须自报家门：你的名字和 Email 地址。
```
user@lenovo:~$ git config --global user.name mz8023yt
user@lenovo:~$ git config --global user.email mz8023yt@163.com
```
有没有经常敲错命令？比如 git status？哎 status 这个单词真心不好记。  
如果敲 git st 就表示 git status 那就简单多了，当然这种偷懒的办法我们是极力赞成的。我们只需要敲一行命令，告诉 git，以后 st 就表示 status：
```
user@lenovo:~$ git config --global alias.st status
user@lenovo:~$ git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```

#### 2.2.5 创建 ssh 秘钥对
git 和 github 之间是通过 ssh 加密协议通信的，因此需要创建一对 ssh 秘钥对。
```
user@lenovo:~$ ssh-keygen -t rsa -C mz8023yt@163.com
```

#### 2.2.6 创建 github 仓库
第一步，肯定是注册一个 github 账号了，我的账户名是 mz8023yt。  
第二步，创建 mz8023yt.github.io 仓库。由于我们是通过 github.io 机制搭建个人博客，因此需要创建和用户名同名的 github.io 仓库。  
第三步，创建一个 blog 仓库，这个仓库是 hexo blog 的源码。  
第四步，将 ssh 公钥添加到 github 中。

### 2.3 hexo
#### 2.3.1 hexo 介绍
什么是 Hexo？  
Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

#### 2.3.2 安装 hexo
安装 Hexo 相当简单。然而在安装前，必须检查电脑中是否已安装下列应用程序：
 - Node.js
 - Git

上面这两个工具我们已经安装好了，因此接下来只需要使用 npm 即可完成 Hexo 的安装。
```
# user@lenovo:~$ npm install -g hexo-cli # hexo 官网推荐使用这一条命令
user@lenovo:~$ npm install -g hexo
npm WARN deprecated swig@1.4.2: This package is no longer maintained
... ...
+ hexo@3.4.4
added 253 packages in 24.503s
```
查看一下 hexo 的版本好，确认 hexo 安装成功。
```
user@lenovo:~$ hexo -v
hexo-cli: 1.0.4
os: Linux 4.10.0-28-generic linux x64
http_parser: 2.7.0
node: 8.9.4
v8: 6.1.534.50
uv: 1.15.0
zlib: 1.2.11
ares: 1.10.1-DEV
modules: 57
nghttp2: 1.25.0
openssl: 1.0.2n
icu: 59.1
unicode: 9.0
cldr: 31.0.1
tz: 2017b
```

#### 2.3.3 获取 hexo 站点源文件
安装 Hexo 完成后，请执行下列命令，Hexo 将会在指定文件夹中新建所需要的文件。
```
user@lenovo:~$ mkdir blog
user@lenovo:~$ cd blog/
user@lenovo:~/blog$ hexo init
INFO  Cloning hexo-starter to ~/blog
正克隆到 '/home/user/blog'...
... ...
added 315 packages in 12.99s
INFO  Start blogging with Hexo!
```

#### 2.3.4 使用 git 管理网站源文件
这一步不是必须的，但我还是觉得很有必要。  
为什么我觉得很有必要，比如说换电脑了或者重装系统了，源码还是有备份的。
```
user@lenovo:~/blog$ git init
初始化空的 Git 仓库于 /home/user/blog/.git/
user@lenovo:~/blog$ git remote add mz8023yt git@github.com:mz8023yt/blog.git
user@lenovo:~/blog$ git add --all
user@lenovo:~/blog$ git commit -m "feature: start the blog with hexo"
```

#### 2.3.5 修改配置文件

```
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
-  type:
+  type: git
+  repo: https://github.com/mz8023yt/mz8023yt.github.io.git
+  branch: master
```

#### 2.3.6 生成博客静态页面并部署
使用以下命令进行网站的部署：
 - hexo c: hexo clean 清除生成的静态页面
 - hexo g: hexo generate 重新生成博客静态页面
 - hexo d: hexo deploy 部署到 github.io 仓库

```
user@lenovo:~/blog$ hexo c && hexo g && hexo d
... ...
INFO  28 files generated in 620 ms
ERROR Deployer not found: git
```
报错了，不慌，百度说这样可以解决，试试。
```
user@lenovo:~/blog$ npm install --save hexo-deployer-git
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@1.1.3 (node_modules/fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.1.3: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})
+ hexo-deployer-git@0.3.1
added 16 packages in 7.932s
```
再来一次。
```
user@lenovo:~/blog$ hexo c && hexo g && hexo d
To git@github.com:mz8023yt/mz8023yt.github.io.git
 * [new branch]      HEAD -> master
分支 master 设置为跟踪来自 git@github.com:mz8023yt/mz8023yt.github.io.git 的远程分支 master。
INFO  Deploy done: git
```
很好，成功了，访问 <https://mz8023yt.github.io/> 看看。

## 三. 更换博客主题
