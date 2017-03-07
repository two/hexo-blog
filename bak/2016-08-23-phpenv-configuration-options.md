layout: title
title: phpenv安装自定义配置
date: 2016-08-23 16:45:47
tags: php
---

## 自定义配置
在使用phpenv安装php是，有时候需要对内置扩展进行自定义控制是否开启，比如我要开启`zts`模块, 源码安装我么可以用`./configure --enable-maintainer-zts`来安装，但是phpenv不支持直接这么写，这时候就要phpenv自己的方式来安装了。可以在phpenv安装的路径里找到下面这个文件：`~/.phpenv/plugins/php-build/bin/php-build`, 这个文件就是phpenv install时运行的脚本，可以找到如下内容:
```php
...
CONFIGURE_OPTIONS=$(cat "$PHP_BUILD_ROOT/share/php-build/default_configure_options")
...
if [ -n "$PHP_BUILD_CONFIGURE_OPTS" ]; then
    CONFIGURE_OPTIONS="$CONFIGURE_OPTIONS $PHP_BUILD_CONFIGURE_OPTS"
fi
...
local append_default_libdir='yes'
for option in $CONFIGURE_OPTIONS; do
  case "$option" in
    "--with-libdir"*) append_default_libdir='no' ;;
  esac
done
if [ "$(uname -p)" = "x86_64" ] && [ "${append_default_libdir}" = 'yes' ]; then
    argv="$argv --with-libdir=lib64"
fi
...
./configure $argv > /dev/null
...
```

可见，默认会读取`~/.phpenv/plugins/php-build/share/php-build/default_configure_options`里面的配置加到`./configure`的参数里，当存在变量`$PHP_BUILD_CONFIGURE_OPTS`时，会把这个变量的值也加到`./configure`的参数里。
所以就存在两种方式实现上面的安装方法：
1. `~/.phpenv/plugins/php-build/share/php-build/default_configure_options`文件末尾加上`--enable-maintainer-zts`
2. 运行`PHP_BUILD_CONFIGURE_OPTS=--enable-maintainer-zts phpenv install 5.6.2`
