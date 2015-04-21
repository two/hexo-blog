---
layout: post
title: "linux shell 编程之语法学习"
description: "linux shell 基本语法学习"
category: "SHELL"
tags: [shell]
---

>shell语法跟一般类C的语法有些相识，但是却有很多独特的地方，如果不能够好好理解这些语法特性，难免在编写shell脚本的过程中会遇到很多令人难以察觉的，头疼的问题。细节决定成败，这篇博客就根据我自己的学习过程做一下总结吧。

## 独特的开头
一般的脚本语言都有一个基本表示自己是何方神圣的开头，比如php语言的`<?php`, jsp语言的`<%jsp`。shell也有自己独特的开头。比如`#! /bin/bash`, 不过跟其他语言表示自己的语言名称不一样，这里的开头表明shell要指明使用那个解释器。因为shell有很多标准，每个标准的解释器对shell的理解是不一样的，所有你写的shell脚本很可能是其他解释器不认识的内容，所以需要指明你自己使用哪个解释器。这里要注意的是如果写了`#! /bin/sh` 表明使用的是当前系统默认的解释器。

>ps小贴士：
>>
 1. `#!`一定要写在脚本第一行才能生效， 否则视为注释行, 不起任何作用
 2. 这一行是会被执行的，如果写其他命令如`#! /bin/more` 则会执行相应的命令

## 执行shell脚本的方法
shell执行的方式不同，其与运行的远离也不同，参考[Shell如何执行命令](http://learn.akae.cn/media/ch31s02.html)这篇文章，进行介绍一下：
首先有一个脚本`script.sh`,内容如下：
{% codeblock %}
#! /bin/sh

cd ..
ls
{% endcodeblock %}
执行这个脚本的方法有两种：

* `$ ./script.sh`
* `$ sh ./script.sh`

但其实第一种方法会转化为第二种方法，比如如果用第一种方法执行脚本，实际上会转化为：`/bin/sh ./script.sh`。接下来就调用shell子进程来执行脚本了，在脚本执行的所有都是针对子进程的，不会对父进程产生影响，这点可以参考博客中举的例子。

>ps小贴士：
>>
  1. 第一种方式执行脚本需要这个脚本具有可执行的权限, 第二种则不需要

## 变量的表示 
Bash变量像一般的脚本语言一样，不区分变量类型，本质上Bash变量都是字符串。  

###变量替换
变量的名字就是变量保存值的地方，引用变量的值就叫*变量替换*。我们用`$+变量名称`来表示变量替换。下面几种情况变量不带`$`:

* 变量被声明: `variable`
* 变量被赋值: `variable=12`
* 变量被unset: `unset variable`
* 变量被export: `export variable=23`

*弱引用（部分引用）：*双引号("")括起来的变量。这种引用变量替换不会被阻止。
*强引用（全引用）：*单引号（''）括起来的变量。替换被阻止，解释为字符串。

ps: $variable其实是${variable}的简写形式,但是有些时候简写形式不能够满足变量的更多特性（比如参数替换），这时候就要用全写形式了。
###变量赋值:
变量赋值的方式也有多种：

1. 使用赋值操作:=(注意等号两边不能有空白，否则就是视为条件判断了)赋值.`a=12`
2. 使用let赋值.`let a=2`
3. for循环中赋值(伪赋值).`for a in \`seq 100\``

###特殊的变量类型

* 位置参数： 命令行运行脚本可以传递参数，而脚本接受参数可以通过位置参数来获取。
    - $0:代表脚本名称
    - $1:表示第一个传递的参数(依次类推, 但是$10开始需要需要用${10}来获取)
    - $#:表示参数的个数

## 条件判断
###条件判断格式：
{% codeblock %}
# 1 一般的格式
if 条件
then
     ...
else
     ... 
fi

# 2 两条语句写到一行是，语句间要用分号隔开
if 条件; then
    ...
else
    ...
fi

# 3 多个条件判断
if 条件
then
    ...
elif 条件
    ...
fi

# 4 嵌套条件判断
if 条件
then
    if 条件
    then
        ...
    fi
fi
{% endcodeblock %}

###条件格式：

1. `[ ... ]` : `[`是一个内建命令，其作用就是后面的表达式退出状态码为0则条件为真。
2. `[[ ... ]]`: `[[`是一个关键字，其作用就是和上面相同，但是其表达式的解释不同。
3. `(( ... ))`: 测试条件是一个算术表达式，表达式的结果为非零时条件为真。
4. `一般命令`: 可跟一般的shell命令，命令的退出状态码为测试条件。

> ps: `[]` 与`[[]]`的区别：  
    `[`是一个内建命令, 其后面跟的是一般命令的参数和选项，比如`[ 1 -lt 2]`可以认为1和2是参数，-lt是一个选项。但是 `[ 1 && 2 ]`则不正确，因为 &&不是一个有效的参数或者选项。
    `[[`是一个关键字，可以正确解释&&， 是一个更加通用的结构。

## 循环
循环的方式主要有以下几种：
{% codeblock %}
#1 基本结构
for arg in [list]
do
commands...
done

#2 使用$@(即位置参数)
for arg
do
commands...
done

#3 使用命令替换[list]
for arg in `command`
do
commands...
done

#4 使用C风格
for ((a=1; a <= LIMIT ; a++))
do
commands...
done

#5 while
while [condition] #与条件判断的condition一样
do
commands...
done

#6 until 类似于C的do...while
until [condition]
do
commands...
done

#7 嵌套循环与控制
for a in [list]
do
    for b in [list]
    do
        for c in [list]
        do
            break 2 #带层参数：退出从本层算，往外到第二层的循环
        done
        for d in [list]
        do
           continue #不带层参数：继续本层的循环
        done
        break  #不带层参数：退出本层循环
    done
done
{% endcodeblock %}

##分支

{% codeblock %}
#case 
case "$variable" in
$condition1 ) command...;;
$condition2 ) command...;;
...
$conditionn ) command...;;
*) command ...;; #相当于default
esac

#select 介于for和case之间的一个if语句
select variable [in list] #满足条件则执行下面的命令, 不写后面的则跟for一样，取$@
do
commands...
done
{% endcodeblock %}

##函数

###函数定义的两种方式：  
{% codeblock %}
function function_name {
$1 #位置参数作为传递的参数
$2
command...
}

function_name {
command...
}
{% endcodeblock %}

###函数调用
{% codeblock %}
function_name $arg1 $arg2
{% endcodeblock %}
ps:
1. 函数调用之前必须先定义

##结语
到这里一个shell的基本架构有了，但是shell学习才刚刚开始，以后陆续有文章就shell的某个点进行深入探讨～
