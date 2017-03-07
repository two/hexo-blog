---
layout: post
title: "chapter1:Programmable Web及其分类"
description: ""
category: "RESTful Web Services"
tags: []
---

>*本章首先对Programmable Web的定义进行了通俗的介绍，然后有讲了最基础的HTTP协议的相关知识，以及对各种web service架构与HTTP协议的密切关系进行了介绍，让读者对不同的web service架构有了一个初步的认识。*

## Programmable Web的分类
首先要明确是什么事Programmable Web。这里主要是与Human Web进行了对比，下面是对它们的定义：

* Programmable Web: 面向计算机程序使用的Web
* Human Web: 面向人类使用的web

这么说其实很不好理解，但是介绍它们之间的不同之处是就很好理解了：

* Programmable Web:数据主要是供计算机进行读取分析的，对人类不友好的，比如XML，json等数据格式
* Human Web: 主要是供人类读取的，比如HTML页面等

说道Programmable Web的分类，其实现在主要是根据实现的技术（URI, SOAP, XML-RPC等），或者背后的架构与设计思想进行分类的。根据技术进行分类其实存在缺陷，在某些场合容易混淆，只有根据其架构进行分类才是王道。  

## HTTP信封里的文档
HTTP协议作为web的基础，也作为Programmable Web分类的重要标志，需要在这里进行简要的介绍一下（关于HTTP协议的详细介绍请参考《HTTP协议详解》这本书）。

* HTTP方法: 如同编程语言的方法名，表示客户端希望服务器如何处理该信封。
* 路径（path):是URI主机后面的部分，表示信封的地址。
* 请求报头（request headers):是一组k-v对，起到元数据的作用。
* 实体主体（entity-body):是放在信封里的文档。

HTTP相应可分为三个部分：

* HTTP相应代码
* 响应报头
* 实体主体

## 方法信息
客户端发起什么样的请求，对应的服务器端采取何种方式操作数据，这就是方法信息。按照REST的Web服务的做法，增删改查的方法信息都放在了HTTP方法里，分别对应的是POST/DELETE/PUT/GET。有些则是把这些信息放到URI信息里进行传递。还有些是放到实体主体和HTTP报头里，比如SOAP服务请求的方法细节都在WSDL文件里。
## 作用域信息
客户端如何告诉服务器对那些数据进行操作，这种信息就是作用域信息。作用域信息的放置也可以是URI或者SOAP服务的实体主体里。
## 相互竞争的服务架构
常见的web服务架构有三种：REST式架构、RPC式架构和RPC-REST式架构。

**REST式面向资源的架构**  
REST结构意味着,方法信息都在HTTP方法里；面向资源的架构（SOA)意味着，作用域信息都在URI里。REST的具体细节将在后面进行详细的介绍。  
**RPC式架构**  
RPC式web服务通常从客户端收到一个充满数据的信封，然后发回一个同样充满数据的信封。RPC式架构意味着：方法信息和作用域信息都在信封里或报头里。HTTP是一种常见的信封格式，SOAP也是一种常见的信封格式，但是SOAP是通过HTTP进行传输的。  
**REST-RPC式架构**  
混合式的架构是指不是彻底的贯彻REST风格的架构，既有一部分是符合REST架构的要求，又有一部分是RPC架构的要求，这些或者是有意的，或者是无意的。  

