---
layout: post
title: "jekyll添加plantuml模块"
description: ""
category: "tools"
tags: [jekyll, plantuml]
---

>*讲完了如何正常显示中文后，突然想到使用jekyll + plantuml的组合岂不是更加神奇，于是折腾之~*

首先plantuml的[running](http://plantuml.sourceforge.net/running.html)页面列出了plant所支持的平台及编辑器，可以看到jekyll赫然在列，所以如果按照这个[教程](https://github.com/yjpark/jekyll-plantuml)应该没问题了，可并没有这么简单。
这个教程说的是基于octopress的，虽然octopress也是基于jekyll的，可是还是有差别的，按照教程把plugin进去，在build的时候，发现又错误提示：`./plugins/raw`不存在，这个包确实不存在，加上我队ruby还不熟悉，于是找这个包也找不到，折腾了半天，把行注释掉，又提示了其他的错误，加上jekyll添加plugin我也不是太清楚，所以干脆先放弃，寻找替代方案。

## JS实现plantuml渲染

越是发现列表页还支持js渲染，如果能直接引用js进行渲染也是不错的选择，于是把完整包下载下来，引入default.html页面中，但是第一次并没有成功，是因为rawdeflate.js是异步加载的，必须使用http协议解析才行，而且jquery_plantuml.js的路径必须要改成你的js文件放置的路径，而且默认渲染的是图片，而我想要矢量图，所以把代码`$(this).attr("src", "http://www.plantuml.com/plantuml/img/"+encode64(e.data));`中的img路径改为了svg就可以了。效果如下:  

{% codeblock %}
<img uml="
Bob -> Alice : Love
">
{% endcodeblock %}
<img uml="
Bob -> Alice : Love
">
{% codeblock %}
<img uml="
Bob -> Alice: 访问
Alice --> Bob: 响应
">
{% endcodeblock %}

<img uml="
Bob -> Alice: 访问
Alice --> Bob: 响应
">

发现这里有一个问题：第一个可以正常现实，第二个却不能正常现实，原因是传递`uml`变量的值时，正常的应该是把换行符也传递过去，但是在实际对js传递的参数进行调试的时候换行符不见了。

## Plugin实现plantuml渲染

由于js生产的uml图出现了问题，不能正确解析，而且对泳道图不支持，我只有研究一下真正的plantuml plugin了。
还是接着上面的，git地址下载下来`plantuml.rb`到`_plugins/`目录后，然后在blog里写出下面的代码：
{% codeblock %}
{% raw %}
{% plantuml %}
{% endraw %}
|Web|
start
:action;
|Server|
:running;
:return;
|Web|
end;
{% raw %}
{% endplantuml %}
{% endraw %}
{% endcodeblock %}

发现执行`jekyll build`时会提示不能够加载`./plugins/raw`, 由于对ruby不熟悉，所以找了好久也没找到这个原因，所以最后还得靠自己，我把加载这个包的代码注释了，又运行了一下build,仔细看了一下出错信息，发现里面有一个错误提示Digest这个执行有问题，发现并没有引入这个包，于是在前面加上了`require "digest"`，再次运行发现成功了，也生成了图片地址，但是在`_site`目录里并没有找到这个目录，因为明明在目录里生成了，但是build完后自己又消失了，很奇怪，但是我在网上找了好久也没找到build的内部流程，所以只有另寻出路了，发现`_site`里面除了生成的`blog`目录里都是静态博客外，还有一个`assets`目录，里面正是根目录下的`assets` copy过来的，所以我想到在`assets`目录下建立一个plantuml生成文件的路径，果然再次build的时候可以了，但是有一个不好的点就是每次这个目录下的图片不是重新生成的，而且这个不能放到版本控制里。下面就是我的具体修改方案：
{% codeblock %}
# _config.yml文件添加下面两行
plantuml_jar: ./_lib/plantuml.jar
plantuml_folder: /assets/img/plantuml/
# plantuml.rb 文件添加
require 'digest'

# plantuml.rb 修改生成图片的路径
# folder = "/images/plantuml/"
folder = site.config['plantuml_folder']
# folderpath = site.dest + folder
folderpath = site.dest + "/.." + folder
# filepath = site.dest + folder + filename
filepath = site.dest + "/.." +folder + filename

# plantuml.rb 修改生产svg和解决中文乱码问题
filename = Digest::MD5.hexdigest(code) + ".svg"
cmd = "java -jar " + plantuml_jar + " -tsvg -charset utf-8 -pipe > " + filepath
{% endcodeblock %}

发现在我的电脑上build后能够正常现实，也不能正常现实，只有先把`jekyll serve`重启一下才能成功。但是传到git pages上却不行,我还不太清楚github pages的jekyll运行原理与本地有什么区别，该如何去控制github pages的服务器，所以这个方法还是行不通。

##  JS与Plugin混合渲染plantuml

上面两个方法都有问题，该如何是好呢？于是我又回到了使用js的想法，因为这个渲染后的结果是通过plantuml服务器保存的，通过仔细观察`jquery_plantuml.js`的内容我了解到，其实这个js是实现了一个数据压缩的算法，把传递的代码进行压缩，然后生产一个压缩后的字符串，由于字符串和代码内容是一一对应的，所以plantuml服务器就以此字符串为key对应此代码的图片（分别对应svg和png两种格式）。此前能够通过html标签传递代码，但是换行有问题，另一方面直接写html代码也不美观，可不可以把`planutml.rb`这个plugin改造一下，来传递代码和生成img标签呢？于是再来看plantuml.rb的代码，发现除了中间生产图片的需要用到plantuml.jar，其它的逻辑就是我们刚才想得到的东西，于是把这个插件改造如下：
{% codeblock %}

module Jekyll

  class PlantUMLFile < StaticFile
    def write(dest)
      true
    end
  end

  class PlantUMLBlock < Liquid::Block
    def render(context)
      site = context.registers[:site]
      code = @nodelist.join
      source = "<center>"
      source += "<img alt='this is uml image' uml='"+code+"'>"
      source += "</center>"
      source
    end
  end
end

Liquid::Template.register_tag('plantuml', Jekyll::PlantUMLBlock)
{% endcodeblock %}

plantuml的内容还是第二种方法的内容，效果如下：
可以看到这时候plantuml的展示就正常了，而且github pages同样正常了，这时候再把第二种方法说的添加的无用目录`img/plantuml/`和无用文件`plantuml.jar`都删除就可以了～(￣▽￣)

但是上传后又发现github pages不能正常显示，真是要命啊！我怀疑是不是自己的环境和github pages的环境不一样导致的，于是我又按照github pages 的[官方步骤](https://help.github.com/articles/using-jekyll-with-pages)又来了一次，发现这次在本地运行提示不能找到自己定义的tag，这就对了，线上发邮件说也是这个错误。于是我又找了好久原因，原来是`safe`这个配置导致的，本地默认是false,可以使用第三方插件，如果为`true`就不能使用第三方插件，而github pages为了保证安全，默认的都是`true`，也就是说线上不能使用第三方插件！

### 网站使用静态文件
为了支持这个东西我只好再次想办法了，于是找到了一个[方法](http://blog.nitrous.io/2013/08/30/using-jekyll-plugins-on-github-pages.html)，发现github pages是可以不是用jekyll而使用生成后的静态文件的，步骤如下：

* 新建一个seanerchen.github.io.raw的repo,然后把原来seanerchen.github.io的代码copy过来，放到这个版本控制里管理。
* 删除seanerchen.github.io的所有文件，然后`touch .nojekyll`这个文件。
* copy -rf seanerchen.github.io.raw/_site/* seanerchen.github.io/
* 然后把seanerchen.github.io 下的push上去
* 等待缓冲实效，打开页面。可以了～

效果如下：
{% plantuml %}
|Web|
start
:action;
|Server|
:running;
:return;
|Web|
end;
{% endplantuml %}
