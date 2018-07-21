---
title: '[Mini2440] 如何添加 u-boot 命令'
date: 2018-07-20 22:02:16
tags:
  - mini2440
  - u-boot
categories: Mini2440
---

看书总是介绍说，使用 `U_BOOT_CMD` 宏可以定义一个命令，但是并没有仔细研究过，这里在卫东山老师的 u-boot-1.1.6 源码基础上简单分析下。  
搜索下，看看这个宏是什么东西：

    user@vmware:~/mini2440/u-boot-1.1.6$ grep -rsn " U_BOOT_CMD" ./
    ./common/command.c:329:/* This do not ust the U_BOOT_CMD macro as ? can't be used in symbol names */
    ./include/command.h:97:#define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \
    ./include/command.h:102:#define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \
    ./doc/README.commands:5:Then using the U_BOOT_CMD() macro to fill in a cmd_tbl_t struct.

好家伙，居然有文档，那看看 ./doc/README.commands 文档怎么说：  
文档很简洁，原文如下：

> Commands are added to U-Boot by creating a new command structure.
> This is done by first including command.h
> 
> Then using the U_BOOT_CMD() macro to fill in a cmd_tbl_t struct.
> 
> U_BOOT_CMD(name,maxargs,repeatable,command,"usage","help")
> 
> name:	 is the name of the commad. THIS IS NOT a string.
> maxargs: the maximumn numbers of arguments this function takes
> command: Function pointer (*cmd)(struct cmd_tbl_s *, int, int, char *[]);
> usage:	 Short description. This is a string
> help:	 long description. This is a string
> 
> 
> **** Behinde the scene ******
> 
> The structure created is named with a special prefix (__u_boot_cmd_)
> and placed by the linker in a special section.
> 
> This makes it possible for the final link to extract all commands
> compiled into any object code and construct a static array so the
> command can be found in an array starting at __u_boot_cmd_start.
> 
> If a new board is defined do not forget to define the command section
> by writing in u-boot.lds ($(TOPDIR)/board/boardname/u-boot.lds) these
> 3 lines:
> 
> __u_boot_cmd_start = .;
> .u_boot_cmd : { *(.u_boot_cmd) }
> __u_boot_cmd_end = .;

简单翻译下：

> 添加一个命令本质其实就是添加一个命令的结构体
> 第一件要做的事情就是包含 command.h 头文件
> 然后使用 U_BOOT_CMD 宏来填充 cmd_tbl_t 命令结构体，U_BOOT_CMD 宏的原型如下：
> 
> U_BOOT_CMD(name,maxargs,repeatable,command,"usage","help")
> name: 命令的名称，注意，不是字符串，不要用双引号
> maxargs: 这个函数的参数的最大数量
> command: 回调函数的函数指针 (*cmd)(struct cmd_tbl_s *, int, int, char *[]);
> usage: 简述该命令的用法 字符串
> help: 详细描述改名的用法 字符串
> 
> 使用该宏所创建的结构都使用同一个前缀命名，该前缀为 (__u_boot_cmd_)。并且，这些结构由链接器放置在一个特殊的区域。
> 这使得最后这些结构被静态的连接成为了一个数组，通过 __u_boot_cmd_start 可以访问的到。
> 
> __u_boot_cmd_start = .;
> .u_boot_cmd : { *(.u_boot_cmd) }
> __u_boot_cmd_end = .;

将 `U_BOOT_CMD` 展开看看：

    user@vmware:~/mini2440/u-boot-1.1.6$ grep -rsn " U_BOOT_CMD" ./
    ./common/command.c:329:/* This do not ust the U_BOOT_CMD macro as ? can't be used in symbol names */
    ./include/command.h:97:#define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \
    ./include/command.h:102:#define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \
    ./doc/README.commands:5:Then using the U_BOOT_CMD() macro to fill in a cmd_tbl_t struct.

看看这两行中是如何定义的：

    ./include/command.h:97:#define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \
    ./include/command.h:102:#define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \

打开 `command.h` 文件看看：

    93  #define Struct_Section  __attribute__ ((unused,section (".u_boot_cmd")))
    94
    95  #ifdef  CFG_LONGHELP
    96  
    97  #define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \
    98      cmd_tbl_t __u_boot_cmd_##name Struct_Section = {#name, maxargs, rep, cmd, usage, help}
    99  
    100 #else	/* no long help info */
    101 
    102 #define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \
    103     cmd_tbl_t __u_boot_cmd_##name Struct_Section = {#name, maxargs, rep, cmd, usage}
    104 
    105 #endif	/* CFG_LONGHELP */

两个宏的作用是一致的，只是一个用详细的 help 说明，一个省略了而已。  
另外，看 93 行，可以发现，`Struct_Section` 也是一个宏，用于指定链接的存放的位置。  
随便找一个命令代入，将宏展开后看看，不妨就以经常使用的 `printenv` 代入看看吧

    user@vmware:~/mini2440/u-boot-1.1.6$ grep -rsn "printenv," ./
    ./common/cmd_nvedit.c:580:	printenv, CFG_MAXARGS, 1,	do_printenv,
    ./doc/README.cmi:71:* U-Boot commands: go, loads, loadb, all memory features, printenv,

打开 `cmd_nvedit.c` 文件看看：

    579 U_BOOT_CMD(
    580 	printenv, CFG_MAXARGS, 1,	do_printenv,
    581 	"printenv- print environment variables\n",
    582 	"\n    - print values of all environment variables\n"
    583 	"printenv name ...\n"
    584 	"    - print value of environment variable 'name'\n"
    585 );

将 `U_BOOT_CMD` 和 `Struct_Section` 展开后看看：

    cmd_tbl_t __u_boot_cmd_printenv __attribute__ ((unused,section (".u_boot_cmd"))) =
    {
        printenv,
        16,
        1,
        do_printenv,
        "printenv- print environment variables\n",
        "printenv name ...\n    - print value of environment variable 'name'\n"
    }

这里就是定义了一个 `cmd_tbl_t` 的结构体对象，并将它们存到了 `.u_boot_cmd` 段中。  
注释和文档已经说的很清楚了，`cmd_tbl_t` 不多扯了，这里还是贴出来吧：

    typedef struct cmd_tbl_s	cmd_tbl_t;
    
    struct cmd_tbl_s {
            char            *name;          /* Command Name                     */
            int             maxargs;        /* maximum number of arguments      */
            int             repeatable;     /* autorepeat allowed?              */
                                            /* Implementation function          */
            int             (*cmd)(struct cmd_tbl_s *, int, int, char *[]);
            char            *usage;         /* Usage message (short)            */
    #ifdef CFG_LONGHELP
            char            *help;          /* Help  message (long)             */
    #endif
    #ifdef CONFIG_AUTO_COMPLETE
            /* do auto completion on the arguments */
            int             (*complete)(int argc, char *argv[], char last_char, int maxv, char *cmdv[]);
    #endif
    };

上面是添加命令的方式，那么命令添加到 `u_boot_cmd` 后，谁会去访问？谁来用？  
文档中说链接脚本中有这样的描述，我们当然相信，不过最好是自己确认下：

    	__u_boot_cmd_start = .;
    	.u_boot_cmd : { *(.u_boot_cmd) }
    	__u_boot_cmd_end = .;

去 100ask 的板级信息下确认下：

    user@vmware:~/mini2440/u-boot-1.1.6$ cd board/100ask24x0/
    user@vmware:~/mini2440/u-boot-1.1.6/board/100ask24x0$ grep -rsn "__u_boot_cmd_start" ./
    ./u-boot.lds:50:	__u_boot_cmd_start = .;

    49     . = .;
    50     __u_boot_cmd_start = .;
    51     .u_boot_cmd : { *(.u_boot_cmd) }
    52     __u_boot_cmd_end = .;

没错，确实是这样的，看看在哪里用？

    user@vmware:~/mini2440/u-boot-1.1.6/board/100ask24x0$ cd ../../
    user@vmware:~/mini2440/u-boot-1.1.6$ grep -rsn "&__u_boot_cmd_start" ./
    ./lib_mips/board.c:320: 	for (cmdtp = &__u_boot_cmd_start; cmdtp !=  &__u_boot_cmd_end; cmdtp++) {
    ./common/command.c:246:				&__u_boot_cmd_start;	/* pointer arith! */
    ./common/command.c:251:		cmdtp = &__u_boot_cmd_start;
    ./common/command.c:349:	cmd_tbl_t *cmdtp_temp = &__u_boot_cmd_start;	/*Init value */
    ./common/command.c:360:	for (cmdtp = &__u_boot_cmd_start;
    ./common/command.c:435:		for (cmdtp = &__u_boot_cmd_start; cmdtp != &__u_boot_cmd_end; cmdtp++) {
    ./common/command.c:468:	for (cmdtp = &__u_boot_cmd_start; cmdtp != &__u_boot_cmd_end; cmdtp++) {
    ./lib_m68k/board.c:444:	for (cmdtp = &__u_boot_cmd_start; cmdtp !=  &__u_boot_cmd_end; cmdtp++) {
    ./lib_ppc/board.c:633:	for (cmdtp = &__u_boot_cmd_start; cmdtp !=  &__u_boot_cmd_end; cmdtp++) {

搜索 `__u_boot_cmd_start` 关键字，结果太多了，都是链接脚本的搜索结果，因此这里加了个 `&` 搜索 `&__u_boot_cmd_start`，或者在 source insight 下搜索

    ---- __u_boot_cmd_start Matches (7 in 2 files) ----
    do_help in command.c (common) : 				&__u_boot_cmd_start;	/* pointer arith! */
    do_help in command.c (common) : 		cmdtp = &__u_boot_cmd_start;
    find_cmd in command.c (common) : 	cmd_tbl_t *cmdtp_temp = &__u_boot_cmd_start;	/*Init value */
    find_cmd in command.c (common) : 	for (cmdtp = &__u_boot_cmd_start;
    complete_cmdv in command.c (common) : 		for (cmdtp = &__u_boot_cmd_start; cmdtp != &__u_boot_cmd_end; cmdtp++) {
    complete_cmdv in command.c (common) : 	for (cmdtp = &__u_boot_cmd_start; cmdtp != &__u_boot_cmd_end; cmdtp++) {
    command.h (include) line 57 : extern cmd_tbl_t  __u_boot_cmd_start;

不难看到有多个地方会遍历 `.u_boot_cmd` 段中的 `cmd_tbl_t` 对象，拿出来 `cmd_tbl_t` 对象出来搞事情。   
猜想：u-boot 命令模式下，等待用户输入命令后，会去遍历 `__u_boot_cmd_start` 的表格，找到用于输入的对应命令，并执行其中的 cmd 回调函数。  
验证猜想：从启动流程这块接着分析：

    main_loop
        run_command
            /* Look up command in command table */
            find_cmd
            /* OK - call function to do the command */
            cmdtp->cmd

没错，就是在 find_cmd 函数中去遍历所有的命令，找到当前要执行的命令，具体实现如下：

    cmd_tbl_t *find_cmd (const char *cmd)
    {
    	cmd_tbl_t *cmdtp;
    	cmd_tbl_t *cmdtp_temp = &__u_boot_cmd_start;	/*Init value */
    	const char *p;
    	int len;
    	int n_found = 0;
    
    	/*
    	 * Some commands allow length modifiers (like "cp.b");
    	 * compare command name only until first dot.
    	 */
    	len = ((p = strchr(cmd, '.')) == NULL) ? strlen (cmd) : (p - cmd);
    
    	for (cmdtp = &__u_boot_cmd_start;
    	     cmdtp != &__u_boot_cmd_end;
    	     cmdtp++) {
    		if (strncmp (cmd, cmdtp->name, len) == 0) {
    			if (len == strlen (cmdtp->name))
    				return cmdtp;	/* full match */
    
    			cmdtp_temp = cmdtp;	/* abbreviated command ? */
    			n_found++;
    		}
    	}
    	if (n_found == 1) {			/* exactly one match */
    		return cmdtp_temp;      // 找到了就返回命令的 cmd_tbl_t 对象
    	}
    
    	return NULL;	/* not found or ambiguous command */
    }

u-boot 启动后最后会运行到 mian_loop 中，main_loop 会去判断 bootm 和 bootargs 有没有，有的话，就自动启动，没有的话，就进入 u-boot 命令行，等到用户输入命令。  
但是不管是自动启动，还是等待用户输入命令，最后的结果都是调用 run_command 执行 u-boot 命令，最后都会通过 find_cmd 得到命令对应的 cmd_tbl_t 结构对象，运行命令对应的 cmd 回调函数。



