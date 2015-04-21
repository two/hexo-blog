---
layout: post
title: "python打包高级进阶:pip and easy_install"
description: ""
category: "python"
tags: [python,]
---


##如何选择已经安装的不同版本？
其实很简单。以myproject为例：`easy_install myproject==version` 就可以了，默认是先从本地开始找，如果本地找不到就会从远程源上找。
##能不能指定安装路径？
经过再次了解，发现其实是可以的,`easy_install -h`命令可以看到有以下两条：
{% codeblock %}
  --install-dir (-d)             install package to DIR
  --script-dir (-s)              install scripts to DIR
  --prefix                       installation prefix 
{% endcodeblock %}
第一个其实就是可以指定默认搜索路径的。第二个是可以指定二进制脚本的安装路径，如果指定了第一个而没有指定第二个那就会默认都安装到第一个,第三个是实际包的安装路径，但是只是路径前缀，默认会加上lib/python2.6/site-packages  
还有一个问题，如果按照路径不在PYTHONPATH里，那么就会报错，可以通过先把你要制定的路径加入进去：
{% codeblock %}
PYTHONPATH="${PYTHONPATH}:/search/sean/python/packages"
PYTHONPATH="${PYTHONPATH}:/search/sean/lib/python2.6/site-packages"
export PYTHONPATH
{% endcodeblock %}
然后再次安装：
{% codeblock %}
easy_install -d /search/sean/python/packages/ -s /search/sean/python/ --prefix /search/sean  myproject==1.9
{% endcodeblock %}
可以看到-d需要PYTHONPATH 而-s并不需要，安装完后可以在两个目录中找到安装后的文件
更正：之前说安装多版本的时候会有一个去掉版本的文件夹，其实这个是pip的实现方式，而easy_install的实现方式是：每次安装都会把easy-install.pth路径修改了，python就会去找修改过的路径。
具体pip的版本选择和安装路径的问题时间限制先不发了，研究后再补充~
