---
layout: post
title: "在OSX10.9上编译openJDK"
date: 2014-10-23 01:04:16 +0800
comments: true
categories: 
---
###起因
最近想着捣腾一下java虚拟机的实现技术，然后就有了这一篇文章。因为其实oracle java jdk并没有完全开源，而整出了open jdk这样的玩意，open jdk与oracle java jdk的区别貌似就只有字体渲染部分是不同的。所以用openjdk来做研究是完全够用的。

<!--more-->

###参照
基本上都是参照了这个网页：

<http://gvsmirnov.ru/blog/tech/2014/02/07/building-openjdk-8-on-osx-maverick.html>

下面将我的经验贴出。

###系统环境
系统：osx 10.9.5

xcode：5.1

openjdk版本：openjdk-8

XQuartz： 2.7.6


###概要
1. 安装一堆东西
2. 获取openjdk源码
3. bash ./configure
4. 打上几个补丁
5. 修改一个系统库的头文件
6. make
7. ok

###安装一堆东西

* 安装XCode
* 安装XQuartz
* 安装homebrew

注意如果安装完XQuartz后发现configure时报错说缺少freetype库，那么还是需要独立安装一个freetype库。可以这样简单的安装：` brew install freetype `

{% highlight bash %}

brew install mercirual
brew install freetype
brew tap homebrew/dupes
brew install apple-gcc42
sudo mkdir /usr/bin/backup
sudo mv /usr/bin/gcc /usr/bin/g++ /usr/bin/backup
sudo ln -s /usr/local/Cellar/apple-gcc42/4.2.1-5666.3/bin/g++-4.2 /usr/bin/g++
sudo ln -s /usr/local/Cellar/apple-gcc42/4.2.1-5666.3/bin/gcc-4.2 gcc

{% endhighlight %}

###获取openjdk源码

{% highlight bash %}

hg clone http://hg.openjdk.java.net/jdk8/jdk8
cd jdk8
bash ./get_source.sh

{% endhighlight %}

###configure

{% highlight bash %}
bash ./configure
{% endhighlight %}

###打几个补丁

{% highlight bash %}
pushd hotspot
curl https://gist.github.com/gvsmirnov/8634644/raw > os_bsd_cpp_defined_fix.patch
hg import os_bsd_cpp_defined_fix.patch
curl https://gist.github.com/gvsmirnov/8664413/raw > saproc_make_fobjc_exceptions_flag_fix.patch
hg import saproc_make_fobjc_exceptions_flag_fix.patch
popd
pushd jdk
curl https://gist.github.com/gvsmirnov/8664662/raw > private_extern_fix.patch
hg import private_extern_fix.patch
curl https://gist.github.com/gvsmirnov/8718663/raw > platform_libraries_fobjc_exceptions_fix.patch
hg import platform_libraries_fobjc_exceptions_fix.patch
popd
{% endhighlight %}

将/System/Library/Frameworks/Foundation.framework/Headers/NSUserNotification.h 的第16行，注释掉` NS_AVAILABLE(10_9, NA) `

###make
{% highlight bash %}
make all
{% endhighlight %}

###提醒一点
openjdk源码里边有一份readme，里边写出了各种configure参数和make参数



