---
title: '[C] 引用外部函数'
date: 2018-06-27 22:01:23
tags:
  - c
  - extern
categories: C
---

### 问题

当一个文件中需要引用另一个文件中的函数时，一般情况下都会使用 extern 关键字来声明外部函数，然后调用它。
如果某个时候，我们修改了被引用的函数原型，但是在 extern 外部申明的位置却忘了修改，会发生什么？
如果函数原型改变的话，对应的每个 extern 声明的地方都要改一遍，如果有地方没改到呢？我们通过一个例子来看下悲剧是怎么发生的。

### 示例

文件1：fun.c
```c
#include <stdio.h>

void fun(int data)
{
        printf("data = %d\n", data);
}
```

文件2：main.c
```c
#include <stdio.h>

// 注意这里把 fun 声明成无参数的了
extern void fun();

int main(int argc, char* argv[])
{
        fun();
        return 0;
}
```

文件3：Makefile
```
objs := main.o fun.o

app:$(objs)
	gcc -Wall -O2 -o app $(objs)

%.o:%.c
	gcc -Wall -O2 -c -o $@ $<

clean:
	rm -f *.o app
```

执行效果：

    user@vmware:~/workspace/c/extern-bug$ make
    gcc -Wall -O2 -c -o main.o main.c
    gcc -Wall -O2 -c -o fun.o fun.c
    gcc -Wall -O2 -o app main.o fun.o
    user@vmware:~/workspace/c/extern-bug$ ./app 
    data = 1

### 分析

1. 编译的时候即使加了 -Wall 选项也没有编译告警？  
   在 main.c 中，fun 函数被 extern 重定义成无参数的了，所以编译不会告警。如果把 extern 声明去掉，编译器还会给个"函数未显式定义"的警告。

2. 程序链接也没报错？  
   C 语言中，编译出来的函数符号表中是不包含参数的，如下所示，也就是为什么 C 语言不能做编译时的多态。所以，别指望在链接的时候会报错。

       user@vmware:~/workspace/c/extern-bug$ nm fun.o 
       0000000000000000 T fun
                        U printf

3. 程序竟然还能运行？  
   程序输出了1， 这个1是哪里来的？看看下面的执行效果，你是不是明白为什么了？

       user@vmware:~/workspace/c/extern-bug$ ./app 2 3 4 5 6
       data = 6
       user@vmware:~/workspace/c/extern-bug$ ./app 2 3 4 5 6 7 8 9
       data = 9
       user@vmware:~/workspace/c/extern-bug$ ./app 2 3
       data = 3

   竟然把 argc 的数值打印出来了！  
   运行的时候调用的肯定还是带参数的 func 函数，但是参数从哪里来呢？处于栈顶的 argc 就被取出来塞给 func 函数了，所以它的数值被打印出来了{压栈方式不同的话，也有可能打印argv)。

都这样了，接下来离各种异常还远吗？这种问题定位起来会搞死人的。

### 改进

建议通过头文件引用的方式来调用外部函数，追加一个新文件 fun.h

```c
#ifndef __FUN_H__
#define __FUN_H__

void fun(int data);

#endif
```

如果 main.c 中 fun 调用还是没有传参

```c
#include <stdio.h>
#include "fun.h"

int main(int argc, char* argv[])
{
        fun();
        return 0;
}
```

则会报错，可以及时发现问题

    main.c: In function ‘main’:
    main.c:6:9: error: too few arguments to function ‘fun’
             fun();
             ^
    In file included from main.c:2:0:
    fun.h:4:6: note: declared here
     void fun(int data);
          ^
    Makefile:7: recipe for target 'main.o' failed
    make: *** [main.o] Error 1



