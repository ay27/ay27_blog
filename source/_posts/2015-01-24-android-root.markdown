---
layout: post
title: 关于android里root权限的使用及提权
date: 2015-01-24 13:26:59 +0800
comments: true
categories: 
---

注意，这里讲述的是如何使用root权限，不是如何获得root权限。也就是说，这里讲的是手机已经root后，如何在我们的app里使用root权限。

<!-- more -->

##在app中获取root权限
在app中申请root权限是相当简单的，在任何一个位置添加这么一条代码：

{%highlight java %}
process = Runtime.getRuntime().exec("su");
{% endhighlight %}

当app运行到此处时，superuser（系统root后的root权限管理器）就会检测到app需要申请权限，就会弹出窗体告知用户并让用户选择。

通过上面的命令，我们获得了一个拥有管理员权限的进程。然后，我们就可以往里边写入需要执行的命令了：

{%highlight java %}
os = new DataOutputStream(process.getOutputStream());
String cmd1="chmod 777 " + pkgCodePath;
os.writeBytes(cmd1 + "\n");
{% endhighlight %}

##把我们的app伪装成系统应用
比如说，我们想在我们的app里边直接修改系统的一些安全设置，比如关闭锁屏，关闭usb等。但是这些都需要`WRITE_SECURE_SETTINGS`这个权限。问题是这个权限是系统应用才能获取的，普通应用是无法获取到的。

但是既然我们有root权限了，我们可以直接利用root权限给我们的app分配一个系统权限就好了。 具体代码如下：

{%highlight java %}
os = new DataOutputStream(process.getOutputStream());
String cmd2 = "pm grant your_app_package_name  the_system_permission_you_want_to_get";
os.writeBytes(cmd2 + "\n");
{% endhighlight %}

关于系统权限和root权限的区别，可以见这篇文章：
<http://blog.csdn.net/superkris/article/details/7709504>
