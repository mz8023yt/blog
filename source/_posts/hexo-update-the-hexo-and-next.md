---
title: '[Hexo] 升级 next 版本到 6.2.0'
date: 2018-04-22 09:19:12
tags:
  - hexo
  - next
categories: Hexo
---

#### 下载最新版本的 next

https://github.com/theme-next/hexo-theme-next/releases

#### 修改语言为中文

```
diff --git a/_config.yml b/_config.yml
index 8a1abf3..cbd41d3 100644
--- a/_config.yml
+++ b/_config.yml
@@ -7,7 +7,7 @@ title: Paul's blog
 subtitle: "学而不思则罔 思而不学则殆"
 description: "但行好事 莫问前程"
 author: Paul Wang
-language: zh-Hans
+language: zh-CN
 timezone:

 # URL
```

#### 设置首页文章预览

```
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index 5007812..9492813 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -254,7 +254,7 @@ excerpt_description: true
 # Automatically Excerpt. Not recommend.
 # Please use <!-- more --> in the post to control excerpt accurately.
 auto_excerpt:
-  enable: false
+  enable: true
   length: 150

 # Post meta display settings
```

#### 修改主题风格为 Mist

```
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index 9492813..e231be5 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -136,8 +136,8 @@ menu_settings:
 # ---------------------------------------------------------------

 # Schemes
-scheme: Muse
-#scheme: Mist
+#scheme: Muse
+scheme: Mist
 #scheme: Pisces
 #scheme: Gemini
```

#### 添加标题栏页面

```
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index e231be5..5bf65c3 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -118,10 +118,10 @@ index_with_subtitle: false
 # Value after `||` delimeter is the name of FontAwesome icon. If icon (with or without delimeter) is not specified, question icon will be loaded.
 menu:
   home: / || home
-  #about: /about/ || user
-  #tags: /tags/ || tags
-  #categories: /categories/ || th
   archives: /archives/ || archive
+  categories: /categories/ || th
+  tags: /tags/ || tags
+  about: /about/ || user
   #schedule: /schedule/ || calendar
   #sitemap: /sitemap.xml || sitemap
   #commonweal: /404/ || heartbeat
```

#### 取消侧栏目录自动编号

```
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index 5bf65c3..74974d3 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -199,7 +199,7 @@ toc:
   enable: true

   # Automatically add list number to toc.
-  number: true
+  number: false

   # If true, all words will placed on next lines if header width longer then sidebar width.
   wrap: false
```

#### 修改网站图标

```
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index 74974d3..2e315a7 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -45,10 +45,10 @@ cache:
 # For example, you put your favicons into `hexo-site/source/images` directory.
 # Then need to rename & redefine they on any other names, otherwise icons from Next will rewrite your custom icons in Hexo.
 favicon:
-  small: /images/favicon-16x16-next.png
-  medium: /images/favicon-32x32-next.png
-  apple_touch_icon: /images/apple-touch-icon-next.png
-  safari_pinned_tab: /images/logo.svg
+  small: /maziot/maziot-16x16.png
+  medium: /maziot/maziot-32x32.png
+  apple_touch_icon: /maziot/maziot-apple.png
+  safari_pinned_tab: /maziot/maziot.svg
   #android_manifest: /images/manifest.json
   #ms_browserconfig: /images/browserconfig.xml
```

#### 修改博客头像

```
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index 2e315a7..caa9ecc 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -192,7 +192,7 @@ links_layout: block
 # Sidebar Avatar
 # in theme directory(source/images): /images/avatar.gif
 # in site  directory(source/uploads): /uploads/avatar.gif
-#avatar: /images/avatar.gif
+avatar: /maziot/avatar.jpg

 # Table Of Contents in the Sidebar
 toc:
```

#### 添加侧栏社交链接

```
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index caa9ecc..9a922b0 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -154,7 +154,12 @@ site_state: true
 # Key is the link label showing to end users.
 # Value before `||` delimeter is the target permalink.
 # Value after `||` delimeter is the name of FontAwesome icon. If icon (with or without delimeter) is not specified, globe icon will be loaded.
-#social:
+# 图标可以在 http://fontawesome.dashgame.com/ 网站上找
+social:
+  E-Mail: mailto:mz8023yt@163.com || envelope
+  GitHub: https://github.com/mz8023yt || github
+  Jianshu: https://www.jianshu.com/u/30fc01f695b9 || book
+  Geeker: http://www.androidgeeker.com/ || android
   #GitHub: https://github.com/yourname || github
   #E-Mail: mailto:yourname@gmail.com || envelope
   #Google: https://plus.google.com/yourname || google
```

#### 添加侧栏友情链接

```
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index 9a922b0..b325026 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -191,8 +191,13 @@ links_icon: link
 links_title: Links
 links_layout: block
 #links_layout: inline
-#links:
-  #Title: http://example.com/
+links:
+  icon: http://fontawesome.dashgame.com/
+  kernel: https://www.kernel.org/
+  mipi: https://mipi.org/
+  git: https://git-scm.com/
+  hexo: https://hexo.io/
+  next: http://theme-next.iissnan.com/

 # Sidebar Avatar
 # in theme directory(source/images): /images/avatar.gif
```

#### 修改友情链接布局

```
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index b325026..55a78f9 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -189,8 +189,8 @@ social_icons:
 # Blog rolls
 links_icon: link
 links_title: Links
-links_layout: block
-#links_layout: inline
+#links_layout: block
+links_layout: inline
 links:
   icon: http://fontawesome.dashgame.com/
   kernel: https://www.kernel.org/
```

#### 添加字数统计

https://github.com/theme-next/hexo-symbols-count-time

        npm install hexo-symbols-count-time --save

```
diff --git a/_config.yml b/_config.yml
index cbd41d3..025ef2d 100644
--- a/_config.yml
+++ b/_config.yml
@@ -10,6 +10,13 @@ author: Paul Wang
 language: zh-CN
 timezone:

+# Word Count
+symbols_count_time:
+  symbols: true
+  time: true
+  total_symbols: true
+  total_time: true
+
 # URL
 ## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
 url: http://yoursite.com
```

#### 添加文章打赏

```
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index 1748ad2..a5c5d51 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -293,9 +293,9 @@ codeblock:
   #description: ex. subscribe to my blog by scanning my public wechat account

 # Reward
-#reward_comment: Donate comment here
-#wechatpay: /images/wechatpay.jpg
-#alipay: /images/alipay.jpg
+reward_comment: 坚持原创技术分享，您的支持将鼓励我继续创作！
+wechatpay: /maziot/weixin.png
+alipay: /maziot/alipay.png
 #bitcoin: /images/bitcoin.png

 # Related popular posts

```

#### 添加评论系统

```
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index 4b0a05f..c751ba3 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -521,18 +521,29 @@ valine:
 # Gitment
 # Introduction: https://imsun.net/posts/gitment-introduction/
 gitment:
-  enable: false
-  mint: true # RECOMMEND, A mint on Gitment, to support count, language and pro                                                                                                              xy_gateway
-  count: true # Show comments count in post meta area
-  lazy: false # Comments lazy loading with a button
-  cleanly: false # Hide 'Powered by ...' on footer, and more
-  language: # Force language, or auto switch by theme
-  github_user: # MUST HAVE, Your Github Username
-  github_repo: # MUST HAVE, The name of the repo you use to store Gitment comme                                                                                                              nts
-  client_id: # MUST HAVE, Github client id for the Gitment
-  client_secret: # EITHER this or proxy_gateway, Github access secret token for                                                                                                               the Gitment
-  proxy_gateway: # Address of api proxy, See: https://github.com/aimingoo/inter                                                                                                              sect
-  redirect_protocol: # Protocol of redirect_uri with force_redirect_protocol wh                                                                                                              en mint enabled
+  enable: true
+  # RECOMMEND, A mint on Gitment, to support count, language and proxy_gateway
+  mint: true
+  # Show comments count in post meta area
+  count: true
+  # Comments lazy loading with a button
+  lazy: false
+  # Hide 'Powered by ...' on footer, and more
+  cleanly: true
+  # Force language, or auto switch by theme
+  language:
:
-  cleanly: false # Hide 'Powered by ...' on footer, and more
-  language: # Force language, or auto switch by theme
-  github_user: # MUST HAVE, Your Github Username
-  github_repo: # MUST HAVE, The name of the repo you use to store Gitment comments
-  client_id: # MUST HAVE, Github client id for the Gitment
-  client_secret: # EITHER this or proxy_gateway, Github access secret token for the Gitment
-  proxy_gateway: # Address of api proxy, See: https://github.com/aimingoo/intersect
-  redirect_protocol: # Protocol of redirect_uri with force_redirect_protocol when mint enabled
+  enable: true
+  # RECOMMEND, A mint on Gitment, to support count, language and proxy_gateway
+  mint: true
+  # Show comments count in post meta area
+  count: true
+  # Comments lazy loading with a button
+  lazy: false
+  # Hide 'Powered by ...' on footer, and more
+  cleanly: true
+  # Force language, or auto switch by theme
+  language:
+  # MUST HAVE, Your Github Username
+  github_user: mz8023yt
+  # MUST HAVE, The name of the repo you use to store Gitment comments
+  github_repo: blog.comments
+  # MUST HAVE, Github client id for the Gitment
+  client_id: 3b0f546a740181c255a9
+  # EITHER this or proxy_gateway, Github access secret token for the Gitment
+  client_secret: 41b269fc8cc71a77e91b3485769b325e249b15dd
+  # Address of api proxy, See: https://github.com/aimingoo/intersect
+  proxy_gateway:
+  # Protocol of redirect_uri with force_redirect_protocol when mint enabled
+  redirect_protocol:

 # Baidu Share
 # Available value:
```


#### 添加站内搜索

```
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index 55a78f9..4b0a05f 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -713,12 +713,12 @@ algolia_search:
 # Local search
 # Dependencies: https://github.com/theme-next/hexo-generator-searchdb
 local_search:
-  enable: false
+  enable: ture
   # if auto, trigger search by changing input
   # if manual, trigger search by pressing enter key or search button
   trigger: auto
   # show top n results per article, show all results by setting to -1
-  top_n_per_article: 1
+  top_n_per_article: 3
   # unescape html strings to the readable one
   unescape: false
```

```
diff --git a/_config.yml b/_config.yml
index 025ef2d..b863a15 100644
--- a/_config.yml
+++ b/_config.yml
@@ -10,6 +10,13 @@ author: Paul Wang
 language: zh-CN
 timezone:

+# Search
+search:
+  path: search.xml
+  field: post
+  format: html
+  limit: 10000
+
 # Word Count
 symbols_count_time:
   symbols: true
```

#### 添加在线歌单

        hexo new page music

```
diff --git a/source/music/index.md b/source/music/index.md
new file mode 100644
index 0000000..07bf7ce
--- /dev/null
+++ b/source/music/index.md
@@ -0,0 +1,13 @@
+---
+title: '[Maziot] 网易云歌单'
+date: 2018-04-22 18:52:56
+---
+
+<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/aplayer@1.7/dist/APlayer.min.css">
+<script src="https://cdn.jsdelivr.net/npm/aplayer@1.7/dist/APlayer.min.js"></script>
+<div class="aplayer"
+    data-id="821903731"
+    data-server="netease"
+    data-type="playlist">
+</div>
+<script src="https://cdn.jsdelivr.net/npm/meting@1.1/dist/Meting.min.js"></script>
diff --git a/themes/next/_config.yml b/themes/next/_config.yml
index c751ba3..e44e5ec 100755
--- a/themes/next/_config.yml
+++ b/themes/next/_config.yml
@@ -122,6 +122,7 @@ menu:
   categories: /categories/ || th
   tags: /tags/ || tags
   about: /about/ || user
+  music: /music/ || music
   #schedule: /schedule/ || calendar
   #sitemap: /sitemap.xml || sitemap
   #commonweal: /404/ || heartbeat
diff --git a/themes/next/languages/zh-CN.yml b/themes/next/languages/zh-CN.yml
index 8b73150..db73de1 100644
--- a/themes/next/languages/zh-CN.yml
+++ b/themes/next/languages/zh-CN.yml
@@ -14,6 +14,7 @@ menu:
   search: 搜索
   schedule: 日程表
   sitemap: 站点地图
+  music: 音乐
   commonweal: 公益 404
 sidebar:
   overview: 站点概览
```


