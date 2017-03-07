---
layout: post
title: "php内核系列1:PHP程序运行的入口"
description: "PHP程序运行的入口"
category: "PHP"
tags: [php]
---

>最近看了[深入理解PHP内核](http://www.php-internals.com/book/)这本书，并结合源码进行阅读。发现刚开始入门的时候很难，往往看了有点儿不知所云，因为虽然说了运行的周期，过程等，但是始终不明白的是:PHP程度的入口到底在哪里？不知道入口就没有起步，我始终走不下去。直到后来看[Apache模块](http://www.php-internals.com/book/?p=chapt02/02-02-01-apache-php-module)这章内容的时候才恍然大悟。这里就PHP的入口问题进行总结一下。

## PHP入口
首先回顾一下，我们使用PHP的方式有哪些?最常用的无非两种情况：

* 与WEB服务器结合，在浏览器下运行。
* 命令行下直接运行PHP脚本文件。

这两种方式的入口分别是什么呢？其实总结一下，就是下面三种入口。这点还可以从我们的PHP编译后的二进制文件找打足丝马迹，编译完PHP后我们会发现有一下几种二进制文件`php`, `php-cgi`等，没错，这些二进制文件就是PHP程序的入口。
## CLI
我们知道PHP脚本可以直接在命令行下运行，像下面这样:
{% codeblock %}
$php test.php
{% endcodeblock %}
其中`php`是PHP源代码编译后的二进制文件，当命令行执行php脚本时，其实是执行了源码`sapi/cli/php_cli.c`下的`main`函数, 这个函数大概内容如下:
{% codeblock %}
# ifdef PHP_CLI_WIN32_NO_CONSOLE
int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nShowCmd)
# else
int main(int argc, char *argv[])
# endif
{
# ifdef ZTS
    void ***tsrm_ls;
# endif
# ifdef PHP_CLI_WIN32_NO_CONSOLE
    int argc = __argc;
    char **argv = __argv;
# endif

    int c;
    int exit_status = SUCCESS;
    int module_started = 0, sapi_started = 0; 
    char *php_optarg = NULL;
    int php_optind = 1, use_extended_info = 0; 
    char *ini_path_override = NULL;
    char *ini_entries = NULL;
    int ini_entries_len = 0; 
    int ini_ignore = 0; 
    sapi_module_struct *sapi_module = &cli_sapi_module; //sapi结构

    /*   
     * Do not move this initialization. It needs to happen before argv is used
     * in any way.
     */
    argv = save_ps_args(argc, argv);

...
...
//根据参数选择处理程序
    while ((c = php_getopt(argc, argv, OPTIONS, &php_optarg, &php_optind, 0, 2))!=-1) {
        switch (c) {
            case 'c':
                if (ini_path_override) {
                    free(ini_path_override);
                }
                ini_path_override = strdup(php_optarg);
                break;
            case 'n':
                ini_ignore = 1;
                break;
            case 'd': {
                /* define ini entries on command line */
                int len = strlen(php_optarg);
                char *val;
...
...

    /*
     * Do not move this de-initialization. It needs to happen right before
     * exiting.
     */
    cleanup_ps_args(argv);
    exit(exit_status);
}
{% endcodeblock %}
具体这个函数是怎么执行的，我们先不关心，我们只是知道这个就是命令行下PHP代码执行的入口程序。
## Apache Module
Apache Module是按照Apache的编码规范，给Apapche写的module, 这些module在Apache服务器启动时会加载进来，并且加载进来的module可以执行执行的代码。关于这点[Apache模块](http://www.php-internals.com/book/?p=chapt02/02-02-01-apache-php-module)这里讲的更清楚。Apahce启动时其实就加载了mod_php5这个module,然后把请求传递给这个module进行处理，这就是PHP程序的入口。在`sapi`下我们可以发现一共有几个跟apache相关的目录`apache`, `apache2filter`, `apache2handler`, `apache_hooks`。这些都是apache相关的module，都会被apache调用。
## CGI
上面说了apache调用php的方式，但是还有一种常见的方式就是cgi。我们知道cgi是通用网关接口，其实就是一种通信的协议。我们经常用的另外一个服务器nginx并没有专门的php module，那么这种服务器是怎么调用PHP处理程序的呢？就是用的CGI。
我们在使用nginx的会配置cgi，最常见的就是`fastcgi_pass   127.0.0.1:9000;`这个配置。其实nginx收到请求之后就会根据配置把请求都转发到了这个端口上，然后等处理完了返回时再交给nginx处理(epoll方式，参见nginx的网络模型)。其实真正调用php脚本的是cgi。php编译后会有一个`php-cgi`的二进制文件，这个就是php的cgi入口。但是一般情况下我们不是直接使用这个，主要是因为处理的请求很多，一个进程肯定处理不过来，需要对php-cgi进程进行管理财型，所以一般情况下我们使用的是`spawn-cgi`或`php-fpm`等，这些程序最大的作用就是管理cgi进程，关于他们的不同，这里就不说了。

## 总结
开篇讲的就是PHP的入口，知道这个等于是入门了，以后阅读源码就是顺藤摸瓜了。根据PHP源码，我们可以看到，这些入口都放在一个`sapi`目录下，其实PHP的所有代码的执行都是通过SAPI(Server Application Programming Interface指的是PHP具体应用的编程接口)这个抽象层来完成的。
