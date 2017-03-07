---
layout: post
title: "python命令行程序打包"
description: "打包python程序，并通过命令行调用"
category: "python"
tags: [setuptools, python]
---

>*我们知道在linux下通过命令行调用程序其实就是先把程序编译生成二进制文件，然后再把文件放入到PATH中，就可以直接通过文件名进行调用了，这里python程序也不例外。前面讲过了如何将一个python程序打包并发布，这里就讲一下命令行的python打包发布和下载安装过程。*

我们还是利用前面博客中的myproject为例，在原来的基础上添加一些代码：
{% codeblock %}
.
├── setup.py
└── src
    ├── bin
    │   ├── command-line
    │   └── myproject
    └── myproject
        ├── cmds.py
        ├── __init__.py
        └── __init__.pyc

3 directories, 6 files
{% endcodeblock %}
可以看出，比myproject一共多了三个文件command-line, myproject, cmds.py三个文件和bin这个文件夹。下面对他们进行详细的介绍：
command-line，mypreject文件  
这两个文件的内容很简单
{% codeblock %}
$ cat command-line 
  #!/usr/bin/env python
  import myproject
  myproject.test()
$ cat myproject
  #!/usr/bin/env python
  print 'welcome to command line mod!\n'
{% endcodeblock %}
可以看出，command-line文件只是调用了myproject的test函数，二myproject只是输出了一个字符串，为了能够执行这两个文件，并且是通过命令行的方式，我们需要在setup.py这个文件中添加下面这个配置：
{% codeblock %}
scripts=['src/bin/command-line', 'src/bin/myproject'],
{% endcodeblock %}
然后安装修改后的源码包:
{% codeblock %}
$ python setup.py install
...
Installing command-line script to /search/virtual/pypienv/bin
Installing myproject script to /search/virtual/pypienv/bin
...
{% endcodeblock %}
我们可以看到上面的输出，其含义就是把这两个脚本安装在了bin目录下，由于这个路径是PATH路径，所以可以直接通过命令进行执行了，下面是执行的效果：
{% codeblock %}
$ command-line 
  Hello World!
$ myproject
  welcome to command line mod!
{% endcodeblock %}
还有一个文件是cmds.py这个文件也是可以通过命令行直接运行的，只不过是通过另外一种方式进行安装的
先看一下cmds.py的内容吧：
{% codeblock %}
$ cat cmds.py 
  import myproject 
  def main():
      myproject.test()
{% endcodeblock %}
可以看出，其实这个文件和command-line一样，引用了myproject并输出，唯一不同的是他自己顶一个了一个函数main来调用。
同样setup.py中也要添加一个配置，来找到这个文件并对其进行编译：
{% codeblock %}
entry_points = {
                 'console_scripts': ['cmds=myproject.cmds:main'],
                }
{% endcodeblock %}
这个配置和前面的不一样，并且里面指定了要执行的脚本的函数名，通过从新安装我们可以看：
{% codeblock %}
$ python setup.py install
  ...
  Installing cmds script to /search/virtual/pypienv/bin
  ...
{% endcodeblock %}
同样，这个文件编译后也放到在PATH下，通过命令行可以运行：
{% codeblock %}
$ cmds 
  Hello World!
{% endcodeblock %}
到这里一个通过命令行运行的包就介绍完了.
同样我们可以这个写好的包上传到私有源，然后在其他机器上下载安装，同样可以运行~



