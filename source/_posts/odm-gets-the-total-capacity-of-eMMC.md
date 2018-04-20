---
title: '[ODM] 获取 eMMC 总容量需求实现'
date: 2018-03-26 22:37:18
tags:
  - eMMC
  - sys
categories: ODM
---


### 问题描述

现时代的安卓手机厂商都喜欢自己克客制化 Android 系统，做一套具有品牌特点的 UI 界面。我们客户也不例外，由于客户克制化了 UI 界面后，在 setting 中显示手机存储容量的界面上，需要读取 /data/data/emmc_total_size 节点获取 eMMC(Flash) 的大小。因此驱动这边需要实现 /data/data/emmc_total_size 这个节点，将 eMMC 大小传递到上层。

### 功能实现

kernel 中的需要创建 sys 节点，这里修改了 mmc.c 和 card.h 两个文件，具体修改的 diff 如下：

```c
diff --git a/drivers/mmc/core/mmc.c b/drivers/mmc/core/mmc.c
index dc2e206..b9d567a 100755
--- a/drivers/mmc/core/mmc.c
+++ b/drivers/mmc/core/mmc.c
@@ -591,7 +591,9 @@ static int mmc_read_ext_csd(struct mmc_card *card，u8 *ext_csd)
 {
 	int err = 0，idx;
 	unsigned int part_size;
+// mz8023yt@163.com 20180324 begin >>> [1/6] add the mmc total size debug node
+	unsigned int size = 0;
+// mz8023yt@163.com 20180324 end   <<< [1/6] add the mmc total size debug node
 	BUG_ON(!card);
 
 	if (!ext_csd)
@@ -636,6 +638,14 @@ static int mmc_read_ext_csd(struct mmc_card *card，u8 *ext_csd)
 		/* Cards with density > 2GiB are sector addressed */
 		if (card->ext_csd.sectors > (2u * 1024 * 1024 * 1024) / 512)
 			mmc_card_set_blockaddr(card);
+// mz8023yt@163.com 20180324 begin >>> [1/6] add the mmc total size debug node 
+		size = card->ext_csd.sectors >> 20;
+		card->mmc_total_size = 1;
+		while ((size > 0) && (size >> 1) > 0) {
+			size = size >> 1;
+			card->mmc_total_size = card->mmc_total_size * 2;
+		}
+// mz8023yt@163.com 20180324 end   <<< [1/6] add the mmc total size debug node
 	}
 
 	card->ext_csd.raw_card_type = ext_csd[EXT_CSD_CARD_TYPE];
@@ -986,6 +996,14 @@ out:
 	return err;
 }
 
+// mz8023yt@163.com 20180324 begin >>> [2/6] add the mmc total size debug node
+static unsigned int get_emmc_total_size(struct mmc_card *card)
+{
+	BUG_ON(!card);
+	return card->mmc_total_size;
+}
+// mz8023yt@163.com 20180324 end   <<< [2/6] add the mmc total size debug node

 MMC_DEV_ATTR(cid，"%08x%08x%08x%08x\n"，card->raw_cid[0]，card->raw_cid[1],
 	card->raw_cid[2]，card->raw_cid[3]);
 MMC_DEV_ATTR(csd，"%08x%08x%08x%08x\n"，card->raw_csd[0]，card->raw_csd[1],
@@ -1013,6 +1031,10 @@ MMC_DEV_ATTR(enhanced_rpmb_supported，"%#x\n",
 		card->ext_csd.enhanced_rpmb_supported);
 MMC_DEV_ATTR(rel_sectors，"%#x\n"，card->ext_csd.rel_sectors);
 
+// mz8023yt@163.com 20180324 begin >>> [3/6] add the mmc total size debug node
+MMC_DEV_ATTR(emmc_total_size，"%d\n"，get_emmc_total_size(card));
+// mz8023yt@163.com 20180324 end   <<< [3/6] add the mmc total size debug node

 static struct attribute *mmc_std_attrs[] = {
 	&dev_attr_cid.attr,
 	&dev_attr_csd.attr,
@@ -1034,6 +1056,9 @@ static struct attribute *mmc_std_attrs[] = {
 	&dev_attr_raw_rpmb_size_mult.attr,
 	&dev_attr_enhanced_rpmb_supported.attr,
 	&dev_attr_rel_sectors.attr,
+// mz8023yt@163.com 20180324 begin >>> [4/6] add the mmc total size debug node
+	&dev_attr_emmc_total_size.attr,
+// mz8023yt@163.com 20180324 end   <<< [4/6] add the mmc total size debug node
 	NULL,
 };
 ATTRIBUTE_GROUPS(mmc_std);

================================================================================

diff --git a/include/linux/mmc/card.h b/include/linux/mmc/card.h
old mode 100644
new mode 100755
index f6cec9c..8dc3599
--- a/include/linux/mmc/card.h
+++ b/include/linux/mmc/card.h
@@ -431,6 +431,10 @@ struct mmc_card {
 	u8 *cached_ext_csd;
 	bool cmdq_init;
 	struct mmc_bkops_info bkops;
+
+// mz8023yt@163.com 20180324 begin >>> [5/6] add the mmc total size debug node
+	unsigned int mmc_total_size;
+// mz8023yt@163.com 20180324 end   <<< [5/6] add the mmc total size debug node
 };
```

sys 节点创建好了，接下来创建到 /data/data/ 目录下的链接文件。  
创建链接文件需要修改 device 目录下的 init 配置文件。

```bash
diff --git a/project_name/init.target.rc b/project_name/init.target.rc
index 45902de..e3a3f08 100755
--- a/project_name/init.target.rc
+++ b/project_name/init.target.rc
@@ -87,6 +87,10 @@ on post-fs-data
     mkdir /persist/data/tz 0700 system system
     mkdir /data/vendor/hbtp 0750 system system
     mkdir /data/misc/dts 0770 media audio
+# mz8023yt@163.com 20180326 begin >>> [6/6] add the mmc total size debug node
+    symlink /sys/devices/soc/7824900.sdhci/mmc_host/mmc0/mmc0:0001/emmc_total_size /data/data/emmc_total_size
+    chmod 0777 /data/data/emmc_total_size
+# mz8023yt@163.com 20180326 end   <<< [6/6] add the mmc total size debug node
 
 # add by qiancheng 20171128 start
     setprop persist.sys.fp.subid 0
```

### 如何验证

找到一台 16+2 的手机，先看看 sys 下的节点功能是否生效。  
找不到节点的话，可以先 find 下节点名，找到节点路径。

```bash
C:\Users\wangbing>adb shell
android:/ # cd sys
android:/sys # find -name "emmc_total_size"
./devices/soc/7824900.sdhci/mmc_host/mmc0/mmc0:0001/emmc_total_size
android:/sys # cat devices/soc/7824900.sdhci/mmc_host/mmc0/mmc0:0001/emmc_total_size
16
```

找到路径后下次直接 cat 就可以了。

```bash
C:\Users\wangbing\Desktop\unlock>adb shell cat /sys/devices/soc/7824900.sdhci/mmc_host/mmc0/mmc0:0001/emmc_total_size
16
```

sys 节点已经生效了，再看看到 /data/data 的链接文件有没有功能。

```bash
C:\Users\wangbing\Desktop\unlock>adb shell cat /data/data/emmc_total_size
16

C:\Users\wangbing\Desktop\unlock>adb shell ls -l /data/data/emmc_total_size
lrwxrwxrwx 1 root root 70 1970-01-01 02:27 /data/data/emmc_total_size -> /sys/devices/soc/7824900.sdhci/mmc_host/mmc0/mmc0:0001/emmc_total_size
```

### 核心模块

查看代码前后逻辑，发现这里计算 ROM 容量的大小是通过计算 eMMC 的 sector(扇区) 的个数乘以 sector 的大小计算到的总容量。  
EMMC 中的 ROM 由扇区(sector)组成，扇区有 512Byte 和 4KB 两种大小。

1. 具体的扇区的大小是记录在 Extended CSD register 的 ExtCSD[61] 中.
2. 拥有的扇区的个数是记录在 Extended CSD register 的 ExtCSD[215:212] 中.

详细的信息请查阅 《JESD84-B51》 协议手册 7.4 小节。

相关代码如下:

```c
#define EXT_CSD_DATA_SECTOR_SIZE        61      /* R */
#define EXT_CSD_SEC_CNT                 212     /* RO，4 bytes */

static int mmc_read_ext_csd(struct mmc_card *card，u8 *ext_csd) {
        ... ...
        /*
         * The EXT_CSD format is meant to be forward compatible. As long
         * as CSD_STRUCTURE does not change，all values for EXT_CSD_REV
         * are authorized，see JEDEC JESD84-B50 section B.8.
         */
        // 读取 emmc 协议版本号
        card->ext_csd.rev = ext_csd[EXT_CSD_REV];

        if (card->ext_csd.rev >= 2) {
                // 读取 sector 的个数，保存在 card->ext_csd.sectors 中
                card->ext_csd.sectors = ext_csd[EXT_CSD_SEC_CNT + 0] << 0 |
                                        ext_csd[EXT_CSD_SEC_CNT + 1] << 8 |
                                        ext_csd[EXT_CSD_SEC_CNT + 2] << 16 |
                                        ext_csd[EXT_CSD_SEC_CNT + 3] << 24;

                /**
                 * 这里根据 sector 个数计算 ROM 大小，并换算成 GB 单位
                 * 一个 sector 的大小为 512Byte = 2048(512*8)bit
                 * 以 16G 为例，先看看 16G 有几个 sector
                 * 理论上有 16G = 16*1024M = 16*1024*1024K = 16*1024*1024*1024B = 16*1024*1024*2 sector
                 * 将这个值 16*1024*1024*2 右移 20 位后得出 16*2=32
                 * 但实际上，没有这么多扇区，看后面 log 上打印出来的 size = 29
                 */
                size = card->ext_csd.sectors >> 20;
                card->mmc_total_size = 1;
                while ((size > 0) && (size >> 1) > 0) {
                        size = size >> 1;
                        card->mmc_total_size = card->mmc_total_size * 2;
                }

                // [    6.691712] [Paul][EMMC] size = 29
                // [    6.691714] [Paul][EMMC] mmc_total_size = 16
+               printk("[Paul][EMMC] size = %d\n"，size);
+               printk("[Paul][EMMC] mmc_total_size = %d"，card->mmc_total_size);

        ... ...
        /* eMMC v4.5 or later */
        if (card->ext_csd.rev >= 6) {

                // [    6.691726] [Paul][EMMC] ext_csd[EXT_CSD_DATA_SECTOR_SIZE] = 0
+               printk("[Paul][EMMC] ext_csd[EXT_CSD_DATA_SECTOR_SIZE] = %d"，ext_csd[EXT_CSD_DATA_SECTOR_SIZE]);

                // 这里根据 EXT_CSD_DATA_SECTOR_SIZE 确定一个扇区的大小
                // 上面 log 说明扇区大小为 512Byte
                if (ext_csd[EXT_CSD_DATA_SECTOR_SIZE] == 1)
                        card->ext_csd.data_sector_size = 4096;
                else
                        card->ext_csd.data_sector_size = 512;
        }
}
```

总结下就是:
读取 eMMC Extended CSD 寄存器中的第 61 位获取到扇区大小，这里获取到的是 512KB/扇区。
读取 eMMC Extended CSD 寄存器中的第 212~215 这四位获取到 eMMC 拥有的扇区数目。
然后使用扇区大小乘以扇区数目，得到 ROM 容量，最后通过移位操作将得到的扇区容量换算成 GB 为单位的大小值。  
最后在 sys 节点中将这个计算出的 GB 为单位的大小值传递给上层。

### 扩展思考

上面仅仅是实现了从 emmc 内部动态的读取出 ROM 的容量，那么如果我们要动态获取到 RAM 容量要怎么操作?

思路1: 通过读取 /proc/meminfo 节点，获取到 MemTotal 的大小对应的字符串(获取前27个字符)，通过解析这段字符串，得到 MemTotal 大小，再换算成 GB 单位即可。

```
C:\Users\wangbing>adb shell cat /proc/meminfo
MemTotal:        1853224 kB
MemFree:           28664 kB
MemAvailable:     798400 kB
Buffers:           30224 kB
Cached:           837224 kB
SwapCached:         5068 kB
Active:           808916 kB
Inactive:         480956 kB
Active(anon):     387560 kB
Inactive(anon):   131520 kB
Active(file):     421356 kB
Inactive(file):   349436 kB
Unevictable:      105380 kB
Mlocked:          105380 kB
SwapTotal:        633748 kB
SwapFree:         556328 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:        527880 kB
Mapped:           437540 kB
Shmem:             13324 kB
Slab:             141104 kB
SReclaimable:      53496 kB
SUnreclaim:        87608 kB
KernelStack:       32016 kB
PageTables:        41228 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     1560360 kB
Committed_AS:   89069992 kB
VmallocTotal:   244318144 kB
VmallocUsed:      182136 kB
VmallocChunk:   243976548 kB
```
