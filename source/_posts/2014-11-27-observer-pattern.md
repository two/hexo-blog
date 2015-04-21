---
layout: post
title: "设计模式-观察者模式"
description: "观察者模式"
category: "设计模式"
tags: []
---

>在学习js事件的时候，说js事件其实就是观察者模式，所以打算先把观察者模式了解一下，这样就能更好的理解事件机制了。

由于观察者模式，其实比较好理解，这里就举[wikipedia的例子](http://zh.wikipedia.org/wiki/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F)
观察者模式的UML类图如下:

{% plantuml %}
class Observer {
+notify()
}
class Subject {
+addObserver()
+deleteObserver()
+notifyObserver()
}
class ConcreteObserverA {
+notify();
}
class ConcreteObserverB {
+notify();
}
class ConcreteSubjectA {
+notifyObserver()
}
class ConcreteSubjectB {
+notifyObserver()
}
ConcreteObserverA -up-|> Observer
ConcreteObserverB -up-|> Observer
ConcreteSubjectA -up-|> Subject
ConcreteSubjectB -up-|> Subject
Observer --o Subject
ConcreteSubjectA ..> ConcreteObserverA
ConcreteSubjectB ..> ConcreteObserverB

{% endplantuml %}

分析一下这个类图。主要分为两种类。一个是观察者类(Observer), 一个是被观察者类(Subject), 被观察者类有三种方法分别是：

* `addObserver()`: 添加观察者到被观察者类中
* `deleteObserver()`: 从被观察者类中删除观察者
* `notifyObserver()`: 通知已经注册过的观察者

这里`ConcreteSubjectA`和`ConcreteSubjectB`是继承`Subject`的。当有多个被观察者类时可以用这种方式，当只有一个被观察者类时，可以直接用`Subject`进行实例话，不需要派生类。
`Observer`是观察者的抽象类。所有的观察者都继承自这个类。每个观察者类都要实现一个`notify()`函数。这个函数是当被观察者发生变化需要通知观察者的时候调用的，也就是`notifyObserver()`调用的。


##代码事例:
```php
class Student implements SplObserver{
    protected $tipo = "Student";
    private $nome;
    private $endereco;
    private $telefone;
    private $email;
    private $_classes = array();
 
    public function GET_tipo() {
        return $this->tipo;
    }
 
    public function GET_nome() {
        return $this->nome;
    }
 
    public function GET_email() {
        return $this->email;
    }
 
    public function GET_telefone() {
        return $this->nome;
    }
 
    function __construct($nome) {
        $this->nome = $nome;
    }
 
    public function update(SplSubject $object){
        $object->SET_log("Comes from ".$this->nome.": I'm a student of ".$object->GET_materia());
    }
 
}

class Teacher implements SplObserver{
    protected $tipo = "Teacher";
    private $nome;
    private $endereco;
    private $telefone;
    private $email;
    private $_classes = array();
 
    public function GET_tipo() {
        return $this->tipo;
    }
 
    public function GET_nome() {
        return $this->nome;
    }
 
    public function GET_email() {
        return $this->email;
    }
 
    public function GET_telefone() {
        return $this->nome;
    }
 
    function __construct($nome) {
        $this->nome = $nome;
    }
 
    public function update(SplSubject $object){
        $object->SET_log("Comes from ".$this->nome.": I teach in ".$object->GET_materia());
    }
}
class Subject implements SplSubject {
    private $nome_materia;
    private $_observadores = array();
    private $_log = array();
 
    public function GET_materia() {
        return $this->nome_materia;
    }
 
    function SET_log($valor) {
        $this->_log[] = $valor ;
    }
    function GET_log() {
        return $this->_log;
    }
 
    function __construct($nome) {
        $this->nome_materia = $nome;
        $this->_log[] = " Subject $nome was included";
    }
    /* Adiciona um observador */
    public function attach(SplObserver $classes) {
        $this->_classes[] = $classes;
        $this->_log[] = " The ".$classes->GET_tipo()." ".$classes->GET_nome()." was included";
    }
 
    /* Remove um observador */
    public function detach(SplObserver $classes) {
        foreach ($this->_classes as $key => $obj) {
            if ($obj == $classes) {
                unset($this->_classes[$key]);
                $this->_log[] = " The ".$classes->GET_tipo()." ".$classes->GET_nome()." was removed";
            }
        }
    }
 
    /* Notifica os observadores */
    public function notify(){
        foreach ($this->_classes as $classes) {
            $classes->update($this);
        }
    }
}

$subject = new Subject("Math");
$marcus = new Teacher("Marcus Brasizza");
$rafael = new Student("Rafael");
$vinicius = new Student("Vinicius");
 
// Include observers in the math Subject
$subject->attach($rafael);
$subject->attach($vinicius);
$subject->attach($marcus);
 
$subject2 = new Subject("English");
$renato = new Teacher("Renato");
$fabio = new Student("Fabio");
$tiago = new Student("tiago");
 
// Include observers in the english Subject
$subject2->attach($renato);
$subject2->attach($vinicius);
$subject2->attach($fabio);
$subject2->attach($tiago);
 
// Remove the instance "Rafael from subject"
$subject->detach($rafael);
 
// Notify both subjects
$subject->notify();
$subject2->notify();
 
echo "First Subject <br />";
echo "<pre>";
print_r($subject->GET_log());
echo "</pre>";
echo "<hr>";
echo "Second Subject <br />";
echo "<pre>";
print_r($subject2->GET_log());
echo "</pre>";
```
输出结果:

```php

//First Subject

Array (

   [0] =>  Subject Math was included
   [1] =>  The Student Rafael was included
   [2] =>  The Student Vinicius was included
   [3] =>  The Teacher Marcus Brasizza was included
   [4] =>  The Student Rafael was removed
   [5] => Comes from Vinicius: I'm a student of Math
   [6] => Comes from Marcus Brasizza: I teach in Math
)



//Second Subject

Array (

   [0] =>  Subject English was included
   [1] =>  The Teacher Renato was included
   [2] =>  The Student Vinicius was included
   [3] =>  The Student Fabio was included
   [4] =>  The Student tiago was included
   [5] => Comes from Renato: I teach in English
   [6] => Comes from Vinicius: I'm a student of English
   [7] => Comes from Fabio: I'm a student of English
   [8] => Comes from tiago: I'm a student of English
)

```

结合代码我们再具体分析。上面的代码中有`SplObserver`是观察者基类,观察者类`Student`和`Teacher`都继承它。被观察者`Subject`继承自`SplSubject`基类,实现了三个函数:添加观察者的`attach`、删除观察者的`detach`及通知观察者的`notify`。我们还注意到每个观察者类都有一个`update`函数，这个函数是被观察者通知观察者时的接口，由`notify`函数调用并传递被观察者的信息到观察者。这就是一个完整的观察者模式的代码实例。

我们再看如何使用:
开始的时候初始化观察者和被观察者类:

```php
$subject = new Subject("Math");
$marcus = new Teacher("Marcus Brasizza");
$rafael = new Student("Rafael");
$vinicius = new Student("Vinicius");
```

然后把观察者注册到被观察者中:

```php
$subject->attach($rafael);
$subject->attach($vinicius);
$subject->attach($marcus);
```

接着又把`$rafael`从观察者类中删除了:

```php
$subject->detach($rafael);
```

最后由被观察者通知注册的观察者做出改变:

```php
$subject->notify();
```

所以最后的结果如下:

```php
Array (

   [0] =>  Subject Math was included
   [1] =>  The Student Rafael was included
   [2] =>  The Student Vinicius was included
   [3] =>  The Teacher Marcus Brasizza was included
   [4] =>  The Student Rafael was removed
   [5] => Comes from Vinicius: I'm a student of Math
   [6] => Comes from Marcus Brasizza: I teach in Math
)
```

第二个分析方法相同,不在赘述了。
