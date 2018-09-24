---
title: '[Tiny4412] 实现自己的 uboot 命令'
date: 2018-09-24 09:41:15
tags:
  - uboot
  - tiny4412
  - FrindlyARM
categories: Tiny4412
---

### uboot 命令介绍

Tiny 4412 开机倒数3秒计时的时候，按下 enter 键将进入 uboot 终端。  
输入 `?` 或者 `help` 可以查看 uboot 支持的所有命令

    TINY4412 # ?
    ?       - alias for 'help'
    base    - print or set address offset
    bdinfo  - print Board Info structure
    boot    - boot default, i.e., run 'bootcmd'
    bootd   - boot default, i.e., run 'bootcmd'
    bootelf - Boot from an ELF image in memory
    bootm   - boot application image from memory
    bootp   - boot image via network using BOOTP/TFTP protocol
    bootvx  - Boot vxWorks from an ELF image
    chpart  - change active partition
    cmp     - memory compare
    coninfo - print console devices and information
    cp      - memory copy
    crc32   - checksum calculation
    dcache  - enable or disable data cache
    dnw     - dnw     - initialize USB device and ready to receive for Windows server (specific)

    echo    - echo args to console
    editenv - edit environment variable
    emmc    - Open/Close eMMC boot Partition
    env     - environment handling commands
    exit    - exit script
    ext2format- ext2format - disk format by ext2

    ext2load- load binary file from a Ext2 filesystem
    ext2ls  - list files in a directory (default /)
    ext3format- ext3format - disk format by ext3

    false   - do nothing, unsuccessfully
    fastboot- fastboot- use USB Fastboot protocol

    fatformat- fatformat - disk format by FAT32

    fatinfo - fatinfo - print information about filesystem
    fatload - fatload - load binary file from a dos filesystem

    fatls   - list files in a directory (default /)
    fdisk   - fdisk for sd/mmc.

    go      - start application at address 'addr'
    help    - print command description/usage
    icache  - enable or disable instruction cache
    iminfo  - print header information for application image
    imxtract- extract a part of a multi-image
    itest   - return true/false on integer compare
    loadb   - load binary file over serial line (kermit mode)
    loads   - load S-Record file over serial line
    loady   - load binary file over serial line (ymodem mode)
    loop    - infinite loop on address range
    md      - memory display
    mm      - memory modify (auto-incrementing address)
    mmc     - MMC sub system
    mmcinfo - mmcinfo <dev num>-- display MMC info
    movi    - movi	- sd/mmc r/w sub system for SMDK board
    mtdparts- define flash/nand partitions
    mtest   - simple RAM read/write test
    mw      - memory write (fill)
    nfs     - boot image via network using NFS protocol
    nm      - memory modify (constant address)
    ping    - send ICMP ECHO_REQUEST to network host
    printenv- print environment variables
    reginfo - print register information
    reset   - Perform RESET of the CPU
    run     - run commands in an environment variable
    saveenv - save environment variables to persistent storage
    setenv  - set environment variables
    showvar - print local hushshell variables
    sleep   - delay execution for some time
    source  - run script from memory
    test    - minimal test like /bin/sh
    tftpboot- boot image via network using TFTP protocol
    true    - do nothing, successfully
    usb     - USB sub-system
    version - print monitor version

uboot 支持很多命令，命令对应的源文件都在 `common` 目录下

    user@vmware:~/tiny4412/FriendlyARM.uboot-2010.12/common$ ls
    bedbug.c                 cmd_exit.o      cmd_mii.c        cmd_source.c       dlmalloc.c       kgdb.c
    cmd_ambapp.c             cmd_ext2.c      cmd_misc.c       cmd_source.o       dlmalloc.o       kgdb_stubs.c
    cmd_bdinfo.c             cmd_ext2.o      cmd_misc.o       cmd_spibootldr.c   dlmalloc.src     lcd.c
    cmd_bdinfo.o             cmd_fastboot.c  cmd_mmc.c        cmd_spi.c          env_auto.c       libcommon.o
    cmd_bedbug.c             cmd_fastboot.o  cmd_mmc_fdisk.c  cmd_strings.c      env_auto.o       lynxkdi.c
    cmd_bmp.c                cmd_fat.c       cmd_mmc_fdisk.o  cmd_terminal.c     env_common.c     main.c
    cmd_boot.c               cmd_fat.o       cmd_mmc.o        cmd_test.c         env_common.o     main.o
    cmd_bootldr.c            cmd_fdc.c       cmd_movi.c       cmd_test.o         env_dataflash.c  Makefile
    cmd_bootm.c              cmd_fdos.c      cmd_movi.o       cmd_tsi148.c       env_eeprom.c     memsize.c
    cmd_bootm.o              cmd_fdt.c       cmd_mp.c         cmd_ubi.c          env_embedded.c   memsize.o
    cmd_boot.o               cmd_flash.c     cmd_mtdparts.c   cmd_ubifs.c        env_flash.c      miiphyutil.c
    cmd_cache.c              cmd_fpga.c      cmd_mtdparts.o   cmd_universe.c     env_mgdisk.c     modem.c
    cmd_cache.o              cmd_help.c      cmd_nand.c       cmd_usb.c          env_mmc.c        serial.c
    cmd_console.c            cmd_help.o      cmd_net.c        cmd_usbd3.c        env_nand.c       serial.o
    cmd_console.o            cmd_i2c.c       cmd_net.o        cmd_usbd.c         env_nowhere.c    s_record.c
    cmd_cplbinfo.c           cmd_ide.c       cmd_nvedit.c     cmd_usbd.o         env_nvram.c      s_record.o
    cmd_cramfs.c             cmd_immap.c     cmd_nvedit.o     cmd_usb.o          env_onenand.c    stdio.c
    cmd_dataflash_mmc_mux.c  cmd_irq.c       cmd_onenand.c    cmd_version.c      env_sf.c         stdio.o
    cmd_date.c               cmd_itest.c     cmd_otp.c        cmd_version.o      exports.c        system_map.c
    cmd_dcr.c                cmd_itest.o     cmd_pci.c        cmd_vfd.c          exports.o        update.c
    cmd_df.c                 cmd_jffs2.c     cmd_pcmcia.c     cmd_ximg.c         fdt_support.c    usb.c
    cmd_diag.c               cmd_license.c   cmd_pcmcia.o     cmd_ximg.o         flash.c          usb_kbd.c
    cmd_display.c            cmd_load.c      cmd_portio.c     cmd_yaffs2.c       flash.o          usb.o
    cmd_dtt.c                cmd_load.o      cmd_reginfo.c    command.c          hush.c           usb_storage.c
    cmd_echo.c               cmd_log.c       cmd_reginfo.o    command.o          hush.o           xyzModem.c
    cmd_echo.o               cmd_mac.c       cmd_reiser.c     console.c          hwconfig.c       xyzModem.o
    cmd_eeprom.c             cmd_mem.c       cmd_sata.c       console.o          image.c
    cmd_elf.c                cmd_mem.o       cmd_scsi.c       ddr_spd.c          image.o
    cmd_elf.o                cmd_mfsl.c      cmd_setexpr.c    decompress_ext4.c  iomux.c
    cmd_exit.c               cmd_mgdisk.c    cmd_sf.c         decompress_ext4.o  kallsyms.c

每一个命令都由 U_BOOT_CMD 宏描述，宏定义的位置在 `include/command.h` 文件中，感兴趣的小伙伴可以去研究下。

    user@vmware:~/tiny4412/FriendlyARM.uboot-2010.12$ grep -rsnE " U_BOOT_CMD("
    include/command.h:112:#define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \
    include/command.h:120:#define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \
    doc/README.commands:5:Then using the U_BOOT_CMD() macro to fill in a cmd_tbl_t struct.

### 二 分析 help 命令

这里我们分析下 help 命令来了解 uboot 命令的结构。  
`common/cmd_help.c` 文件内容如下：

    #include <common.h>
    #include <command.h>

    extern cmd_tbl_t  __u_boot_cmd_bdinfo;
    extern cmd_tbl_t  __u_boot_cmd_showvar;

    int do_help(cmd_tbl_t * cmdtp, int flag, int argc, char * const argv[])
    {
        return _do_help(&__u_boot_cmd_bdinfo,
                &__u_boot_cmd_showvar - &__u_boot_cmd_bdinfo + 1,
                cmdtp, flag, argc, argv);
    }

    U_BOOT_CMD(
        help, CONFIG_SYS_MAXARGS, 1, do_help,
        "print command description/usage",
        "\n"
        " - print brief description of all commands\n"
        "help command ...\n"
        " - print detailed usage of 'command'"
    );

    /* This does not use the U_BOOT_CMD macro as ? can't be used in symbol names */
    cmd_tbl_t __u_boot_cmd_question_mark Struct_Section = {
        "?", CONFIG_SYS_MAXARGS, 1, do_help,
        "alias for 'help'",
    #ifdef  CONFIG_SYS_LONGHELP
        ""
    #endif /* CONFIG_SYS_LONGHELP */
    };


### 三 编写一个自定义的 hello 命令

从 help 命令的实现，猜测实现一个新的 uboot cmd 的步骤为：

1. 在 common 目录下创建 cmd_hello.c 文件
2. 包含头文件

       #include <common.h>
       #include <command.h>

3. 使用 U_BOOT_CMD 定义一个命令

       U_BOOT_CMD
       (
           <命令的名称>,
           <最大参数个数>,	
           <重复次数>,	
           <命令底层执行的回调函数>,
           <说明命令的功能>,
           <说明命令的详细用法>
       );

4. 实现回调函数  
   函数原型为

       /**
        * @param cmdtp 描述命令的结构体，保存了 U_BOOT_CMD
        * @param flag 
        * @param argc 命令传入的参数的个数
        * @param argv 命令传入的参数
        */
       int do_help(cmd_tbl_t * cmdtp, int flag, int argc, char * const argv[])

### 四 编译 uboot 命令

1. 将实现的 cmd_hello.c 放到 common 目录下
2. 修改 Makefile，将 cmd_hello.c 包含到到编译系统中

       user@vmware:~/tiny4412/FriendlyARM.uboot-2010.12$ git diff
       diff --git a/common/Makefile b/common/Makefile
       old mode 100644
       new mode 100755
       index bfa39d5..92b9a66
       --- a/common/Makefile
       +++ b/common/Makefile
       @@ -48,6 +48,7 @@ COBJS-y += cmd_bootm.o
        COBJS-y += cmd_help.o
        COBJS-y += cmd_nvedit.o
        COBJS-y += cmd_version.o
       +COBJS-y += cmd_hello.o
        
        # environment
        COBJS-y += env_common.o

3. 重新编译
4. 下载验证

       TINY4412 # hello xxx
       hello - this is a demo command

       Usage:
       hello 
           - this is a demo long help

       TINY4412 # hello
       hello uboot



