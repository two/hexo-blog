---
layout: post
title: "C语言：可变参数函数"
description: ""
category: "C"
tags: []
---

>函数一般的参数都是固定的，但是有些时候我们需要让函数的参数是可变的，为了满足这个需求，C语言提供了库函数stdarg.h来满足要求。

##可变参数参数简介

##使用方法
可变参数函数的使用要求比较严谨，必须按照下面的方法进行使用：

1. 在函数原型中使用省略号。
2. 在函数定义中创建一个va_list类型的变量。
3. 用宏将改变量初始化为一个参数列表。
4. 用宏访问这个参数列表。
5. 用宏完成清理工作。

##函数原型
{% codeblock %}
void f1(int n, ...); //合法
int f2(int n, const char *s, ...);  //合法
char f3(char c1, ..., char c2); //无效，省略号必须是最后一个参量
doubel f3(); //无效，没有任何参量
{% endcodeblock %}


##程序举例

{% codeblock %}
#include <stdio.h>
#include <stdarg.h>

double sum(int , ...);
void printSth(int, const char *, ...);

int main(void)
{
    double s, t;

    s = sum(3, 1.1, 2.5, 13.3);
    t = sum(6, 1.1, 2.1, 13.1, 4.1, 5.1, 6.1);
    u = sum(6, 1.1, 2.1, 13.1, 4.1, 5.1, 6.1);
    printf("return value for "
            "sum(3, 1.1, 2.5, 13.3):    %g\n",s);
    printf("return value for "
            "sum(6, 1.1, 2.1, 13.1, 4.1, 5.1, 6.1)  %g\n", t); 
    
    return 0;
}


double sum(int lim, ...)
{
    va_list ap; //声明用户存放参数的变量
    double tot = 0;
    int i;
    va_start(ap, lim);  //把ap初始化为参数列表
    for(i = 0; i < lim; i++)
        tot += va_arg(ap, double); //访问参数列表中的每一个项目
    va_end(ap); //清理工作
    return tot;
}
{% endcodeblock %}

##程序分析

* 具有可变参数的函数`sum`的第一个参数是表示一共有多少个不确定的参数，在调用的时候传递。这个参数主要是高数`var_arg`能够取多少次传递的参数。
* 函数[`va_start()`](http://www.tutorialspoint.com/c_standard_library/c_macro_va_start.htm)第一个参数是va_list类型的，第二个参数是这个函数确定的参数中最有一个参数，var_start()函数的作用就是把最有一个确定参数后面的所有不确定参数做一个参数列表存储到va_list类型的变量中。
* 函数[`va_arg`](http://www.tutorialspoint.com/c_standard_library/c_macro_va_arg.htm)的目的是从未知参数列表中取出参数，每次调用取一个，按照参数顺序取。第一个参数是存储参数列表的变量，第二个参数标识要去的参数的类型，类型决定了要从内存栈中读取多少位。
* 函数[`va_end`](http://www.tutorialspoint.com/c_standard_library/c_macro_va_end.htm)函数主要是做清理工作,主要是释放动态分配的用于存放参数的内存。

##运用
我在学了《C Primer Plus》这本书之后打算看看redis源码的时候发现redis的源码中，所有的系统log的记录都是通过这中函数来实现的：
{% codeblock %}
void redisLog(int level, const char *fmt, ...) {
    va_list ap;                
    char msg[REDIS_MAX_LOGMSG_LEN]; 
    
    if ((level&0xff) < server.verbosity) return;
    
    va_start(ap, fmt);         
    vsnprintf(msg, sizeof(msg), fmt, ap);
    va_end(ap);                
    
    redisLogRaw(level,msg);    
}   
{% endcodeblock %}
