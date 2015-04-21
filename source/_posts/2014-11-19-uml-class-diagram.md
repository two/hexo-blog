---
layout: post
title: "UML类图详解"
description: ""
category: "uml"
tags: []
---

>类图(the class diagram)是UML中很重要的一种静态试图，主要用途是在系统设计环节，描述各个类之间的关系。

##类的表示方法
有面向对象知识的人都知道，类是一类事物的抽象。它封装了数据和行为。在UML中类的表示如下：
{% plantuml %}
skinparam classAttributeIconSize 0
class 类名 {
 -field1-私有
 #field2-受保护
 +field3-公共
 {static} field4-下划线：静态
 ~method1(包)
 /method2(继承)
}
{% endplantuml %}
顶部是类的名字，中间是类的属性，底部是类的方法。

##类成员
UML提供机制，以代表类的成员，如属性和方法，对他们的其他信息。
指定一个类成员的可见性（即任何属性或方法）有下列符号，必须摆在各成员的名字之前。

```
+         公共
-         私有
#         受保护
~         包
/         继承
下划线     静态
```

##类之间的关系
类图主要是表现各个类之间的关系，他们的关系以及表示方式如下：
###外部链接
###继承
继承是面向对象最基本的特性，这个就不再说了，这个设计一般没有什么争议性。关系图如下:
{% plantuml %}
Class01 <|-- Class02
{% endplantuml %}
###实现
实现，是指类(class)实现一个接口(interface)的功能,接口有关的定义在PHP里参见文档[对象接口](http://php.net/manual/zh/language.oop5.interfaces.php)。相当于C++里的抽象类。关系图如下:
{% plantuml %}
InterfaceA <|.. ClassB
InterfaceA: +function1();
ClassB: +function1();
{% endplantuml %}
###依赖
依赖关系（Dependency）可以简单的理解为一个类A使用到了另一个类B，而这种使用关系是具有偶然性、临时性的、非常弱的，但是B类的变化会影响到A；表现在代码层面，为类B作为参数被类A在某个method方法中使用；用带燕尾箭头的虚线表示。
{% plantuml %}
skinparam classAttributeIconSize 0
ClassB <.left. ClassA
ClassA: +depend(ClassB classB):void
{% endplantuml %}
###关联
一个关联（Association）代表一个家族的联系。
关联可以命名，并结束一个关联可以饰以角色名称，所有权指标，多重性，可视性，以及其他属性，如相互关联和有方向的关联。语义上是两个类、或者类与接口之间一种强依赖关系，是一种长期的稳定的关系；表现在代码层面，为被关联类以类属性的形式出现在关联类中，也可能是关联类引用了一个类 型为被关联类的全局变量；目前定义有五种不同类型的关联。双向（Bi-directional）和单向（uni-directional）的关联是最常见的。
下图中展示的关联类A引用了一个类型为被关联类B的全局变量(A关联B);
{% plantuml %}
skinparam classAttributeIconSize 0
ClassB <-left- ClassA
ClassA: -classB:ClassB
{% endplantuml %}
在比如人（person）与杂志(magazine)是一种关联(双向):
{% plantuml %}
skinparam classAttributeIconSize 0
Person "0..*" -- "0..*" Magazine
Person : +subscriber
Magazine : +subscribed magazine
{% endplantuml %}
学生与课程的关系(通过注册关联起来)：
{% plantuml %}
class Student {
  Name
}
Student "0..*" - "1..*" Course
(Student, Course) .. Enrollment

class Enrollment {
  drop()
  cancel()
}
{% endplantuml %}
###聚合
聚合（Aggregate）是表示整体与部分的一类特殊的关联关系，是“弱”的包含（has a）关系，成分类别可以不依靠聚合类别而单独存在，可以具有各自的生命周期，部分可以属于多个整体对象，也可以为多个整体对象共享。例如，池塘与（池塘中的）鸭子。再例如教授与课程就是一种聚合关系。聚集可能不涉及两个以上的类别。图形以空心的菱形与实线来表示。
{% plantuml %}
Family o-left-> Child 
{% endplantuml %}
上面举的例子是家庭和孩子，家庭是整体，孩子是不分，家庭和孩子都是独立的，但是孩子又是家庭的一部分。
###组成
组成（Composition）关系，是一类“强”的整体与部分的包含关(contains a)系。成分类别必须依靠合成类别而存在。整体与部分是不可分的，整体的生命周期结束也就意味着部分的生命周期结束。合成类别完全拥有成分类别，负责创建、销毁成分类别。例如汽车与化油器，又例如公司与公司部门就是一种组成关系。图形以实心的菱形与实线表示。
{% plantuml %}
Brain <-left-* Person
{% endplantuml %}
上面举的例子是人与大脑的关系，大脑是人的组成部分，谁也离不开谁。

> 参考文献:  
[1. UML类图关系（泛化 、继承、实现、依赖、关联、聚合、组合） ](http://blog.csdn.net/zhaoxu0312/article/details/7212152)  
[2. 类别图－维基百科](http://zh.wikipedia.org/wiki/%E9%A1%9E%E5%88%A5%E5%9C%96)
