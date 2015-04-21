---
layout: post
title: "plantuml在Mac OS X系统下中文乱码问题解决"
description: ""
category: "tools"
tags: [plantuml]
---

>安装完plantuml使用plantuml.jar生成的uml图中文会出现乱码，这个问题困扰了我大半天，终于解决了～

首先编写uml程序的文件的编码方式是utf-8的，test.uml内容如下：
{% codeblock %}
@startuml
|Brower|
start
:点击领取按钮;
|WebServer|
:领取勋章;
|Cache|
:更新缓存;
|DB|
:更新DB;
@enduml
{% endcodeblock %}
但是利用`java -jar plantuml.jar -tsvg test.uml`或者利用plantuml的vim插件命令`:make`都可以正常生成，但是里面的中文都是乱码。实在是让我发愁啊，在网上找了好久，只有一篇博客提到了这个问题，就是[ubuntu plantuml的中文问题 ](http://blog.sina.com.cn/s/blog_53e449530101408m.html)。但这并不是我想要的，首先我的系统是Mac OS X,跟博文中的系统不一样，其次我的系统对中文编码是支持的，其他的都没问题，只是这里有问题。最后还是自己帮了自己，使用`java -jar plantuml.jar -h`可以看到有个`-charset`选项，但是后面说的默认编码格式却是GBK2312，总有找到问题了，于是我在命令里加了下面的参数`java -jar plantuml.jar -charset utf-8 -tsvg test.uml`，在看结果终于可以了～中文显示正常了！  
