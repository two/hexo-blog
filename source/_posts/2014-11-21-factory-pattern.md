---
layout: post
title: "设计模式-工厂模式"
description: "设计模式"
category: "设计模式"
tags: []
---

>工厂模式如果细分的话可以分为工厂方法，简单工厂模式和抽象工厂模式,本文对其进行一一介绍。

##简单工厂模式
首先先举一个例子
需求：创建一个按钮，在不同的系统上有不同的风格.
分析，如果不用模式：

```php
class Draw {
    public function drawButton($type)
    {
        if($type == 'mac') {
            button = new MacButton();  
        } else($type == 'win') {
            button = new WimButton();
        }

        button->draw();
    }
}
```

可以看出，这段代码也很简洁，但是我们只考虑了两种系统，当增加系统时（比如ubuntu ,centos, redhat等) 我们就会不停的添加判断分之：

```php 
        if($type == 'mac') {
            button = new MacButton();  
        } else($type == 'win') {
            button = new WinButton();
        } else ($type == 'ubuntu') {
            ...
        }
            ...

```

这个条件语句就会变的不可控。为了使代码看起来更简洁，并且添加分类更容易可以进行如下改造:

```php
class Draw {
    public function drawButton($type)
    {
        button = ButtonFactory::create($type);
        button->draw();
    }
}
class ButtonFactory(){
    static public function create($type) {
        if($type == 'mac') {
            button = new MacButton();  
        } else($type == 'win') {
            button = new WinButton();
        } else ($type == 'ubuntu') {
            ...
        }
        return button;
    }
}
```

新增了一个ButtonFactory类，负责创建button对象使对象的创建和使用进行了分离。其实PHP还可以这么写:

```
class ButtonFactory(){
    static public function create($type) {
            button = new $type."Button"();  
    }
}
```
用UML类图表示他们之间的关系，就是这样的:

{% plantuml %}
skinparam classAttributeIconSize 0
class ButtonFactory {
+ create()
}
class Button {
+ draw()
}
ButtonFactory .down.> WinButton
ButtonFactory .down.> MacButton
WinButton -up-|> Button
MacButton -up-|> Button
Draw .right.> ButtonFactory
Draw .right.> Button
{% endplantuml %}
ButtonFactory是工厂类,负责创建所需的产品类。Button时产品类，WinButton等继承Button。 Draw 是调用的类用到了工厂和产品类，与它们是关联的关系。

##工厂方法
需求：还是上面的需求，随着操作系统类型的增加，我们会发现这个工厂类还是需要修改和维护，这也很容易冗余而产生问题。这时工厂方法应运而生。先看UML图:
{% plantuml %}
skinparam classAttributeIconSize 0
class ButtonFactory {
+create()
}
class WinButtonFactory {
+create()
}
class MacButtonFactory {
+create()
}
class Button {
+draw()
}
class WinButton {
+draw()
}
class MacButton {
+draw()
}
class Draw {
} 
WinButtonFactory -up-|> ButtonFactory
MacButtonFactory -up-|> ButtonFactory
WinButtonFactory .left.> WinButton
MacButtonFactory .left.> MacButton
WinButton -up-|> Button
MacButton -up-|> Button
Draw ..> Button
Draw ..> ButtonFactory
{% endplantuml %}

工厂方法是针对每一种产品提供一个工厂类。通过不同的工厂实例来创建不同的产品实例。这种进一步抽象化的结果，使这种工厂方法模式可以用来允许系统在不修改具体工厂角色的情况下引进新的系统，这一特点无疑使得工厂方法模式具有超过简单工厂模式的优越性。
代码事例:

```php
class ButtonFactory {
    abstruct public static function create();
}
class WinButtonFactory {
    public static function create() {
        return new WinButton();
    }
}
class MacButtonFactory {
    public static function create() {
        return new MacButton();
    }
}
class Button() {
abstruct public function draw();
}

class WinButton extends Button{
    public function draw() {
        echo "draw win button";
    }
}
class MacButton extends Button{
    public function draw() {
        echo "draw mac button";
    }
}

class Draw {
winButton = WinButtonFactory::create();
WinButton->draw();
macButton = MacButtonFactory::create();
MacButton->draw();
}
```
但是工厂方法存在一个问题，那就是我们如果事先不知道会需要哪一种类，需要根据情况来选择。那就又回到了第一种情况：

```php
class Draw {
    if($type == "win") {
        $button = WinButtonFactory::create();
    } else if ($type == "mac") {
        $button = MacButtonFactory::create();
    }
    ...
    button->draw();
}
```
这时候就需要综合简单工厂和工厂方法两种模式，做出一个混合体：再创建一个简单工厂类，用来根据参数调用工厂方法创建对象。

##抽象工厂
现在需求又复杂化了: 这次不光要创建按钮，还要画边框(border)，画输入框(input)。
让我们再来仔细分析一下需求: 要完成一个画图的程序，这个程序要适应不同的操作系统，这个程序又有许多种元素需要画(比如按钮，边框，输入框)。这里需要注意的是，以前按钮只是一个函数，现在需要多个调用。
“工厂”是创建产品（对象）的地方，其目的是将产品的创建与产品的使用分离。抽象工厂模式的目的，是将若干抽象产品的接口与不同主题产品的具体实现分离开。这样就能在增加新的具体工厂的时候，不用修改引用抽象工厂的客户端代码。
{% plantuml %}
class Factory {
+createButton();
+createBorder();
+createInput();
}
class WinFactory {
+createButton();
+createBorder();
+createInput();
}
class MacFactory {
+createButton();
+createBorder();
+createInput();
}
class Button{
}
class WinButton{
}
class MacButton{
}
class Border{
}
class WinBorder{
}
class MacBorder{
}
class Input{
}
class WinInput{
}
class MacInput{
}
class Draw{
}
WinButton -up-|> Button
MacButton -up-|> Button
WinBorder -up-|> Border
MacBorder -up-|> Border
WinInput -up-|> Input
MacInput -up-|> Input
WinFactory -up-|> Factory
MacFactory -up-|> Factory
WinFactory ..> WinButton
WinFactory ..> WinBorder
WinFactory ..> WinInput
MacFactory ..> MacButton
MacFactory ..> MacBorder
MacFactory ..> MacInput
Draw ..> WinFactory
Draw ..> MacFactory
{% endplantuml %}

##总结

* 简单工厂模式: 把创建对象的方法进行封装，使其使用和创建进行分离.但是每增加一个类型都要修改工厂类.
* 工厂方法: 对每个对象创建都建立一个工厂类，当有类型增加时只需要再创建一个工厂类，不需要修改原来的代码。但是当不确定使用的工厂是哪个时，需要结合简单工厂进行选择。
* 抽象工厂: 每个类型出了本身是一个类外，其要完成的功能比较复杂，是多个类的组合。这时就要对每个使用到的类都进行抽象，根据使用类的不同进行选择，其实是工厂方法的复杂版本。
