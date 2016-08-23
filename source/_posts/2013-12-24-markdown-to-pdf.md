---
layout: post
title: "markdown转pdf"
description: ""
category: "tools"
tags: [markdown]
---

>*现在已经对markdown痴迷了，一直在折腾，想以后把markdown 作为我不管是写博客还是写文档的标准文件格式，因为据说它可以转换成很多其他的格式，适应在不同的设备上使用，这就是一劳永逸啊。可以过程却是很坑爹啊～*

## 工具选择
Google 了很久，终于发现一个强大的工具：[pandoc](http://johnmacfarlane.net/pandoc/)。于是苦逼的旅程开始了～
### linux下安装：

1. pandoc[下载安装](http://johnmacfarlane.net/pandoc/installing.html)页面竟然没有redhat?
2. 查看一下发现有一个[all-platforms](http://johnmacfarlane.net/pandoc/installing.html#all-platforms)
3. 提示需要先安装[Haskell platform](http://www.haskell.org/platform/)
4. 选择linux平台却发现没有redhat,只是看到不起眼的一行写着：*See also: [justhub](http://justhub.org/), for RHEL, CentOS, Scientific Linux, and Fedora support*.
5. 进入下载页面总于看到希望了：
`wget http://sherkin.justhub.org/el6/RPMS/x86_64/justhub-release-2.0-4.0.el6.x86_64.rpm`  
`rpm -ivh justhub-release-2.0-4.0.el6.x86_64.rp`  
`yum install haskell`  
尼玛，这个也太大了，要一个多G,我的`usr`都满了～
6. 再回到pandoc的[下载页面](http://johnmacfarlane.net/pandoc/installing.html)执行：`cabal install --force pandoc pandoc-citeproc`
7. 果然又报错:`pandoc-1.12.2.1 failed during the building phase.`,找了好久，还是[stackoverflow](http://stackoverflow.com/questions/19045380/failed-to-compile-pandoc-with-cabal)靠谱，原来是内存不够用了！果断把其它进程kill了，腾出空间再次运行～
8. 终于看到安装成功的提示了，-\_-#。来，运行一个pandoc命令看看：`pandoc -V`,竟然提示command no found!!!＿|￣|○
9. 找找有没有pandoc吧：`find / -name pandoc`果然找到了，二进制文件在这里：`/root/.cabal/bin/pandoc`。把它们加到PATH里，退出再登录一把。
10. Y(^\_^)Y，终于好了！！！
