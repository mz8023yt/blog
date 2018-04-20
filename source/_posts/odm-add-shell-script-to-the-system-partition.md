---
title: '[ODM] 在 system 只读分区新增 shell 脚本'
date: 2018-03-05 17:22:08
tags:
  - shell
  - system
  - mtk
categories: ODM
---

### 问题来源

电力达人需求需要动态的控制 AAL 的 CABC。  
客户需求，电力达人切换手机工作模式的时候会自动调用 /system/etc/ 目录下的脚本文件，而这些脚本需要我们实现。

### 解决思路

1. 参考指纹 ca ta 的拷贝，将文件拷贝到 src/out/ 下目标目录中。
2. 参考指纹赋权限的方式，在 init.rc 中追加 sh 脚本的权限。

### 实现方式

#### 第一步：将脚本放入代码工程中

随便放在哪个目录里面都是可以的。

```
alps\kernel-3.18\drivers\misc\mediatek\video\common\aal20\aal_shell\pwr-balance.sh
alps\kernel-3.18\drivers\misc\mediatek\video\common\aal20\aal_shell\pwr-normal.sh
alps\kernel-3.18\drivers\misc\mediatek\video\common\aal20\aal_shell\pwr-ultra.sh
```

#### 第二步：修改 device.mk 文件, 将脚本拷贝到 /system/etc 目录下

在 alps\device\ginreen\E262L\device.mk 追加:

```bash
PRODUCT_COPY_FILES += kernel-3.18/drivers/misc/mediatek/video/common/aal20/aal_shell/pwr-normal.sh:system/etc/pwr-normal.sh
PRODUCT_COPY_FILES += kernel-3.18/drivers/misc/mediatek/video/common/aal20/aal_shell/pwr-balance.sh:system/etc/pwr-balance.sh
PRODUCT_COPY_FILES += kernel-3.18/drivers/misc/mediatek/video/common/aal20/aal_shell/pwr-ultra.sh:system/etc/pwr-ultra.sh
```

源文件于目标文件通过冒号分隔, 冒号前是源文件, 冒号后是目标文件。  
源文件默认根目录是 alps 目录, 目标文件默认根目录是 alps\out\target\product\E262L 目录。

#### 第三步：修改 init.mt6755.rc 文件, 给脚本设置执行权限

可以修改以下这几个文件，这里我是修改的 init.mt6755.rc 文件。

```
alps\device\mediatek\mt6755\init.mt6755.rc
alps\device\ginreen\E262L\init.project.rc
alsp\system\core\rootdir\init.rc
```

在 alps\device\mediatek\mt6755\init.mt6755.rc 文件中追加一项：

```bash
on init
    chmod 0755 /system/etc/pwr-balance.sh
    chmod 0755 /system/etc/pwr-normal.sh
    chmod 0755 /system/etc/pwr-ultra.sh
```

发现文件成功拷贝到了 /system/etc/ 目录下，但是赋的权限并没有生效。  
说明直接参考指纹赋权限的方式不行。接下来找原因，参考 online 的文档，找到原因，是因为 /system/ 分区是只读分区，分区挂载后，就没有修改文件的能力，包括改权限都改不了。  
需要在文件系统挂载之前，先挂在为 rw 的权限，修改好脚本的权限后再重新挂在为 ro。就像这样操作：

```bash
on init
    mount ext4 /dev/block/platform/mtk-msdc.0/11230000.msdc0/by-name/system /system rw wait
    chmod 0755 /system/etc/pwr-balance.sh
    chmod 0755 /system/etc/pwr-normal.sh
    chmod 0755 /system/etc/pwr-ultra.sh
    unmount /system
```

但是还是发现权限没有成功赋上。后续提交 case 给 MTK 无果。
