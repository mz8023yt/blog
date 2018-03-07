---
title: '[Hexo] 基于 Hexo 和 github 搭建个人博客'
date: 2018-01-21 18:02:44
tags:
  - github
  - git
categories: Hexo
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
user@lenovo:~$ npm install -g hexo-cli
npm WARN deprecated swig@1.4.2: This package is no longer maintained
... ...
+ hexo@3.4.4
added 253 packages in 24.503s
```

查看一下 hexo 的版本号，确认 hexo 安装成功。

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

#### 2.3.6 生成博客页面并部署

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

```bash
user@lenovo:~/blog$ hexo c && hexo g && hexo d
To git@github.com:mz8023yt/mz8023yt.github.io.git
 * [new branch]      HEAD -> master
分支 master 设置为跟踪来自 git@github.com:mz8023yt/mz8023yt.github.io.git 的远程分支 master。
INFO  Deploy done: git
```
很好，成功了，访问 <https://mz8023yt.github.io/> 看看。

## 三. 更换 next 主题

hexo 默认的主题不是很好看，在 hexo 文档中心中有对应的文档介绍[如何更换主题](https://hexo.io/zh-cn/docs/themes.html)以及给出众多[官网推荐的主题](https://hexo.io/themes/)。

### 3.1 为什么选择 next

我个人选择 nxet 是基于以下几个理由：

 - 界面简洁美观
 - 支持点击跳转的侧栏目录
 - 支持代码高亮
 - 详细清晰的官网文档

### 3.2 安装 next 主题

既然都说了，next 官方文档特别的详细清晰，那就直接跳转到[官方文档](http://theme-next.iissnan.com/)去看看吧。  
官网给出了两种方式获取 next 主题的方式，一种是直接 clone 仓库，一种则是直接下载打包好的 next 稳定版本。  
获取到 next 主题包后，解压所下载的压缩包至站点的 themes 目录下， 并将解压后的文件夹名称更改为 next。  
最后将 hexo 站点配置文件  中的 theme 修改为 next。

```bash
 # git diff _config.yml
@@ -72,7 +72,7 @@ pagination_dir: page
 # Extensions
 ## Plugins: https://hexo.io/plugins/
 ## Themes: https://hexo.io/themes/
-theme: landscape
+theme: next

 # Deployment
 ## Docs: https://hexo.io/docs/deployment.html
```

### 3.3 配置 hexo 站点

#### 3.3.1 修改站点语言为中文

这里直接修改站点配置文件的 language 为 zh-Hans 即可。  
这里可能有的小伙伴会觉得有点奇怪，zh-Hans 是什么东西？不应是 zh-CN 吗？  
其实这是因为 next 主题中将中文对应的资源文件命名为 zh-Hans。

```bash
 # git diff _config.yml
@@ -7,7 +7,7 @@ title: Hexo
 subtitle:
 description:
 author: John Doe
-language:
+language: zh-Hans
 timezone:

 # URL
```

#### 3.3.2 修改站点描述信息

```bash
 # git diff _config.yml
@@ -3,10 +3,10 @@
## Source: https://github.com/hexojs/hexo/

# Site
-title: Hexo
-subtitle:
-description:
-author: John Doe
+title: Paul's blog
+subtitle: "学而不思则罔 思而不学则殆"
+description: "但行好事 莫问前程"
+author: Paul Wang
language: zh-Hans
timezone:
```

### 3.4 配置 next 主题

#### 3.4.1 博客主页显示博文预览

刚刚安装好 next 之后，不难发现，在 next 的首页上先博文所有的内容都贴出来，这样十分不方便我们查找博文，最好是能够像其他博客网站一样，能够有个预览界面，截取博文中的一小段，贴在首页即可。

博客预览有两种设置方法：

 - 第一种是设置自动生成摘要，仅仅截取文章开头的部分文字。
 - 第二种也是截取文章开口的部分，并且会保留原始格式，但是需要每一篇文章都通过 <\!\-\- more \-\-\> 指定从文章开头到预览结束的位置。

```bash
 # git diff themes/next/_config.yml
@@ -216,7 +216,7 @@ excerpt_description: true
 # Automatically Excerpt. Not recommend.
 # Please use <!-- more --> in the post to control excerpt accurately.
 auto_excerpt:
-  enable: false
+  enable: true
   length: 150

 # Post meta display settings
```

#### 3.4.2 取消侧栏目录自动编号

侧边栏自动创建目录，并且点击可以跳转的功能是我选择 next 最重要的理由。这个功能实在是太贴心了，尤其对于比较长的文章，很是受用。  
但是很多的时候我们会为自己的文章的层次结构做好规划，有自己的给目录编号的方式，这个时候就需要取消目录自动编号的功能。

```bash
 # git diff themes/next/_config.yml
@@ -161,7 +161,7 @@ toc:
   enable: true

   # Automatically add list number to toc.
-  number: true
+  number: false

   # If true, all words will placed on next lines if header width longer then sidebar width.
   wrap: false
```

#### 3.4.3 添加博客头像

将头像的资源文件放在站点文件目录中的某一处，然后在 next 主题配置文件中指定 avator 属性为该资源的路径即可。  

```bash
 # git diff themes/next/_config.yml
@@ -154,7 +154,7 @@ links_layout: block
 # Sidebar Avatar
 # in theme directory(source/images): /images/avatar.gif
 # in site  directory(source/uploads): /uploads/avatar.gif
-#avatar: /images/avatar.gif
+avatar: /images/avatar.jpg

 # Table Of Contents in the Sidebar
 toc:tubiao
```

#### 3.4.4 修改网站图标

将网站图标相关的资源文件放在站点文件目录中的某一处，然后在 next 主题配置文件中指定下面的属性为该资源的路径即可。  
这里值得注意的是，next 有好几个站点图标的资源，我也不清除具体在什么场景下哪一个会生效，因此我全部找了对应分辨率的资源将之替换。

这里顺便推荐一个图标网站：[阿里适量图库](http://iconfont.cn/home/index)。

```bash
 # git diff themes/next/_config.yml
@@ -27,10 +27,14 @@ override: false
 # For example, you put your favicons into `hexo-site/source/images` directory.
 # Then need to rename & redefine they on any other names, otherwise icons from Next will rewrite your custom icons in Hexo.
 favicon:
-  small: /images/favicon-16x16-next.png
-  medium: /images/favicon-32x32-next.png
-  apple_touch_icon: /images/apple-touch-icon-next.png
-  safari_pinned_tab: /images/logo.svg
+  #small: /images/favicon-16x16-next.png
+  #medium: /images/favicon-32x32-next.png
+  #apple_touch_icon: /images/apple-touch-icon-next.png
+  #safari_pinned_tab: /images/logo.svg
+  small: /favicon/favicon-16x16.png
+  medium: /favicon/favicon-32x32.png
+  apple_touch_icon: /favicon/favicon.png
+  safari_pinned_tab: /favicon/favicon.svg
   #android_manifest: /images/manifest.json
   #ms_browserconfig: /images/browserconfig.xml
```

#### 3.4.5 修改代码高亮的方式

```bash
 # git diff themes/next/_config.yml
@@ -277,7 +277,7 @@ custom_logo:
 # Available value:
 #    normal | night | night eighties | night blue | night bright
 # https://github.com/chriskempson/tomorrow-theme
-highlight_theme: normal
+highlight_theme: night


 # ---------------------------------------------------------------
```

#### 3.4.6 添加侧边栏社交链接

```bash
 # git diff themes/next/_config.yml
@@ -130,9 +130,9 @@ scheme: Gemini
 # Key is the link label showing to end users.
 # Value before `||` delimeter is the target permalink.
 # Value after `||` delimeter is the name of FontAwesome icon.
 #   If icon (with or without delimeter) is not specified, globe icon will be loaded.
-#social:
-  #GitHub: https://github.com/yourname || github
-  #E-Mail: mailto:yourname@gmail.com || envelope
+social:
+  E-Mail: mailto:mz8023yt@163.com || envelope
+  GitHub: https://github.com/mz8023yt || github
   #Google: https://plus.google.com/yourname || google
   #Twitter: https://twitter.com/yourname || twitter
   #FB Page: https://www.facebook.com/yourname || facebook
```

#### 3.4.7 加宽博文宽度

对于显示器比较大的电脑，在使用 next 主题的时候，发现两边大量的留白，对于站点整理的美观有所影响。  
能不能将文章对应的宽度修改的宽一点呢？  

```bash
 # git diff themes/next/source/css/_schemes/Pisces/_layout.styl
@@ -1,7 +1,7 @@
 .header {
   position: relative;
   margin: 0 auto;
-  width: $main-desktop;
+  width: 80%;

   +tablet() {
     width: auto;
@@ -47,7 +47,7 @@
 }

 .container .main-inner {
-  width: $main-desktop;
+  width: 80%;

   +tablet() {
     width: auto;
@@ -61,7 +61,7 @@
   float: right;
   box-sizing: border-box;
   padding: $content-desktop-padding;
-  width: $content-desktop;
+  width: calc(100% - 2能不能将文章对应的宽度修改的宽一点呢？60px);
   background: white;
   min-height: 700px;
   box-shadow: $box-shadow-inner;
@@ -127,4 +127,3 @@
     padding-right: 260px;
   }
 }
```

### 3.5 添加页面

#### 3.5.1 添加关于界面

参考 [Next 官网文档](http://theme-next.iissnan.com/theme-settings.html)，添加页面可以用 hexo new page 命令。

```bash
user@lenovo:~/blog$ hexo new page "about"
INFO  Created: ~/blog/source/about/index.md
```

在生成的 index.md 下编辑你的关于页面即可。

#### 3.5.2 添加标签界面

首先，在终端窗口下，定位到 Hexo 站点目录下。使用 hexo new page 新建一个页面，命名为 tags。

```bash
user@lenovo:~/blog$ hexo new page "tags"
INFO  Created: ~/blog/source/tags/index.md
```

修改 ~/blog/source/tags/index.md 文件，设置页面类型为 tags。

```bash
 # git diff source/tags/index.md
@@ -0,0 +1,5 @@
+---
+title: 标签
+date: 2018-01-22 21:19:54
+type: tags
+---
```

最后修改主题配置文件，在侧边栏添加标签菜单项。

```bash
 # git diff themes/next/_config.yml
@@ -98,9 +98,9 @@ index_with_subtitle: false
 menu:
   home: / || home
   categories: /categories/ || th
   about: /about/ || user
-  #tags: /tags/ || tags
+  tags: /tags/ || tags
   archives: /archives/ || archive
   about: /about/ || user
   #schedule: /schedule/ || calendar
   #sitemap: /sitemap.xml || sitemap
   #commonweal: /404/ || heartbeat
```

#### 3.5.3 添加分类界面

分类界面的添加和上小节的 tags 很类似。  
首先，在终端窗口下，定位到 Hexo 站点目录下。使用 hexo new page 新建一个页面，命名为 categories。

```bash
user@lenovo:~/blog$ hexo new page "categories"
INFO  Created: ~/blog/source/categories/index.md
```

修改 ~/blog/source/categories/index.md 文件，设置页面类型为 categories。

```bash
 # git diff source/categories/index.md
@@ -0,0 +1,5 @@
+---
+title: 标签
+date: 2018-01-22 21:19:54
+type: categories
+---
```

最后修改主题配置文件，在侧边栏添加标签菜单项。

```bash
 # git diff themes/next/_config.yml
@@ -97,9 +97,9 @@ index_with_subtitle: false
 menu:
   home: / || home
+  categories: /categories/ || th
   about: /about/ || user
   #tags: /tags/ || tags
-  #categories: /categories/ || th
   archives: /archives/ || archive
   #schedule: /schedule/ || calendar
   #sitemap: /sitemap.xml || sitemap
```

## 四. 开始写第一篇文章

### 4.1 新建文章

### 4.2 预览文章

### 4.3 发布文章

## 五. 重新搭建博客

### 5.1 重装系统或者更换电脑

之前描述的是在 ubuntu 环境下搭建博客环境，现在我买了一台  windows 主机或者说重装了一下系统，但是我想将我的博客源码移植到 windows 继续使用怎么办？  
其实只要将之前的 blog 源码拷贝到新的电脑上，重新配置一下 nodejs、hexo、git 就好。

### 5.2 重新搭建环境和从零开始的区别

安装 nodejs 和 git 和之前一样，请参考第二章。唯一有区别的是 hexo 的安装。

先不管那么多，执行 npm 命令安装 hexo 看看。

```
$ npm install -g hexo-cli
C:\Users\mz802\AppData\Roaming\npm\hexo -> C:\Users\mz802\AppData\Roaming\npm\node_modules\hexo-cli\bin\hexo
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@1.1.3 (node_modules\hexo-cli\node_modules\fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.1.3: wanted {"os":"darwin","arch":"any"} (current: {"os":"win32","arch":"x64"})

+ hexo-cli@1.0.4
added 217 packages in 18.824s
```

安装好了，看看版本信息确认一下，看看是不是 ok 了。

```
$ hexo -v
ERROR Local hexo not found in E:\blog
ERROR Try running: 'npm install hexo --save'
```

有问题？不过没有关系，提示不说叫我们试试 npm install hexo --save 命令吗，那不妨试一下。

```
$ npm install hexo --save

> nunjucks@3.1.2 postinstall E:\blog\node_modules\hexo\node_modules\nunjucks
> node postinstall-build.js src

npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@1.1.3 (node_modules\fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.1.3: wanted {"os":"darwin","arch":"any"} (current: {"os":"win32","arch":"x64"})

+ hexo@3.5.0
added 470 packages in 17.778s
```

成功了，再看看版本信息。

```
$ hexo -v
hexo: 3.5.0
hexo-cli: 1.0.4
os: Windows_NT 10.0.16299 win32 x64
http_parser: 2.7.0
node: 8.9.3
v8: 6.1.534.48
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

此时如果细心的话，不难发现，package-lock.json 和 package.json 文件有更新，这里自己去看看 git diff 吧，其实是 hexo 版本有变化，才更新到了这两个文件。

## 六. 常见问题

### 6.1 无法访问 localhost:4000

hexo 安装成功，并且正确运行，但是执行 hexo s 的时候，出现 localhost:4000 不能访问。  
百度查了下是因为 hexo 默认使用 4000 端口，但是如果安装了福昕阅读器，则 4000 端口已经被福昕阅读器使用了，导致 hexo 没有办法使用 4000 端口。解决方法是换个端口，使用 -p 选项可以指定端口号。

```
hexo s -p 5000
```

这里换成5000端口，访问 localhost:4000 正常访问。
