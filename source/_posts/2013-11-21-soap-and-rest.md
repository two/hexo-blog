---
layout: post
title: "SOAP and REST"
description: ""
category: "web"
tags: []
---

>*SOAP(Simple Object Access Protocol)和REST(Representational State Transfer)是现在Web Service开发中最流行的两个架构，这篇文章详细分析了他们的来源以及优缺点。*

作为一个web后台开发人员我竟然拿分不清web service 和 web server!!!自觉面壁~~~
所以我做不住了，我睡不着了，听说了REST后我知道了SOAP，知道SOAP后我知道了WSDL,但是始终百思不得其解，原来是我最基本的东西都不知道，何谈了解他们！所以之能静下心来从头开始了！


## web server与 web service
其实这两个差的太远了，如果不细看还以为是一个东西，因为写法差不多，但是仔细品味一下就知道了:  

* web server是指web服务器，如果apache、nginx等，主要是用来用来处理HTTP服务的底层软件(下次把学到的web server的发展说一下)。
* web service是指基于web的一个服务，是web技术的一个应用。就像你开发一个网站就是一个service，提供一个HTTP的API接口也是service。

其实web service这个东西只要是做web开发或者移动端开发的人都会用到，只不过各有各的用法，各有各的标准。而这里的SOAP和REST只是在众多的应用中提取出来的通用的协议标准。当然拟开发的时候也可以都不用都没有任何关系。  
再看看W3C对Web service的定义：

1. Web Services 是应用程序组件
2. Web Services 使用开放协议进行通信
3. Web Services 是独立的（self-contained）并可自我描述
4. Web Services 可通过使用UDDI来发现
5. Web Services 可被其他应用程序使用
6. XML 是 Web Services 的基础


其实SOAP和REST主要解决的是一个分布式应用的情况，什么是分布式应用呢？我的理解是比如一个MVC结构的网站。一般情况下都会把所有代码放到一台前端机器上，我们开发的时候也是C直接调用M的函数接口。这样对于一般应用是满足的，但是存在两个问题：

1. 网站发展的一定程度后所有的服务压力都在前端机器不能满足业务需求
2. 为了提高性能M采用的是更快速的语言开发的，C采用的是另一种语言，它们之间怎么通信？

针对第一个问题，我们知道M和C的作用不一样，M主要是底层服务，而C主要是业务逻辑相关的服务，网站开发过程中其性能瓶颈主要是集中在M这一层，所以有可能我们需要把M和C完全分离开，M放在单独的底层服务器，C放在前端机，这样就可以各司其职了。但是这样做就会面临第二个问题，它们之间如何通信？比如C需要调用M的某个函数，怎么实现？这就是SOAP和REST发挥作用的地方了，只要C和M都遵循这两个标准开发，整个世界就都清净了~
(详细的过程下面将具体的协议时会说道。)

## RPC协议
说道这两个协议，不得不提一下RPC协议，其实他们都是在RPC的思想上发展起来的~
《TCP/IP协议详解卷一：协议》这本书的第29章网络文件系统介绍了RPC协议。
当终端用户编程的时候其实是不关心网络传输细节的，比如一个C/S架构的程序，Client只需要把网络的数据流当做是一个文件进行操作就可以了，不管是向服务器发送请求还是接受来之服务器的请求都只需调用系统函数就可以实现；同样Server端也是只需要调用系统函数获取客户端发出的请求并且队请求进行处理并把处理结果返回到客户端就行了。C和S之间的交互都是通过网络实现的，但是对于两端来说并无这个过程是透明的。中间的过程其实就是由RPC协议实现的，交互的过程中底层发生的细节入下：  

1. 当客户程序调用远程的过程时，它实际上只是掉哟呵过年了一个位于本机上的、有RPC程序包生成的函数。这个函数被称为客户残桩(stub)。客户残桩将过程的参数封装在一个网络报文，并且将这个报文发送给服务器程序。
2. 服务器主机上的一个服务器残桩负责接收这个网络报文。它从网络报文中提取参数，然后调用应用程序员编写的服务器过程。
3. 当服务器函数返回时，它返回到服务器残桩。服务器残桩提取返回值，把返回值封装成一个网络报文，然后将网络报文发送给客户残桩。
4. 客户残桩从接收到的网络报文中取出返回值，将其返回给客户程序。

一个RPC程序包提供好很多好处：

* 程序设计更加容易，因为很少或几乎没有涉及到网络编程。应用程序设计院只需要编写一个客户程序和客户程序调用的服务器过程。
* 如果使用了一个不可靠的协议，如UDP,像超时和重传等细节一样就由RPC程序包来处理。这就简化了用户应用程序。
* RPC库为参数和返回值的传输提供任何需要的数据转换。

下面这个图片可以展示一下在报文中RPC所处的位置：  
![RPC报文](/assets/img/others/fig1.png)  
![RPC报文](/assets/img/others/fig2.png)  


## SOAP协议
上节说道的RPC协议是应用于网络通信服务，但是HTTP却不是为此设计的，RPC 会产生兼容性以及安全问题；防火墙和代理服务器通常会阻止此类流量。对于web应用来说通过HTTP协议进行通信是更好的方法,因为 HTTP 得到了所有的因特网浏览器及服务器的支持。SOAP 就是被创造出来完成这个任务的。  
SOAP 提供了一种标准的方法，使得运行在不同的操作系统并使用不同的技术和编程语言的应用程序可以互相进行通信。(以上来[W3C](http://www.w3school.com.cn/soap/soap_intro.asp))

Web services 平台的元素：

1. SOAP (简易对象访问协议)
2. UDDI (通用描述、发现及整合)
3. WSDL (Web services 描述语言)

这三个元素是组成SOAP服务必须标准组件，一个完整的SOAP交互过程可以用下面这个图来表示(下图是基于自己的理解画的，如果不妥之处还请指出)：
![SOAP flow](/assets/img/others/fig3.png)  
为了更好地理解SOAP协议，这里举一个例子：用PHP大家SOAP server服务
首先要安装soap服务：
{% codeblock %}
yum install php-soap
{% endcodeblock %}
然后在服务端建立两个文件
user.wsdl
其内容如下：
{% codeblock %}
<?xml version="1.0" encoding="ISO-8859-1"?>
<definitions xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:SOAP-ENC="http://schemas.xmlsoap.org/soap/encoding/" xmlns:tns="http://www.somelocation.com" xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/" xmlns="http://schemas.xmlsoap.org/wsdl/" targetNamespace="http://www.somelocation.com">
<types>
<xsd:schema targetNamespace="http://www.somelocation.com">
 <xsd:import namespace="http://schemas.xmlsoap.org/soap/encoding/" />
 <xsd:import namespace="http://schemas.xmlsoap.org/wsdl/" />
</xsd:schema>
</types>
<message name="userDataRequest">
  <part name="operation" type="xsd:string" />
  <part name="statement" type="xsd:string" />
</message>
<message name="userDataResponse">
  <part name="return" type="xsd:string" />
</message>
<portType name="userWsdlPortType">
  <operation name="userData">
    <documentation>Query User Data</documentation>
    <input message="tns:userDataRequest" />
    <output message="tns:userDataResponse" />
  </operation>
</portType>
<binding name="userWsdlBinding" type="tns:userWsdlPortType">
  <soap:binding style="rpc" transport="http://schemas.xmlsoap.org/soap/http" />
  <operation name="userData">
    <soap:operation soapAction="http://www.somelocation.com#feelbad" style="rpc" />
    <input><soap:body use="encoded" namespace="http://www.somelocation.com" encodingStyle="http://schemas.xmlsoap.org/soap/encoding/" /></input>
    <output><soap:body use="encoded" namespace="http://www.somelocation.com" encodingStyle="http://schemas.xmlsoap.org/soap/encoding/" /></output>
  </operation>
</binding>
<service name="userWsdl">
  <port name="userWsdlPort" binding="tns:userWsdlBinding">
    <soap:address location="http://www.soap.com/service.php" />
  </port>
</service>
</definitions>
{% endcodeblock %}
然后在服务端建立两个文件

service.php
内容如下：
{% codeblock %}
<?php

//设置不缓存wsdl
ini_set("soap.wsdl_cache_enabled","0");

//初始化wsdl服务
$server = new SoapServer("user.wsdl");

//主功能性的class，这里可以分离出来写各种复杂的逻辑
class USER {

    function getInfo($userId) {
        return json_encode(array('userId'=>$userId,'userName'=>'Zhang San'));
    }   

    function getGroup($userId) {
        return json_encode(array('userId'=>$userId,'userGroup'=>'111'));
    }   

}

//接口的主入口函数
function userData($operation,$statement){
    return USER::$operation($statement);
}

//注册主函数
$server->AddFunction("userData");

//启动soap server
$server->handle();

?>
{% endcodeblock %}

然后在客户端建立一个访问的文件
client.php
{% codeblock %}
<?php

# $client = new SoapClient('http://www.soap.com/service.php?wsdl');
$client = new SoapClient('http://www.soap.com/service.php?wsdl');
$res1 = $client->__soapCall('userData',array('operation'=>'getInfo','statement'=>'111'));
print_r($res1);
?>
{% endcodeblock %}

上面这个例子客户端和服务端都是PHP写的，其实他们可以是不同的语言，只需要这个语言支持SOAP协议即可。
根据以上的例子我相信你已经能对SOAP有个清晰的定义了。根据SOAP的特点可以知道SOAP对于分布式的服务和跨语言的通信提供了一个很好的方案，但是就不足的就是它需要额外定义的SOAP协议，WSDL描述性文件等，具有一定的复杂性。随着HTTP协议的发展，它本身其实已经具有了独立承担这个服务的能力了，利用HTTP协议本身我们能够做得更好，这就是下面要讲的REST协议。

## REST协议
在读这个之前请先移步到[理解本真的REST架构风格](http://www.infoq.com/cn/articles/understanding-restful-style)这篇文章
