---
layout: post
title: "安装新主题"
description: ""
categories: [blog, jekyll]
tags: [jekyll,]
---

>*安装完jekyll后很是兴奋，对于一向是外貌协会的我自然希望能找一个好的博客主题，这个主题一定要高端大气上档次，并且体现黑客的风格，这样才好装B啊，于是一天的时间都花在这上面了~ * 

##心血来潮
对于一个新手来说，安装好jekyll后，我第一件事就是希望能够找到一个漂亮的主题，切合我博客的内容，于是我开始了忙碌的寻找……
在学习jekyll boostrapy的过程中看到了可以[自定义主题](http://jekyllbootstrap.com/usage/jekyll-theming.html)，并且通过简单的命令就可以安装，实在是太爽了，于是我毫不犹豫的操作了起来  
__添加远程主题__：
{% codeblock %}
rake theme:install git="https://github.com/jekyllbootstrap/twitter.git"
{% endcodeblock %}
__安装本地主题__：
{% codeblock %}
rake theme:package title="twitter"
{% endcodeblock %}
__切换主题__:
{% codeblock %}
rake theme:switch title="twitter"
{% endcodeblock %}
到这里其实可以结束了，但是总是感觉自带的主题和我自己的审美有冲突
##痛苦挣扎
##失败告终
##总结
