---
layout: post
title: "python pypi 私有源搭建"
description: ""
category: "python" 
tags: [python, pypi]
---

>*今天学习了pypi的私有源的搭建。过程很曲折，出现了很多问题，这里做个记录，希望以后的人能有所借鉴，自己也做一下总结……*

## 什么是pypi
这个问题可以从[官方文档](https://pypi.python.org/pypi)得到答案。pypi的全称是：The Python Package Index，翻译过来就是一个python包的索引，python包存储在一个统一管理的仓库中，然后为所有在这个仓库的软件生成一个列表,这就是pypi.
## 如何使用pypi
用过python的同学都知道，通过下面三个命令都可以安装python包。但是它们之间的区别又是什么呢？
{% codeblock %}
    1. python setup.py  install
{% endcodeblock %}
{% codeblock %}
    2. pip install packagename
{% endcodeblock %}
{% codeblock %}
    3. easy_install packagename
{% endcodeblock %}
第一种方式就是先把要安装的包的模块下载到本地，然后进入解压目录，执行上面的命令。
第二种和第三种都是直接输入命令，就会自动从pipy库中下载相应的包然后安装的预先定义好的路径。
如果不是要修改源码，第二种和第三种方式无疑是最好的。这里先不深入他们的原理，以后再讲。

## 为什么要搭建pypi私有源
既然有了官方的pypi源，为什么我们还要搭建自己的私有源呢？这个问题其实很好理解。公司内部开发的软件包希望能够通过这种方式让大家安装，但如果上传到官方的源无疑是公司所不希望的，这时私有源就发挥作用了。它只在公司内部服务器上存在，安装方式和官方的源一样，既简单有安全，何乐而不为呢~

## 如何搭建自己的私有源
终于到了正题了，如何搭建我们自己的私有源呢？这里我主要参考了[易先生的世界](http://yijingping.github.io/2013/07/25/setting-up-your-own-pypi-server.html)这篇博客。  
从中我们可以得知python的私有源解决方案有很多种，这些解决方案可以在python关于私有源的[官方文档](https://wiki.python.org/moin/PyPiImplementations)中找到。这里我们选择了据说是最简单好用的一个[pypiserver](https://pypi.python.org/pypi/pypiserver),下面就这个私有源的安装过程进行一个说明(ps:我的软件都是用virtualenv方式安装的）：  
**顺利的安装方式**  
{% codeblock %}
# 进入virtualenv目录。执行：
virtualenv pypienv
# 然后进入pypienv环境
source pypienv/bin/activate
# 安装pypiserver
pip install pypiserver
# 建立一个存放软件包的目录,这个目录路径可按照自己的需求建立
mkdir /PATH/packages
# 启动pypi-server 端口号自定义 后接包的上传地址
exec pypi-server -p 3141 /search/ziyuan/pypiserver/packages
{% endcodeblock %}
到这里其实pypi-server的安装已经完成了。但是我们安装好了要用，这才是重点。  
**上传package**  
假设我们已经做好了一个package，如何上传到指定的私有源呢？
首先上传需要用户名和密码，这个需要在服务端生成，密码文件使用命令htpasswd生成。首先安装passlib模块
{% codeblock %}
    pip install passlib
{% endcodeblock %}
然后安装apache2-utils模块。由于我使用的readhat,并没有在yum中找到apache2-utils源，只有自己下载安装了。[下载地址](http://download.opensuse.org/factory/repo/src-oss/suse/src/apache2-2.4.6-7.3.src.rpm)  
{% codeblock %}
     rpm -ivh apache2-2.4.6-7.3.src.rpm
{% endcodeblock %}
最后生成用户名和密码:
{% codeblock %}
     htpasswd -sc /PATH/.htaccess user #回车输入密码。输入：123
{% endcodeblock %}
用户名密码都生成了，接下来就是从新启动pypiserver，让他读入设置好的用户名密码
{% codeblock %}
    exec pypi-server -p 3141 -P /PATH/.htaccess /search/ziyuan/pypiserver/packages
{% endcodeblock %}
用户上传软件包之前要指定上传的的地址，否则就会传到官方的默认路径去，执行路径的配置文件要放在用户主目录下，文件名.pypirc,内容如下：
{% codeblock %}
    [distutils]
    index-servers =
       local #注意：这个缩进必须要有，不然就会报错，解析不了配置文件 

    [local]
    repository:http://127.0.0.1:3141
    username:user
    password:123 
{% endcodeblock %}
这是就可以在和发布的软件包中域名命令上传软件包到私有源了：
{% codeblock %}
     python setup.py sdist upload -r local 
{% endcodeblock %}
**下载package**  
前面也提到了有两种下载方式可以直接从源中下载，根据打包的方式不同。它们分别是：
{% codeblock %}
   pip install -i http://localhost:3141/simple/ packagename
   easy_install -i http://localhost:3141/simple/ packagename
{% endcodeblock %}
到这里，pypiserver的安装和使用已经完成了，但是其原理还需要研究，以后要做到定制一些功能~
