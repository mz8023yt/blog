---
title: '[Hexo] 精通 Markdown'
date: 2018-04-22 17:50:20
tags:
  - Markdown
  - GitHub
categories: Hexo
---

原文链接: <https://guides.github.com/features/mastering-markdown/>

百度搜索 Markdown GitHub，即可搜索到 `Mastering Markdown · GitHub Guides` 页面，找到原文。

### 前言

Markdown 可以写一些轻量级格式的文档，并且它使用纯文本的方式就可以描述出来。使用纯文本的方式相对于复杂的 word 文档就带来一个非常大的优势，那就是可以很方便的使用版本控制工具去管理。  
本文不会详细的去介绍 Markdown 的详细语法，详细的语法，上面有给出 GitHub 的使用手册，这里仅仅介绍一些常用的语法。

### 超链接

    使用 [链接文字描述](超链接) 的方式可以很容易的添加超链接：
    [link to Google](http://google.com)

[link to Google](http://google.com)

### 图片

    使用 ![图片描述](图片链接) 的方式可以很容易的添加图片，这个图片也可以是 gif 动态图：
    eg. ![adb input command demo](https://raw.githubusercontent.com/mz8023yt/blog.material/master/odm/gif/odm-tp-input-cmd.gif)

![adb input command demo](https://raw.githubusercontent.com/mz8023yt/blog.material/master/odm/gif/odm-tp-input-cmd.gif)

### 内嵌代码块

        在句子中使用 `<code block>` 在句子中嵌入代码块：
        eg. 使用 printk 在 kernel 中打印调试信息。

使用 `printk` 在 kernel 中打印调试信息。

### 引用

        在句首使用 > 引用其他人的句子，句子前会有一个竖线
        eg. > 不经过自己充分思考就去问别人问题，是一种不礼貌的行为，尤其是向这个行业的各位专家们请教问题！ 

> 不经过自己充分思考就去问别人问题，是一种不礼貌的行为，尤其是向这个行业的各位专家们请教问题！ 

### 删除线

        使用 ~~ 或者 <del> 标记实现删除线效果。
        eg. ~~我是被删除的~~
        eg. <del>我也是被删除的<del>

~~我是被删除的~~
<del>我也是被删除的<del>
