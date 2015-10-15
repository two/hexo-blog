layout: title
title: "mean primer"
date: 2015-08-14 14:57:05
tags: [MEAN, angularjs, express, nodejs, moongodb]
---

# MEAN入门 

## MEAN简介
### 什么是MEAN?
根据[官方文档](http://meanjs.org/), MEAN就是MongoDB +  Express +  AngularJS +  Node.js的组合。那么组成MEAN的各个部分又分别是什么? 

* `MongoDB`: 是一个基于分布式文件存储的NoSQL数据库。具体介绍和使用方法请参考[官方文档](http://docs.mongodb.org/manual/core/introduction/)
* `Express`: 是一个简洁、灵活的Node.js Web应用开发框架。其实和PHP的MVC框架作用是一样的。详细介绍参见[官方文档](http://expressjs.com/)
* `AngularJS`: 前端的MVC框架，更接近于 MVVM（Model-View-ViewModel)。具体介绍参考[官方文档](https://angularjs.org/)
* `Node.js`: javascript的一个解析器，提供js在服务器端的运行环境。[官方网站](https://nodejs.org/)

### 为什么是这个组合?
大家已经熟悉了LAMP/LNMP的开发模式，这些开发模式已经能够满足了现在web开发的绝大部分需求。而新型的MEAN开发模式则是另外一个尝试,其目的是为了解决现在开发中的一些问题，是开发更加高效。总结起来主要有以下几个优点:

* Web服务器包含在了应用程序中，可以自动安装，部署过程得到了极大简化。
* 从传统数据库到NoSQL再到以文档为导向的持久存储MongoDB，使用户花费在编写SQL上的时间减少，有更多的时间编写js中的映射/化简功能。还能节省大量的转换逻辑(因为MongoDB存储的时JSON对象，js可以直接用)
* 得益于AngularJS,从传统的服务器端页面变为客户端单页面应用程序越来越方便。

>以上内容参考来源:[http://www.ibm.com/developerworks/cn/web/wa-mean1/](http://www.ibm.com/developerworks/cn/web/wa-mean1/)

另外，任何开发模式都不是万能的，也就是没有银弹，这种开发模式可以給大家带来很多新的思想,开拓思路,对大家以后应对不同应用场景的需求是提供更多的参考。

## MEAN安装

MEAN只是一个组合，可以自己单独安装配置各个模块，也有现成的集成方案，如[meanjs](http://meanjs.org/)和[mean.io](http://mean.io/)(关于他们之间的区别可以参考stackoverflow上面的[讨论](http://stackoverflow.com/questions/23199392/difference-between-mean-js-and-mean-io))。这里我们选择的是meanjs作为开发框架。


meanjs的安装可以参考[官方文档](http://meanjs.org/docs.html#getting-started)。这里需要提前介绍一下安装时用到的一些工具及安装遇到的问题和解决方案。

### 常用工具

* `npm`: 是Node Package Manage的简称,Node.js的包管理工具,它的主要功能就是管理node包，包括：安装、卸载、更新、查看、搜索、发布等。这个类似于centos系统上的yum工具. 可以通过package.json对npm进行配置。可以访问[官网](https://www.npmjs.com/)查看相关文档，也可以编写自己的npm包提交上去。(安利一下我写的一个很简单的包:[https://www.npmjs.com/package/hexo-tag-plantuml](https://www.npmjs.com/package/hexo-tag-plantuml))
* `bower`: 也是包管理工具,由twitter推出.他和npm的区别是npm针对服务端的工具进行管理，bower则是主要管理前端页面的js依赖关系。通过bower.json和.bowerrc进行配置.
* `grunt`: 构建javascript的工具,可以自动的完成代码规范的检查，文件合并，文件压缩，单元测试等流程(参考这边文档[grunt从入门到自定义项目模板](http://www.cnblogs.com/chyingp/archive/2013/05/11/grunt_getting_started.html)).详细信息参考[官网](http://gruntjs.com/)。

### 安装流程
这部分上面提到的meanjs官网有详细的步骤，简单概况一下就是:

1. 安装Node.js&npm, MongoDB, Bower, Grunt等
2. 下载源码: `git clone https://github.com/meanjs/mean.git meanjs`
3. 进入meanjs目录，执行`npm install ; bower install`,执行bower使用会出现下面的提示:

    ```
    Since bower is a user command, there is no need to execute it with superuser permissions.
    If you're having permission errors when using bower without sudo, please spend a few minutes learning more about how your system should work and make any necessary repairs.
    
    http://www.joyent.com/blog/installing-node-and-npm
    https://gist.github.com/isaacs/579814
    
    You can however run a command with sudo using --allow-root option
    ```
    需要通过`bower install --allow-root`命令来执行安装。


只需要这些，一个完成的网站就建成了。meanjs自带了一个博客登陆体系和博客浏览发布的功能。
在根目录下运行`grunt`命令就可以启动服务器了，默认的端口是3000，我们可以通过ip:3000的方式来访问这个网站。

## meanjs结构简介

首先进入根目录可以看到如下的文件内容:
```
[@tc_214_162 meanjs]# tree -aL 1
.
├── app                     #后端MVC的内容目录
├── bower.json              #bower配置包管理的文件
├── .bowerrc                #配置安装路径
├── config                  #相关配置目录
├── .csslintrc
├── Dockerfile
├── .editorconfig
├── fig.yml                 
├── .git
├── .gitignore
├── gruntfile.js           #grunt相关配置
├── .jshintrc
├── karma.conf.js
├── LICENSE.md
├── node_modules          #node模块的安装目录
├── package.json          #npm包管理配置
├── Procfile
├── public                #前端内容
├── README.md
├── scripts               #独立脚本目录
├── server.js             #服务运行的入口文件
├── .slugignore
└── .travis.yml
```
后端MVC的主要结构如下:
```
app/
├── controllers    #C层 
├── models         #M层 
├── routes         #路由规则
├── tests
└── views          #V层
```
前端主要结构如下:
```
public/
├── application.js        #应用入口
├── config.js             #应用配置
├── humans.txt
├── lib                   #angular相关库文件
├── modules               #angular不同模块
└── robots.txt
```
angular的模块结构如下:
```
public/modules/
├── articles
│   ├── articles.client.module.js
│   ├── config
│   ├── controllers         #angular的C
│   ├── services            #angular的服务层
│   ├── tests
│   └── views               #V层
├── core
│   ├── config
│   ├── controllers
│   ├── core.client.module.js
│   ├── css
│   ├── img
│   ├── services
│   ├── tests
│   └── views
└── users
    ├── config
    ├── controllers
    ├── css
    ├── img
    ├── services
    ├── tests
    ├── users.client.module.js
    └── views
```
要相对上面的结构有清晰的了解，必须对熟悉各个模块的用法，还要了解一个页面从访问到展现的流程是怎么样。 下面通过一个页面的访问流程来对整个架构的工作流程有一个大概的认识。

## 打开一个页面的流程
为了便于我们假设你已经注册登录并创建了几篇文章，下面我们就依据对文章列表页的打开流程进行介绍。

1. 首先通过menu进入文章列表页:`http://localhost:3000/#!/articles`我们可以看到文章的列表。通过观察这个url可以看出，其实`#`是一个锚点，后台的部分只是hash参数，前面的才是真正的url,也就是我们其实访问的是根目录，通过访问日志也可以看出来:
```
GET / 200 26.246 ms - -
GET /modules/core/css/core.css 200 31.710 ms - 354
GET /modules/users/css/users.css 200 26.138 ms - 211
GET /lib/bootstrap/dist/css/bootstrap-theme.css 200 41.393 ms - -
GET /lib/angular-resource/angular-resource.js 200 15.583 ms - -
GET /lib/bootstrap/dist/css/bootstrap.css 200 81.453 ms - -
GET /lib/angular-animate/angular-animate.js 200 36.241 ms - -
GET /lib/angular-ui-utils/ui-utils.js 200 22.724 ms - -
GET /lib/angular-bootstrap/ui-bootstrap-tpls.js 200 28.636 ms - -
GET /lib/angular-ui-router/release/angular-ui-router.js 200 44.805 ms - -
GET /config.js 200 24.392 ms - 791
GET /application.js 200 37.393 ms - 1016
GET /modules/articles/articles.client.module.js 200 30.888 ms - 133
GET /modules/core/core.client.module.js 200 24.737 ms - 129
GET /modules/users/users.client.module.js 200 18.616 ms - 129
GET /modules/articles/config/articles.client.config.js 200 13.114 ms - 389
GET /lib/angular/angular.js 200 161.376 ms - -
GET /modules/articles/config/articles.client.routes.js 200 36.936 ms - 700
GET /modules/articles/services/articles.client.service.js 200 25.093 ms - 295
GET /modules/core/config/core.client.routes.js 200 19.327 ms - 384
GET /modules/core/controllers/header.client.controller.js 200 13.791 ms - 495
GET /modules/articles/controllers/articles.client.controller.js 200 34.063 ms - -
GET /modules/core/controllers/home.client.controller.js 200 43.584 ms - 224
GET /modules/users/config/users.client.config.js 200 31.473 ms - 708
GET /modules/core/services/menus.client.service.js 200 39.634 ms - -
GET /modules/users/config/users.client.routes.js 200 27.816 ms - -
GET /modules/users/controllers/authentication.client.controller.js 200 22.157 ms - -
GET /modules/users/controllers/password.client.controller.js 200 17.350 ms - -
GET /modules/users/controllers/settings.client.controller.js 200 21.926 ms - -
GET /modules/users/services/authentication.client.service.js 200 15.335 ms - 202
GET /modules/users/services/users.client.service.js 200 9.742 ms - 244
GET /lib/bootstrap/dist/css/bootstrap-theme.css.map 200 8.378 ms - 47721
GET /lib/bootstrap/dist/css/bootstrap.css.map 200 49.468 ms - 390518
GET /modules/articles/views/list-articles.client.view.html 200 8.756 ms - 819
GET /modules/core/views/header.client.view.html 200 17.955 ms - -
GET /articles 200 24.296 ms - 407
GET /modules/core/img/brand/favicon.ico 200 12.350 ms - 32038
```
2. 请求到达之后首先会根据`app/routes`目录下面的路由规则进行匹配，`app/routes/core.server.routes.js`匹配到了这个路由,其内容入下:
```
module.exports = function(app) {
    // Root routing
    var core = require('../../app/controllers/core.server.controller');
    app.route('/').get(core.index);
};
```
可以看出这个请求匹配到后交给了`core.index`进行处理,`app/controllers/core.server.controller.js`内容如下:
```
exports.index = function(req, res) {
    res.render('index', {
        user: req.user || null,
        request: req
    });
};
```
index函数并接收到请求后对`'index'`模板进行了渲染,模板文件`app/views/index.server.view.html`内容如下:
```
{% extends 'layout.server.view.html' %}
{% block content %}
    <section data-ui-view></section>
{% endblock %}
```
这个模板继承了`layout`模板，并且重写了content的block内容。
我们再看一下被继承的模板`app/views/layout.server.view.html`其主要内容入如下:
```
<header data-ng-include="'/modules/core/views/header.client.view.html'" class="navbar navbar-fixed-top navbar-inverse"></header>
    <section class="content">
        <section class="container">
            {% block content %}{% endblock %}
        </section>
    </section>

    <!--Embedding The User Object-->
    <script type="text/javascript">
        var user = {{ user | json | safe }};
    </script>

    <!--Application JavaScript Files-->
    {% for jsFile in jsFiles %}
        <script type="text/javascript" src="{{jsFile}}"></script>
    {% endfor %}

    {% if process.env.NODE_ENV === 'development' %}
    <!--Livereload script rendered -->
    <script type="text/javascript" src="http://{{request.hostname}}:35729/livereload.js"></script>
    {% endif %}
```
上面的主要功是加载页面所需的js文件，这些文件的配置都在`config`里。根据上面的访问日志可以看出主要`public`下的js文件都被加载了。
到这里服务端的工作已经完成一半了。
3. 前端js加载后，angular就开始发挥作用了。angular也是一套MVC框架。在进入流程之前我们再次观察一下URL。我们打开主页时会发现url变成了`http://localhost:3000/#!/`。为什么url会自动加上这部分内容，又为什么需要加这部分内容呢？
根据官方文档[$locaiton](https://docs.angularjs.org/guide/$location) Hashbang and HTML5 Modes部分的介绍,这应该是和浏览器对history支持的兼容性有关系。具体介绍可以看文档。 angualr的路由匹配规则其实是从`#!`之后开始的。再回到本页，`public/modules/articles/config/articles.client.routes.js`匹配到了当前规则代码如下：
```
state('listArticles', {
url: '/articles',
templateUrl: 'modules/articles/views/list-articles.client.view.html'
}).
```
可以看出当匹配时，angualr会自动加载`templateUrl`到页面片上来，在观察一下被加载的页面片`public/modules/articles/views/list-articles.client.view.html`内容如下:
```
<section data-ng-controller="ArticlesController" data-ng-init="find()">
    <div class="page-header">
        <h1>Articles</h1>
    </div>
    <div class="list-group">
        <a data-ng-repeat="article in articles" data-ng-href="#!/articles/{{article._id}}" class="list-group-item">
            <small class="list-group-item-text">
                Posted on
                <span data-ng-bind="article.created | date:'mediumDate'"></span>
                by
                <span data-ng-bind="article.user.displayName"></span>
            </small>
            <h4 class="list-group-item-heading" data-ng-bind="article.title"></h4>
            <p class="list-group-item-text" data-ng-bind="article.content"></p>
        </a>
    </div>
    <div class="alert alert-warning text-center" data-ng-if="articles.$resolved && !articles.length">
        No articles yet, why don't you <a href="/#!/articles/create">create one</a>?
    </div>
</section>
```
页面加载后执行`find()`函数，这个函数在控制层文件`public/modules/articles/controllers/articles.client.controller.js`里可以看到:
```
$scope.find = function() {
    $scope.articles = Articles.query();
};
```
这个函数调用了`Articels.query()`方法，这个方法是Angualr的一个注册的service(参考文档[Services](https://docs.angularjs.org/guide/services)), 位于文件`public/modules/articles/services/articles.client.service.js`中，代码如下:
```
//Articles service used for communicating with the articles REST endpoints
angular.module('articles').factory('Articles', ['$resource',
    function($resource) {
        return $resource('articles/:articleId', {
            articleId: '@_id'
        }, {
            update: {
                method: 'PUT'
            }
        });
    }
]);
```
看到并没有定义`query`方法,是因为这方法是`$resource`默认的(参考[$resource](https://docs.angularjs.org/api/ngResource/service/$resource)),代码中`articleId`是参数，本次查询并没有传递参数，所以实际访问的url是`/articles`,这个是一个RESTful接口，返回的结果赋值給`$scope.articles`,就可以在前端正常展示文章列表了。
4. 第3步的最后提到了访问`/articles`接口，这个接口的作用就是从数据库取数据然后在返回到前端。当访问接口时，服务器接到请求,文件`app/routes/articles.server.routes.js`匹配到路由规则：
```
    app.route('/articles')
        .get(articles.list)
        .post(users.requiresLogin, articles.create);
```
由于是get方法，所以需求转给了`articles.list`方法进行处理，`app/controllers/articles.server.controller.js`中`list`方法代码如下:
```
exports.list = function(req, res) {
    Article.find().sort('-created').populate('user', 'displayName').exec(function(err, articles) {
        if (err) {
            return res.status(400).send({
                message: errorHandler.getErrorMessage(err)
            });
        } else {
            res.json(articles);
        }
    });
};
```
其中`Articles`这个对象是这个文件前面定义的：
```
Article = mongoose.model('Article')
```
其中`mongoose`是MongoDB的一个js封装库,这个module是在`app/models/article.server.model.js`下定义并注册的，代码如下:  

    ```
    var mongoose = require('mongoose'),
        Schema = mongoose.Schema;

    /**
     * Article Schema
     */
    var ArticleSchema = new Schema({
        created: {
            type: Date,
            default: Date.now
        },
        title: {
            type: String,
            default: '',
            trim: true,
            required: 'Title cannot be blank'
        },
        content: {
            type: String,
            default: '',
            trim: true
        },
        user: {
            type: Schema.ObjectId,
            ref: 'User'
        }
    });

    mongoose.model('Article', ArticleSchema);
    ```

到这里，整个页面从访问到处理到返回数据和渲染页面的流程就完毕了。
