title: Dependency inversion principle in Go
date: 2019-06-06 14:54:25
tags:
---

# 依赖反转原则在Go中使用
## 依赖反转原则

[面向对象设计的设计原则](https://en.wikipedia.org/wiki/SOLID)有五个，分别是:

|首字母|  指代 |   概念|
|--|--|--|
| S |  单一功能原则 |   认为对象应该仅具有一种单一功能的概念。|
| O |  开闭原则     |   认为“软件体应该是对于扩展开放的，但是对于修改封闭的”的概念。|
| L |  里氏替换原则 |   认为“程序中的对象应该是可以在不改变程序正确性的前提下被它的子类所替换的”的概念。  参考契约式设计。|
| I |  接口隔离原则 |   认为“多个特定客户端接口要好于一个宽泛用途的接口” 的概念。|
| D |  依赖反转原则 |   认为一个方法应该遵从“依赖于抽象而不是一个实例” 的概念。 依赖注入是该原则的一种实现方式。|

这五个原则简称: `SOLID`。
在面向对象编程领域中，依赖反转原则（Dependency inversion principle，DIP）是指一种特定的解耦（传统的依赖关系创建在高层次上，而具体的策略设置则应用在低层次的模块上）形式，使得高层次的模块不依赖于低层次的模块的实现细节，依赖关系被颠倒（反转），从而使得低层次模块依赖于高层次模块的需求抽象。
该原则规定：

* 高层次的模块不应该依赖于低层次的模块，两者都应该依赖于抽象接口。
* 抽象接口不应该依赖于具体实现。而具体实现则应该依赖于抽象接口。

该原则颠倒了一部分人对于面向对象设计的认识方式。如高层次和低层次对象都应该依赖于相同的抽象接口。

在传统的应用架构中，低层次的组件设计用于被高层次的组件使用，这一点提供了逐步的构建一个复杂系统的可能。在这种结构下，高层次的组件直接依赖于低层次的组件去实现一些任务。这种对于低层次组件的依赖限制了高层次组件被重用的可行性。

依赖反转原则的目的是把高层次组件从对低层次组件的依赖中解耦出来，这样使得重用不同层级的组件实现变得可能。把高层组件和低层组件划分到不同的包/库（在这些包/库中拥有定义了高层组件所必须的行为和服务的接口，并且存在高层组件的包）中的方式促进了这种解耦。由于低层组件是对高层组件接口的具体实现，因此低层组件包的编译是依赖于高层组件的，这颠倒了传统的依赖关系。众多的设计模式，比如插件，服务定位器或者依赖反转，则被用来在运行时把指定的低层组件实现提供给高层组件。

应用依赖反转原则同样被认为是应用了[适配器模式](https://zh.wikipedia.org/wiki/%E9%80%82%E9%85%8D%E5%99%A8%E6%A8%A1%E5%BC%8F)，例如：高层的类定义了它自己的适配器接口（高层类所依赖的抽象接口）。被适配的对象同样依赖于适配器接口的抽象（这是当然的，因为它实现了这个接口），同时它的实现则可以使用它自身所在低层模块的代码。通过这种方式，高层组件则不依赖于低层组件，因为它（高层组件）仅间接的通过调用适配器接口多态方法使用了低层组件，而这些多态方法则是由被适配对象以及它的低层模块所实现的。

>**前面一大堆其实都是从wiki上copy过来的，自己的理解有以下几点:**
* 上层指**调用者**, 下层指**被调用者**
* 原来的编程方式是上层调用下层的时候依赖下层具体的实现方式
* 依赖反转（或叫:依赖倒置)是指下层的实现依赖上层调用的需求
* 最终的解决方式是: 把上层的需求抽象成接口，上层依赖接口的抽象进行调用，下层依赖接口的抽象进行实现(下面要介绍的**面相接口编程**)


### 依赖注入
[依赖注入](https://zh.wikipedia.org/wiki/%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5)是种实现控制反转用于解决依赖性设计模式。一个依赖关系指的是可被利用的一种对象（即服务提供端） 。依赖注入是将所依赖的传递给将使用的从属对象（即客户端）。该服务是将会变成客户端的状态的一部分。 传递服务给客户端，而非允许客户端来建立或寻找服务，是本设计模式的基本要求。

>**上面这段也是wiki上的, 自己理解:**
* 依赖注入就是: 把下层依赖注入(或叫传递)到上层调用
* 要把提供服务的一方(也就是前面说的: 下层)作为实例传递给客户端(即:上层)
* 不要客户端在内部自己实现服务端的实例化。
* 这种方式的好处是: 可以通过传递不同的实例化对象来实现多态。

### 面相接口编程
[面相接口编程](https://zh.wikipedia.org/wiki/%E5%9F%BA%E4%BA%8E%E6%8E%A5%E5%8F%A3%E7%BC%96%E7%A8%8B)是前面实现依赖反转原则的具体方式。
基于接口的编程将应用程序定义为组件的集合，其中组件间的应用程序接口（API）调用可能只通过抽象化接口完成，而没有具体的类。类的实例化一般通过使用如[工厂模式](https://zh.wikipedia.org/wiki/%E5%B7%A5%E5%8E%82%E6%96%B9%E6%B3%95#%E5%B7%A5%E5%8E%82)等技术的其他接口完成。

> **这里也说一点自己的理解:**
上面说到要通过依赖注入方式传递实例，这个实例如何生成呢？如果每次都生成一个，如果这个实例是有状态的，那么每个拿到的可能都是不一样的，这样就**无法共享**。所以一般都是通过工厂模式产生一个实例，其他调用方要共享的话都通过这个工厂拿到**同一个实例**。


另一种定义描述: 在系统分析和架构中，分清层次和依赖关系，每个层次不是直接向其上层提供服务（即不是直接实例化在上层中），而是通过定义一组接口，仅向上层暴露其接口功能，上层对于下层仅仅是接口依赖，而不依赖具体类

#### 面向接口编程和面向对象编程是什么关系:
首先，面向接口编程和面向对象编程并不是平级的，它并不是比面向对象编程更先进的一种独立的编程思想，而是附属于面向对象思想体系，属于其一部分。或者说，它是面向对象编程体系中的思想精髓之一。

#### 接口的本质

#####  接口是一组规则的集合，它规定了实现本接口的类或接口必须拥有的一组规则。体现了自然界“如果你是……则必须能……”的理念

例如，在自然界中，人都能吃饭，即“如果你是人，则必须能吃饭”。那么模拟到计算机程序中，就应该有一个Person接口，并有一个方法叫Eat()，然后我们规定，每一个表示“人”的类，必须实现Person接口，这就模拟了自然界“如果你是人，则必须能吃饭”这条规则。

从这里，我想各位也能看到些许面向对象思想的东西。面向对象思想的核心之一，就是模拟真实世界，把真实世界中的事物抽象成类，整个程序靠各个类的实例互相通信、互相协作完成系统功能，这非常符合真实世界的运行状况，也是面向对象思想的精髓。

##### 接口是在一定粒度视图上同类事物的抽象表示。注意这里我强调了在一定粒度视图上，因为“同类事物”这个概念是相对的，它因为粒度视图不同而不同

例如，在我的眼里，我是一个人，和一头猪有本质区别，我可以接受我和我同学是同类这个说法，但绝不能接受我和一头猪是同类。但是，如果在一个动物学家眼里，我和猪应该是同类，因为我们都是动物，他可以认为“人”和“猪”都实现了Animal这个接口，而他在研究动物行为时，不会把我和猪分开对待，而会从“动物”这个较大的粒度上研究，但他会认为我和一棵树有本质区别。

#### 面相接口编程的优点

* 首先对系统灵活性大有好处。当下层需要改变时，只要接口及接口功能不变，则上层不用做任何修改。甚至可以在不改动上层代码时将下层整个替换掉。
* 使用接口的另一个好处就是不同部件或层次的开发人员可以并行开工。

> **关于面相接口编程的归纳:**
* 面相接口是面向对象编程的重要部分
* 接口本质上是一组规则的集合，是一定粒度上有相同特指的对象的的抽象
* 面相接口编程可以提高编程的灵活性, 可以并行开发。

## Go 中的应用

### Go 中的接口
Go语言中，接口(interface)有其特殊的地方, 其他的语言一般要实现一个接口都需要显示的说明
例如`PHP`(这里没有贬低PHP的意思，大多数语言也是这种实现方式例如`C++`, `Python`, `Rust`等):
```php
<?php

// Declare the interface 'iTemplate'
interface iTemplate
{
    public function setVariable($name, $var);
    public function getHtml($template);
}

// Implement the interface
// This will work
class Template implements iTemplate
{
    private $vars = array();
  
    public function setVariable($name, $var)
    {
        $this->vars[$name] = $var;
    }
  
    public function getHtml($template)
    {
        foreach($this->vars as $name => $value) {
            $template = str_replace('{' . $name . '}', $value, $template);
        }
 
        return $template;
    }
}

```
用到关键字 `implements`。
todo: 对比优缺点

而`Go`语言中，`interface`是`duck typing`(鸭子类型: If it looks like a duck, and it quacks like a duck, then it is a duck), 也就是如果一个对象实现了某个接口的方法，那么这个对象就是这个接口类型了，不需要显示说明是否实现了某个接口。

```go
// Speaker types can say things.
type Speaker interface {
  Say(string)
}

// Person is a strut with filed name
type Person struct {
  name string
}

// Say funciton is defined by Speaker and implement by Person
func (p *Person) Say(message string) {
  log.Println(p.name+":", message)
}
```
上面`Person`实现了函数`Say`, 所以`Person`就是`Speaker`类型了。





### Go 中面相接口编程
这种编程方式不仅是在 Go 语言中是被推荐的，在几乎所有的编程语言中，我们都会推荐这种编程的方式，它为我们的程序提供了非常强的灵活性，想要构建一个稳定、健壮的 Go 语言项目，不使用接口是完全无法做到的。

如果一个略有规模的项目中没有出现任何 type ... interface 的定义，那么作者可以推测出这在很大的概率上是一个工程质量堪忧并且没有多少单元测试覆盖的项目，我们确实需要认真考虑一下如何使用接口对项目进行重构。

事实上官方库也都是按照这个思想来实现的，比如`net/http`包(对这个包的分析参考 [golang 的 webserver 是如何工作的](/2017/07/01/how-golang-webserver-work/))。当我们要启动一个http server时一般代码如下:
```go
http.ListenAndServe(":8090", nil)
```
这个函数的定义
```go
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```
第二个参数是`Handler`类型, 这个函数的类型定义如下:
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

定义的正是一个接口。这个接口只有一个函数`ServeHTTP`， 而最终对请求处理调用的也正是这个函数:

```go
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }
    handler.ServeHTTP(rw, req)
}
```

由于第二个函数我们一般都会传`nil`, 所以会执行上面的逻辑

```go
handler = DefaultServeMux
```

而`DefaultServeMux`就是官方的默认实现。而我们也可以通过传递这个参数来实现自己的处理, 很多Web框架就是怎么做的，比如`gin`:

```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    c := engine.pool.Get().(*Context)
    c.writermem.reset(w)
    c.Request = req
    c.reset()

    engine.handleHTTPRequest(c)

    engine.pool.Put(c)
}
```

`gin`自己实现了连接的处理方式，并且把这个实现作为参数传给`net/http`, 具体代码如下:

```go
func (engine *Engine) Run(addr ...string) (err error) {
    defer func() { debugPrintError(err) }()

    address := resolveAddress(addr)
    debugPrint("Listening and serving HTTP on %s\n", address)
    err = http.ListenAndServe(address, engine)
    return
}
```
这个实现正是前面说讲的: **依赖接口编程**，然后通过**依赖注入**把实例传入

### 测试
我们有一个`user`包，里面是处理用户相关的信息, 还有一个`bank`包，`bank`会调用`user`的方法来获取一些用户信息, 刚开始他们的代码实现分别如下:
```go
package user

import (
    "database/sql"
)

var db *sql.DB

// UserName return the name of user with uid
func UserName(uid int) (string, error) {
    sql := "select name from user where uid = ?"
    rows, err := db.Query(sql, uid)
    if err != nil {
        return "", err
    }
    defer rows.Close()

    var name string
    for rows.Next() {
        if err := rows.Scan(&name); err != nil {
            return "", err
        }
    }

    return name, nil
}

```

```go
package bank

import (
    "user"
)

// UserInfo if uid exist return the name of user
func UserInfo(uid int) string {
    name, err := user.UserName(uid)
    if err != nil {
        return "something was wrong"
    }
    if name == "" {
        return "not found this user"
    }
    return "user name is " + name
}
```

现在如果我们要给`bank`的`UserInfo`添加单元测试应该怎么做呢? 这里有以下几点问题:
* 我们要测试的是`bank`的`UserInfo`函数，而不是为了测试这个函数都调用的函数，所以我们其实不太关心`user.UserName`里面的逻辑
* 我们要测试`UserInfo`就必须要从`UserName`获取一些信息，但是`UserName`的信息需要调用`db`才能获取，这里涉及到一些网络访问，会给我们的测试带来很多麻烦
* 我们需要把`UserName`给Mock掉
关于如何把`UserName` Mock掉, 其实我们可以借助一些`mock`的框架(比如`bou.ke/monkey`)来进行处理， 但是这种方法回避了设计上的一些问题，过度依赖会导致我们的代码质量堪忧，还有一些场景要求我们必须替换这个方法的实现，比如我们不想使用`user.UserName`的查询方式了，我们换了一种实现，这样我们就无法复用原来的代码了。
下面我们介绍如何利用上面介绍的知识来解决这个问题:

#### 面向接口编程
我们看一下第二版的代码
`user`包:
```go
import (
        "database/sql"
)

var db *sql.DB

type User interface {
        UserName(int) (string, error)
}

type DefaultUser struct{}

func New() User {
        return DefaultUser{}
}

// UserName return the name of user with uid
func (DefaultUser) UserName(uid int) (string, error) {
        sql := "select name from user where uid = ?"
        rows, err := db.Query(sql, uid)
        if err != nil {
                return "", err
        }
        defer rows.Close()

        var name string
        for rows.Next() {
                if err := rows.Scan(&name); err != nil {
                        return "", err
                }
        }

        return name, nil
}
```
`bank`代码:
```go
package bank

import (
    "user"
)

var u = user.New()

// UserInfo if uid exist return the name of user
func UserInfo(uid int) string {
    name, err := u.UserName(uid)
    if err != nil {
        return "something was wrong"
    }
    if name == "" {
        return "not found this user"
    }
    return "user name is " + name
}
```
上面我们都做了那些修改呢?

*  面向接口编程
我们定义了一个接口类型`User`:
```go
type User interface {
        UserName(int) (string, error)
}
```
然后`user`包用`DefaultUser`来实现了这个方法，所以`DefaultUser`就是这个类型的包了
`bank`中定义了一个变量`var u = user.New()`, 由于`user.New()`的类型也是`User`，所以`u`的类型就是`User`, 然后在`UserInfo`函数中调用`User`类型的`UserName`方法
也就是说`user`和`bank`都是面向`User`来进行编程的

* 依赖注入
我们第一个版本是直接调用`user.UserName`函数, 但是我们无法自己去修改这个函数的实现，所以我们通过`var u = user.New()`来获取`user`给我传递的一个对象，这样我们就可以通过`u`来调用`UserName`函数了，这时`user.New`就实现了依赖注入，这样做我们就可以通过覆盖`u`这个实例，来完成自己的实现了，下面

#### 单元测试
面对版本二, 我们怎么实现`bank.UserInfo`的单元测试呢？
`bank_test.go`来看一下:
```go
package bank

import (
    "errors"
    "testing"
)

type mockUser struct{}

func (u mockUser) UserName(uid int) (string, error) {
    if uid == 1 {
        return "", errors.New("something was wrong")
    }
    if uid == 2 {
        return "", nil
    }
    if uid == 3 {
        return "John", nil
    }
    return "", errors.New("something was wrong")
}

func TestUserInfo(t *testing.T) {
    u = mockUser{}
    cases := []struct {
        name string
        uid  int
        res  string
    }{
        {"test1", 1, "something was wrong"},
        {"test2", 2, "not found this user"},
        {"test3", 3, "user name is John"},
    }
    for _, v := range cases {
        t.Run(v.name, func(t *testing.T) {
            info := UserInfo(v.uid)
            if info != v.res {
                t.Errorf("got %s; want %s", info, v.res)
            }
        })
    }
}
```

```
go test -v
=== RUN   TestUserInfo
=== RUN   TestUserInfo/test1
=== RUN   TestUserInfo/test2
=== RUN   TestUserInfo/test3
--- PASS: TestUserInfo (0.00s)
    --- PASS: TestUserInfo/test1 (0.00s)
    --- PASS: TestUserInfo/test2 (0.00s)
    --- PASS: TestUserInfo/test3 (0.00s)
PASS
ok      bank    0.013s
```
首先我们定义了一个`mockUser`, 然后实现了`UserName`函数，所以这时`mockUser`已经是`User`类型了，然后我们在测试函数里通过`u = mockUser`替换掉了运来的`var u = user.New()`, 这时候在执行`UserInfo`调用的其实就是`mockUser.UserName`函数了，完美的完成了我们的单元测试。

#### 工厂模式
前面我们虽然用依赖注入的方式完成了调用，但是还有一个问题, 当我们依赖注入的时候用的是`var u = user.New()`的方式来获取的，但是在错综复杂的调用过程中，我们难免会多次调用`user.New()`函数，而且我们还要共享同一个`User`， 这时候就要求我们使用工厂模式保证不管多少次调用，返回的都是同一个`User`, 在上面的代码中其实很好改:
我们把`user`中的
```go
func New() User {
    return DefaultUser{}
}
```
改为下面的实现:
```go
var defaultUser = DefaultUser{}

func New() User {
    return defaultUser
}
```
这样我们每次返回的都是`user`内部的`defaultUser`这个实例，而这个实例只初始化了一次, 所有通过这个方法获取的实例都是同一个实例

#### 简化调用
有时候我们会觉得每次调用都通过依赖注入的传递一个对象，会使得调用变的复杂起来，比如本来我们调用的时候只需要
```go
user.UserName()
```
而现在可能我们的调用变成了
```
var u = user.New()
u.UserName()
```
那么我们如何使用更符合`go`的方式，直接使用包调用而不是每次都传递一个对象呢？我们可以改为下面的方式:
`user`的实现:
```go
package user

import (
    "database/sql"
)

var db *sql.DB

type User interface {
    UserName(int) (string, error)
}

type DefaultUser struct{}

var defaultUser = DefaultUser{}
var definedUser User

func New() User {
        return defaultUser
}

func SetUser(u User) {
        definedUser = u
}

// UserName return the name of user with uid
func (DefaultUser) UserName(uid int) (string, error) {
        sql := "select name from user where uid = ?"
        rows, err := db.Query(sql, uid)
        if err != nil {
                return "", err
        }
        defer rows.Close()

        var name string
        for rows.Next() {
                if err := rows.Scan(&name); err != nil {
                        return "", err
                }
        }

        return name, nil
}

func UserName(uid int) (string, error) {
        if definedUser == nil {
                return defaultUser.UserName(uid)
        }
        return definedUser.UserName(uid)
}
```
这里我们新增加了一个变量`definedUser`来表示用户自定义的实例，然后通过`SetUser`来对其进行复制，我们同时增加了一个包级别的`UserName`函数，里面的实现会判断如果有`definedUser`那么我们就是用这个自定义的实现，如果没有我们就调用默认的实现

`bank`的实现:

```go
package bank

import (
        "user"
)

// UserInfo if uid exist return the name of user
func UserInfo(uid int) string {
        name, err := user.UserName(uid)
        if err != nil {
                return "something was wrong"
        }
        if name == "" {
                return "not found this user"
        }
        return "user name is " + name
}
```
`bank`的实现跟第一个版本一样，如果我们不需要修改默认实现，对于调用来说非常方便，我们不用关系其内部的具体实现。

`bank_test`的实现:
```go
package bank

import (
    "errors"
    "testing"
    "user"
)

type mockUser struct{}

func (u mockUser) UserName(uid int) (string, error) {
        if uid == 1 {
                return "", errors.New("something was wrong")
        }
        if uid == 2 {
                return "", nil
        }
        if uid == 3 {
                return "John", nil
        }
        return "", errors.New("something was wrong")
}

func TestUserInfo(t *testing.T) {
        user.SetUser(mockUser{})
        cases := []struct {
                name string
                uid  int
                res  string
        }{
                {"test1", 1, "something was wrong"},
                {"test2", 2, "not found this user"},
                {"test3", 3, "user name is John"},
        }
        for _, v := range cases {
                t.Run(v.name, func(t *testing.T) {
                        info := UserInfo(v.uid)
                        if info != v.res {
                                t.Errorf("got %s; want %s", info, v.res)
                        }
                })
        }
}
```
`bank_test`由于要对`UserName`进行Mock, 用自己的实现来替换原来的实现，我们只需要在测试的时候调用`SetUser`函数，就完成了替换。

## 参考

[面向对象设计的设计原则](https://en.wikipedia.org/wiki/SOLID)
[依赖注入](https://zh.wikipedia.org/wiki/%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5)
[面相接口编程](https://zh.wikipedia.org/wiki/%E5%9F%BA%E4%BA%8E%E6%8E%A5%E5%8F%A3%E7%BC%96%E7%A8%8B)
[面向接口编程详解（一）——思想基础](https://www.cnblogs.com/leoo2sk/archive/2008/04/10/1146447.html)
[如何写出优雅的 Golang 代码](https://mp.weixin.qq.com/s/FnKkZD4DdPVX7PPH3dY8-w)
[使用Golang的interface接口设计原则](https://gocn.vip/article/1764)
[Duck typing in Go](https://medium.com/@matryer/golang-advent-calendar-day-one-duck-typing-a513aaed544d)
