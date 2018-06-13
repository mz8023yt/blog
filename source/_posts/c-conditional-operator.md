---
title: '[C] 条件表达式'
date: 2018-06-13 11:21:25
tags:
  - c
categories: C
---

### 遇到疑惑

最近在阅读内核源码的时候，发现条件表达式居然可以省略 ture 表达式，示例如下：

```c
// file: src/kernel/msm-3.18/drivers/video/msm/mdss/mdss_dsi_panel.c

void mdss_dsi_parse_esd_params(struct device_node *np, struct mdss_dsi_ctrl_pdata *ctrl)
{
	... ...
	lenp = ctrl->status_valid_params ?: ctrl->status_cmds_rlen;
	... ...
}
```

以往我们在使用条件表达式的格式是: `表达式1 ? 表达式2 : 表达式 3`。对于条件表达式的值是：判断表达式1的值，ture的话条件表达式的值为表达式2的值，false的话条件表达式的值为表达式3的值。  
但是上面 ture 对应的表达式为是空的，因此产生一个疑问: 难道将空值赋给 lenp 变量？

### 示例验证

决定在 ubuntu 上编写一个小示例程序验证下，验证用的示例程序如下：

```c
// file: conditional-operator.c

#include "stdio.h"

int main()
{
	int a = 0, b = 0, c = 0, d = 0;
	a = 7 ? : 1;
	b = 0 ? : 2;
	c = (7 > 4) ? : 3;
	d = (7 < 4) ? : 4;
	printf("a = %d, b = %d, c = %d, d = %d\n", a, b, c, d);
	return 0;
}
```

编译执行结果如下：

    user@vmware:~/c$ gcc conditional-operator.c -o app && ./app 
    a = 7, b = 2, c = 1, d = 4

同步想要确认下，如果省略表达式2，d的值是否是0? 保持其他不变，仅修改 d 的赋值语句。

```c
// file: conditional-operator.c

#include "stdio.h"

int main()
{
	int a = 0, b = 0, c = 0, d = 0;
	a = 7 ? : 1;
	b = 0 ? : 2;
	c = (7 > 4) ? : 3;
	d = (7 < 4) ? 4 : ;
	printf("a = %d, b = %d, c = %d, d = %d\n", a, b, c, d);
	return 0;
}
```

很遗憾，编译报错了，说明表达式3不允许省略。

    user@vmware:~/c$ gcc conditional-operator.c -o app && ./app 
    conditional-operator.c: In function ‘main’:
    conditional-operator.c:9:20: error: expected expression before ‘;’ token
    d = (7 < 4) ? 4 : ;

### 得出结论

从示例程序的运行结果可以发现，在表达式1为 true 的前提下，表达式2可以省略，此情况下会将表达式1的值作为整个条件表达式的结果。

