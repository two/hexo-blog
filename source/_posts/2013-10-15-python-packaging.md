---
layout: post
title: "python打包"
description: ""
category: "python"
tags: [python, setuptools]
---

>*python的打包方式有很多种，而且针对不同的平台又有很多不同的方式，这里指针对linux平台进行介绍……*

## 1 python打包工具选择  
关于python打包相关的工具，这篇文档有一个[列表](https://python-packaging-user-guide.readthedocs.org/en/latest/projects.html#)，其中打包工具主要分为pip, dislib, setuptools, distribute(已经于setuptools合并）。还有博客也是关于python 打包工具的，[这篇博客](http://www.ituring.com.cn/article/19090)对于python的各种打包工具，及其优缺点写的很详细，里面说的Distutils2是更为先进的打包工具，但是适用范围还不是太广泛，这里先不做研究了。经过综合对比，目前选中setuptools和pip这两个个比较成熟的打包工具，下面就对他们用法和原理分别进行详细的介绍。  

### 2 setuptools
[setuptools的官方文档地址](http://peak.telecommunity.com/DevCenter/setuptools)  
首先来一个简单的上手过程，通过这个过程可以快速了解setuptools怎么用的，详情请移步博客[python egg 学习笔记](http://www.worldhello.net/2010/12/08/2178.html)
看完这个教程就能自己做一个简单的包了，但是这里只是涉及了一种包，就是python import 引过来，然后调用其中的函数，还有一种也很普遍的方式就是命令行方式，例如jekyll命令，sentry命令等，这种方式其实可以参考[这篇文档](http://www.scotttorborg.com/python-packaging/index.html)。跟随这两篇博客基本就可以满足我们的需求了
这里对其安装步骤进行一下总结：  
#### 2.1, 安装setuptools
{% codeblock %}
    $ wget http://peak.telecommunity.com/dist/ez_setup.py
    $ sudo python ez_setup.py
{% endcodeblock %}
其实安装的方式有很多种，具体方法参见前面提到的博客  
#### 2.2, 建立一个工程
{% codeblock %}
    $ mkdir mypackage 
    $ cd mypackage
    $ touch setup.py
{% endcodeblock %}
setup.py的内容如下：
{% codeblock %}
  1 #setup.py
  2 #!/usr/bin/env python
  3 #-*- coding:utf-8 -*-
  4 
  5 from setuptools import setup, find_packages
  6 
  7 setup(
  8         name = "myproject",
  9         version="0.1.0",
 10         packages = find_packages(),
 11         zip_safe = False,
 12 
 13         description = "egg test demo.",
 14         long_description = "egg test demo, haha.",
 15         author = "sean",
 16         author_email = "seanerchen@gmail.com",
 17 
 18         license = "GPL",
 19         keywords = ("test", "egg"),
 20         platforms = "Independant",
 21         url = "",
 22         )
{% endcodeblock %}
第8行表示这个工程的名称，第9行是这个工程的版本号，第10行表示对指定目录的文件打包，其他信息请查看相关文档，这里先对用到的配置项进行讲解  
#### 2.3, 通过命令行打包到本地
这是一个空的工程就建好了，我们可以通过一条简单的命令对它进行打包了：
{% codeblock %}
   python setup.py bdist_egg 
   #python setup.py bdist 命令也可以。它们之间的区别?
{% endcodeblock %}
命令执行后，myproject目录的的结构如下：
{% codeblock %}
    .
    ├── build
    │   └── bdist.linux-x86_64
    ├── dist
    │   └── myproject-0.1.0-py2.6.egg
    ├── myproject.egg-info
    │   ├── dependency_links.txt
    │   ├── not-zip-safe
    │   ├── PKG-INFO
    │   ├── SOURCES.txt
    │   └── top_level.txt
    └── setup.py
{% endcodeblock %}
我们可以看到在dist文件夹下生成了一个egg文件，通过使用命令file可以查看这个文件的格式：
{% codeblock %}
    $ file dist/myproject-0.1.0-py2.6.egg
    dist/myproject-0.1.0-py2.6.egg: Zip archive data, at least v2.0 to extract
{% endcodeblock %}
其实egg文件就一个压缩包,关于egg的介绍请参考[这篇博客](http://peak.telecommunity.com/DevCenter/PythonEggs)  
#### 2.4, 写自己的程序并打包到本地
在当前目录下建立一个文件夹，名字与setup.py里写的name字段保持一致,然后在进入目录，创建__init__.py文件
{% codeblock %}
    $ mkdir myproject
    $ cd myproject
    $ touch __init__.py
{% endcodeblock %}
文件\_\_init\_\_.py内容如下：
{% codeblock %}
  1 #!/usr/bin/env python
  2 #-*-coding:utf-8-*-
  3 
  4 def test():
  5     print "Hello World!"
  6 if __name__ == '__main__':
  7     test()
{% endcodeblock %}
再次运行生成egg的命令：

{% codeblock %}
   python setup.py bdist_egg 
{% endcodeblock %}
这时整个目录结构变成了下面的情况：
{% codeblock %}
.
├── build
│   ├── bdist.linux-x86_64
│   └── lib
│       └── myproject
│           └── __init__.py
├── dist
│   └── myproject-0.1.0-py2.6.egg
├── myproject
│   └── __init__.py
├── myproject.egg-info
│   ├── dependency_links.txt
│   ├── not-zip-safe
│   ├── PKG-INFO
│   ├── SOURCES.txt
│   └── top_level.txt
└── setup.py
{% endcodeblock %}
可以看一下这个egg文件的内容：
{% codeblock %}
    $ unzip -l dist/myproject-0.1.0-py2.6.egg 
      Archive:  dist/myproject-0.1.0-py2.6.egg
        Length      Date    Time    Name
      ---------  ---------- -----   ----
            118  11-01-2013 14:35   myproject/__init__.py
            354  11-01-2013 14:35   myproject/__init__.pyc
            232  11-01-2013 14:35   EGG-INFO/PKG-INFO
            194  11-01-2013 14:35   EGG-INFO/SOURCES.txt
              1  11-01-2013 14:35   EGG-INFO/dependency_links.txt
              1  10-21-2013 12:29   EGG-INFO/not-zip-safe
             10  11-01-2013 14:35   EGG-INFO/top_level.txt
      ---------                     -------
            910                     7 files
{% endcodeblock %}
可以看到里面多出来一个myproject文件夹和它所包含的文件，正是我们刚才创建的。  
#### 2.5, 打包到远程
这里我们只是把打的包放在了本地, 也可以根据前面讲的打包到远程的源中，这里结合前面的博客把这个包发送到私有源，命令如下：
{% codeblock %}
$ python setup.py sdist upload -r local 
running sdist
running egg_info
...
removing 'myproject-0.1.0' (and everything under it)
running upload
Submitting dist/myproject-0.1.0.tar.gz to http://127.0.0.1:3141
Server response (200): OK
{% endcodeblock %}
这样我们就可以把本地的包上传到了远程的私有源`local`上供其他人使用了。

#### 2.6, 安装和使用
安装可以分为本地安装和远程安装。这个和之前私有源建立的博客中讲的一样，具体方式如下：
  **1,本地安装:**  
{% codeblock %}
$ sudo python setup.py install
  running install
  running bdist_egg
  running egg_info
  ... 
  ... 
  Installed /root/.local/lib/python2.6/site-packages/myproject-0.1.0-py2.6.egg
  Processing dependencies for myproject==0.1.0
  Finished processing dependencies for myproject==0.1.0
$ python -c "from myproject import test; test()"
  Hello World!
{% endcodeblock %}
可以看到我们的包已经被安装了，并且可以运行我们之前在包里封装的方法。  
 **2,远程安装:**  
我们可以安装远程私有源的包到本地:
{% codeblock %}
# 安装远程包到本地
$ easy_install -i http://10.11.215.61:3141/simple/ myproject  
  Searching for myproject
  Reading http://10.11.215.61:3141/simple/myproject/
  Best match: myproject 0.1.0
  Downloading http://10.11.215.61:3141/packages/myproject-0.1.0.tar.gz
  Processing myproject-0.1.0.tar.gz
  Writing /tmp/easy_install-szH61Z/myproject-0.1.0/setup.cfg
  Running myproject-0.1.0/setup.py -q bdist_egg --dist-dir /tmp/easy_install-szH61Z/myproject-0.1.0/egg-dist-tmp-aI8vPL
  Adding myproject 0.1.0 to easy-install.pth file

  Installed /usr/lib/python2.6/site-packages/myproject-0.1.0-py2.6.egg
  Processing dependencies for myproject
  Finished processing dependencies for myproject
{% endcodeblock %}
这时刚才上传到私有源的包就安装在了本地，安装路径可以从安装的信息中看到。安装的路径为：`/usr/lib/python2.6/site-packages/myproject-0.1.0-py2.6.egg`  
#### 2.7, 代码结构规划
一般情况下为了使代码看起来更加有条理，我们把源代码放到src文件夹下。首先把myproject文件夹移动到src中，然后删除以前产生的文件，只保留自己写的问文件，其结构大致如下：

{% codeblock %}
   ├── setup.py
└── src
    └── myproject
        ├── __init__.py
        └── __init__.pyc 
{% endcodeblock %}
setup.py内容也要修改，修改默认的搜索路径为src，而不是从根目录开始，内容如下：
{% codeblock %}
# setup.py
# !/usr/bin/env python
# -*- coding:utf-8 -*-

from setuptools import setup, find_packages

setup(
        name = "myproject",        
        version="0.1.0", 
        packages = find_packages('src'), 
        package_dir = {'':'src'},
        zip_safe = False,

        description = "egg test demo.", 
        long_description = "egg test demo, haha.", 
        author = "sean",
        author_email = "seanerchen@gmail.com",

        license = "GPL",
        keywords = ("test", "egg"),
        platforms = "Independant",
        url = "",
        )
{% endcodeblock %}
然后运行命令生成egg等文件：
{% codeblock %}
$ python setup.py bdist_egg

$ tree
.
├── build
│   ├── bdist.linux-x86_64
│   └── lib
│       └── myproject
│           └── __init__.py
├── dist
│   └── myproject-0.1.0-py2.6.egg
├── setup.py
└── src
    ├── myproject
    │   ├── __init__.py
    │   └── __init__.pyc
    └── myproject.egg-info
        ├── dependency_links.txt
        ├── not-zip-safe
        ├── PKG-INFO
        ├── SOURCES.txt
        └── top_level.txt

8 directories, 10 files
{% endcodeblock %}
然后用同样的方式进行安装运行，依然没有问题。  
#### 2.8, 卸载程序
安装过的python包，如果要删除该如何做呢？
首先要找到easy-install.pth文件,这个文件是自动查询安装包的路径的文件，如果把安装包的路径从这个文件中去掉，将找不到要使用的包
当运行安装程序发现最后几行信息:  
{% codeblock %}
$ python setup.py install
  ...
  Processing myproject-0.1.0-py2.6.egg
  creating /root/.local/lib/python2.6/site-packages/myproject-0.1.0-py2.6.egg
  Extracting myproject-0.1.0-py2.6.egg to /root/.local/lib/python2.6/site-packages
  Adding myproject 0.1.0 to easy-install.pth file

  Installed /root/.local/lib/python2.6/site-packages/myproject-0.1.0-py2.6.egg
  Processing dependencies for myproject==0.1.0
  Finished processing dependencies for myproject==0.1.0
{% endcodeblock %}
可以看出包的安装路径和包的路径信息都能找到，删除相应的包和路径就会发现无法使用该包了。  
打开文件easy-install.pth，删除./myproject-0.1.0-py2.6.egg这一行，或者直接删除安装好的egg包,再次运行python时就会提示找不到该包
{% codeblock %}
>>> from myproject import test
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ImportError: No module named myproject
{% endcodeblock %}
到这里整个配置文件的写法，源代码的写法，打包到本地和打包到远程，本地安装和远程安装都介绍完了。
