---
layout: post
title: "setuptools配置文件参数详解"
description: ""
category: "python"
tags: [python]
---

>*前面介绍了python的打包工具setuptools的用法，这里对其配置文件setup.py进行一个详细的介绍，以便以后使用的时候能够找到相应的配置。*

##常见的参数
这里的参数主要是参考[setuptools官方文档](http://pythonhosted.org/setuptools/setuptools.html#new-and-changed-setup-keywords)而来，下面经常用到的参数进行一个详细的介绍，以后要是有其他的参数，再做补充。
**name**  
这个参数主要是标识工程的名字。这个名字就是打包后的包的名字。  
**version**  
这个参数主要是标识工程的版本号，打包后的名字包含这个版本号。  
在安装同一个包的不同版本是可以用如下命令：
{% codeblock %}
$ pip install -i http://10.11.215.61:3141/simple/ myproject==1.1.7 
{% endcodeblock %}
**packages**  
这个参数主要是说明要打包的工程的源代码路径，默认不写的话是从本工程的根目录开始，也可自定义，要引入find_packages函数，函数参数是包的相对路径.。  
**package_dir**  
这个主要是为了说明包的路径
**zip_safe**  
**scripts**  
**entry_points**  
**include_package_data**  
这个参数是为了表示打包是需要包含的文件，如果值为True，表示setuptools打包时会自动包含package目录下的所有文件，这些文件可能是在版本控制下或者在MANIFEST.in文件中表明的文件。详情请参考[文档](http://pythonhosted.org/setuptools/setuptools.html#including-data-files)
注意MANIFEST.ini的写法：
{% codeblock %}
include README.rst #包含具体的文件
...
include django/dispatch/license.txt #包含相对目录的文件
recursive-include docs * #循环包含某个文件夹下所有的文件
{% endcodeblock %}
ps:亲测，版本控制这个好像不太靠谱，还是自己写都包含哪些文件吧~  
**package_data**  
跟上面的参数意思差不多，只不过这里表示粒度比较小的文件，上面没有包含到的文件可以在这里包含，一边表示比较具体的文件。
**description**  
**long_description**  
**author**  
**author_email**  
**license**  
**keywords**  
**platforms**  
**url**  




