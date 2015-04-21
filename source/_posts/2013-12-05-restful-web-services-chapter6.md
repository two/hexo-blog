---
layout: post
title: "chapter6:设计可读写的面向资源的服务"
description: ""
category: "restful web services"
tags: []
---

>*本章主要是讲的如何设计一个可以读写资源的服务。需要写服务的话则需要PUT,POST操作。由于其中涉及到很多具体的例子，本博就不再赘述了，只把看到的重点总结一下。*


* 创建一个资源是用POST还是用PUT要根据上一章的原则进行选择。
* HTML5以前FORM的method只支持GET和POST两种方式，没有PUT。为了能使用PUT可以使用发送WADL片段的方法，还可以通过第八章介绍的“重载POST发送的PUT请求”
* 成功的HTTP的相应代码有可能是201、205等。


