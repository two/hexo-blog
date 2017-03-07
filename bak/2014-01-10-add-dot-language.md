---
layout: post
title: "Jekyll 添加DOT language支持"
description: ""
category: "tools"
tags: []
---

>*plantuml是基于graphviz来做的，而graphviz支持DOT language,所以可不可以直接用plantuml支持DOT language来进行画图呢？这样的话岂不是更爽了～，折腾吧，骚年！*

DOT language的[介绍](http://www.graphviz.org/doc/info/lang.html)及[文档](http://www.graphviz.org/Documentation/dotguide.pdf)。在plantuml的官网找到了有关plantuml支持DOT的[说明](http://plantuml.sourceforge.net/dot.html)。根据这个说明的意思，开头的`@startuml`和结尾的`@enduml`需要换成`@startdot`和`@enddot`就可以了，然后用`plantuml.jar`生产图片，官网上说只支持png格式，但是我试了一下svg也是支持的,可能是文档没有更新的原因。  

但是jekyll能不能支持DOT呢？我原来想可能要改一下插件，可以在[http://www.plantuml.com/plantuml/form](http://www.plantuml.com/plantuml/form)试了一下DOT是可以生成图片的，也就是说我用的JS插件生成图片是原生就支持的。所以我直接用下面的的写法放到了jekyll里：  
{% codeblock %}
{% raw %}
{% plantuml %}
{% endraw %}
digraph G {
    subgraph cluster0 {
        node [style=filled,color=white];
        style=filled;
        color=lightgrey;
        a0 -> a1 -> a2 -> a3;
        label = "process #1";
    }
    subgraph cluster1 {
        node [style=filled];
        b0 -> b1 -> b2 -> b3;
        label = "process #2";
        color=blue
    }
    start -> a0;
    start -> b0;
    a1 -> b3;
    b2 -> a3;
    a3 -> a0;
    a3 -> end;
    b3 -> end;
    start [shape=Mdiamond];
    end [shape=Msquare];
}
{% raw %}
{% endplantuml %}
{% endraw %}
{% endcodeblock %}
效果如下：
{% plantuml %}
digraph G {
    subgraph cluster0 {
        node [style=filled,color=white];
        style=filled;
        color=lightgrey;
        a0 -> a1 -> a2 -> a3;
        label = "process #1";
    }
    subgraph cluster1 {
        node [style=filled];
        b0 -> b1 -> b2 -> b3;
        label = "process #2";
        color=blue
    }
    start -> a0;
    start -> b0;
    a1 -> b3;
    b2 -> a3;
    a3 -> a0;
    a3 -> end;
    b3 -> end;
    start [shape=Mdiamond];
    end [shape=Msquare];
}
{% endplantuml %}
成功了～
