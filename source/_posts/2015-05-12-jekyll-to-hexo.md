title: "从jekyll到hexo"
date: 2015-05-12 14:20:44
tags:
---

>最近把博客从jekyll迁移到了hexo，告一段落后写个总结，給后来的人一个参考。

##为什么要迁移
关于为什么把博客从 jekyll迁移到hexo，其原因其实跟这两个的优劣没有任何关系。每个人都有自己爱好，只要你用好了，其实本质上没什么差别，而我的原因则是因为最近团队前端人员极度缺人，只好让后台开发的人员也开始写js代码，不过自从写了js代码后我才发现js的天地是多么的广阔。通过写js代码我了解了的js语言的特性，以及前端的各种框架，比如[require.js](http://requirejs.org/), [grunt](http://www.gruntjs.net/), [angular](https://angularjs.org/),[express](http://expressjs.com/zh/)等等，以及[Node.js](https://nodejs.org/)开发平台和包管理工具[npm](https://www.npmjs.com/).最后当然还有本博客平台[hexo](http://hexo.io/zh-cn/)。
说回原因，其实就是因为我最近在学习js，而且对ruby不熟悉，所有我把博客迁移到了js的博客框架hexo。

##迁移过程中有哪些坑
根据hexo官方文档的[迁移说明](http://hexo.io/zh-cn/docs/migration.html)，其实很简单，只不过是把之前的`_posts/*`下的md文件都copy到hexo的`_post`下，然后把`new_post_name`改为`new_post_name: :year-:month-:day-:title.md`。看似很简单，但是实际情况却比这个要复杂些，主要是之前jekyll写博客的时候md文件会有很多内容在hexo平台不被识别，特别是jekyll用过某些插件后，hexo更不认识了。

###问题一: 高亮代码标签不兼容

jekyll中代码的高亮的tag是这样的:
```
{% highlight %}
{% endhighlight %}
```
而hexo中，高亮代码的tag是这样的:
```
{% codeblock %}
{% endcodeblock %}
```
这个问题有两种解决方案，一种是用`sed`命令替换所有的jekyll高亮tag;第二种是自己写一个解析jekyll的tag插件.显然第一种方案更合适一些，所有我替换了所有的标签。  但是如果你想挑战自己，写一个插件也不错，可以copy hexo的codeblock插件代码，直接用到jekyll的tag上。

###问题二: plantuml不支持
为了画流程图我用了开源的流程图解决方案[plantuml](http://www.plantuml.com/)。但是jekyll其实原生也是不支持plantuml的，可以通过插件来解决，关于jekyll如何使用 plantuml的方法我也写过一篇介绍:[jekyll添加plantuml模块](http://oohcode.com/2013/12/30/jekyll-and-plugin-plantuml/).仍在使用jekyll的同学可以参考一下。  
显然hexo原生也是不支持plantuml的，但是我一时也找不到支持plantuml的插件，所以如果博客一定要支持流程图摆在面前的选择只有两条路了: 
1. 先画好流程图，然后博客中引入。
2. 自己写一个支持plantuml的插件。
关于第一种解决方案，首先是麻烦，你需要提前生成各种图片，然后再引入进来，这会打断写博客的流程，同时也让人觉得很low。第二种解决方案，我研究了一下，发现并不难实现。下面就是plantuml插件实现的过程。  

plantuml解析的两种方案:
1. 首先plantuml提供了java包，可以通过命令行把plantuml文件内容生成svg或png等格式的图片。
2. plantuml还提供了一个很好的平台，可以吧plantuml编码后直接作为参数访问,比如:http://plantuml.com:80/plantuml/png/SyfFKj2rKt3CoKnELR1Io4ZDoSa70000
这种方法是实时计算出来后在plantuml平台生成图片后返回过来的。
下面对比这两种方法:

|方法&特点|方法一：本地生成|方法二 ：同步生成|
|--|--|--|
|优点|页面展示速度快|本地不需要保留任何图片|
|缺点|本地需要维护图片&本地需要提前生产图片|页面图片展示比较慢|

再结合我们写博客的目的，是为了快速方面的使用，所以为了不维护一堆图片，考虑图片的名字等细节问题，我们干脆全放到web端，而且自己的服务器不一定比plantuml的快，所以性能的考虑可以忽略。最终我选择了同步生成的方式来解析plantuml标签。

hexo要添加tag的解析插件，可以参考hexo api文档[标签插件](http://hexo.io/zh-cn/api/tag.html)的介绍。其实就是注册插件，然后把tag里的内容当成参数传给tag处理函数，然后返回结果在页面上渲染。我们的这个hexo-tag-plantuml插件主要目的是把tag里的内容转化为plantuml的图片地址。正好plantuml提供了内容转化为图片的js [API](http://www.plantuml.com/codejavascript.html)，我其实就是把这些代码搬过来，然后根据hexo的tag插件写法实现的。具体源代码及用法请参考[hexo-tag-plantuml](https://www.npmjs.com/package/hexo-tag-plantuml)插件。

###其他问题
有问题时再更新……
