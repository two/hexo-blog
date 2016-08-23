---
layout: post
title: "中文搜索引擎coreseek的安装使用"
description: "中文搜索引擎coreseek的安装使用"
category: ""
tags: []
---

>项目中原来使用的是solr,由于solr是java写的，团队中没有java的经验，并且原来维护的人离职了，所以改用[sphinx search](http://sphinxsearch.com/)来做项目的内部检索服务，由于中文的特殊性，我们选择了[coreseek](http://www.coreseek.cn/)这个基于sphinx的中文搜索引擎。但是好像这个好久没人维护了～一直停留在了4.1beta版本～

##  下载解压
源码下载地址[oreseek-3.1-beta.tar.gz](http://www.coreseek.cn/uploads/csft/4.0/coreseek-4.1-beta.tar.gz);
解压下载的文件,进入目录`cd coreseek-4.1-beta`;

## 安装中文分词库mmseg

* `cd mmseg-3.2.14/`
* `./bootstrap`
* `./configure --prefix=/usr/local/mmseg3`
* `make && make install`

如果你幸运的话，这时候会提示错误：
{% codeblock %} 
css/SynonymsDict.cpp:94: 错误：在 {} 内将‘239’从‘int’转换为较窄的类型‘char’  
css/SynonymsDict.cpp:94: 错误：在 {} 内将‘187’从‘int’转换为较窄的类型‘char’  
css/SynonymsDict.cpp:94: 错误：在 {} 内将‘191’从‘int’转换为较窄的类型‘char’  
{% endcodeblock %}
可能是编译器的要求太严格导致的，只有更改源码了：
将rc/css/SynonymsDict.cpp文件94行的代码`char txtHead[3] = {239,187,191};`改为如下的形式
{% codeblock %}
char txtHead[3] = {}; 
txtHead[0] = (char)239;
txtHead[1] = (char)187;
txtHead[2] = (char)191;
{% endcodeblock %}
再次运行`make && make install`会发现又出现了错误提示：
{% codeblock %} 
mmseg_main.cpp: In function ‘int segment(const char*, css::Segmenter*)’:
mmseg_main.cpp:254: 错误：在 {} 内将‘239’从‘int’转换为较窄的类型‘char’
mmseg_main.cpp:254: 错误：在 {} 内将‘187’从‘int’转换为较窄的类型‘char’
mmseg_main.cpp:254: 错误：在 {} 内将‘191’从‘int’转换为较窄的类型‘char’
{% endcodeblock %}
同样的办法修改`src/mmseg_main.cpp`文件的254行`char txtHead[3] = {239,187,191};`为：
{% codeblock %}
char txtHead[3] = {}; 
txtHead[0] = (char)239;
txtHead[1] = (char)187;
txtHead[2] = (char)191;
{% endcodeblock %}
再次运行`make && make install`，终于成功了～
到这里mmseg就算安装成功了
## 安装coreseek
* `cd csft-4.1/`
* `sh buildconf.sh`
* `./configure --prefix=/usr/local/coreseek  --without-unixodbc   
--with-mmseg --with-mmseg-includes=/usr/local/mmseg3/include/mmseg/   
--with-mmseg-libs=/usr/local/mmseg3/lib/ --with-mysql`
* `make && make install`

## 测试mmseg分词，coreseek搜索（需要预先设置好字符集为zh_CN.UTF-8，确保正确显示中文）
{% codeblock %}
$ cd testpack
$ cat var/test/test.xml    #此时应该正确显示中文
$ /usr/local/mmseg3/bin/mmseg -d /usr/local/mmseg3/etc var/test/test.xml
$ /usr/local/coreseek/bin/indexer -c etc/csft.conf --all
$ /usr/local/coreseek/bin/search -c etc/csft.conf 网络搜索
{% endcodeblock %}
上面是官方文档给的，需要说明的是，csft.conf是由sphinx.conf.dist复制而来的，里面的关于mysql的配置，自己按照实际情况修改。

