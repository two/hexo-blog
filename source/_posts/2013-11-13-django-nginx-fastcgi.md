---
layout: post
title: "django+nginx+fastcgi 配置"
description: ""
category: "python"
tags: [django, python]
---

>*我们知道刚开始学习django的时候使用的是django内置的服务器，当然这个是为了能够快速的搭建django的运行环境，并不能运用到实际的生产环境中。而*django book* 这本书中只接受了django+apache和django+lighttpd的配置。但是我们实际的生产环境中nginx使用的很广泛。于是django+nginx+fastcgi的配置方法就有比较了解一下了...*

## django配置
首先*django book*这本书中介绍了如何在生产过程中配置`setting.py`这个文件,其中提到变量`DJANGO_SETTINGS_MODULE`的值其实就是网站的配置文件的路径,我们可以根据环境的不同选择不同的配置文件，例如开发环境、测试环境和线上环境等。这个变量的设置其实就在网站的根目录下`wsgi.py`这个文件中。其内容如下：
{% codeblock %}
"""
wsgi.py
"""
import os

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "djsite.settings_dev")

#  This application object is used by any WSGI server configured to use this
#  file. This includes Django's development server, if the WSGI_APPLICATION
#  setting points here.
from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()
{% endcodeblock %}
这里设置的默认加载配置文件为：`settings_dev.py`，这是开发环境的配置文件,用户可以根据需求选择不同的配置文件。

## uwsig配置文件
{% codeblock %}
$cat uwsig.xml 
<uwsgi>  
    <socket>0.0.0.0:3001</socket>  
    <listen>20</listen>  
    <master>true</master>  
    <pidfile>/etc/nginx/uwsgi.pid</pidfile>  
    <processes>2</processes>  
    <module>wsgi</module>
    <pythonpath>/search/sean/python/djsite/djsite</pythonpath>  
    <profiler>true</profiler>  
    <memory-report>true</memory-report>
    <enable-threads>true</enable-threads>
    <logdate>true</logdate>
    <limit_as>6048</limit_as>
</uwsgi>
{% endcodeblock %}

