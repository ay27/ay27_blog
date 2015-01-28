---
layout: post
title: android--使用插件化风格编写一个服务管理器
date: 2015-01-28 13:41:49 +0800
comments: true
categories: 
---

##起因
我们知道，一般的app里，都会有若干个服务，以及一个守护服务。

守护服务监控各服务的启动和关闭，并且在合适的时候重启某些服务。这个守护服务本身是自启动自重启的。

当我们的app服务较多时，我们需要一种比较便捷的方式，去管理这些服务的开启和关闭时机。由于我们在开发的时候，通常难以预料以后会有什么服务添加进来，所以这个守护服务必须写的足够灵活，使得每添加一个服务，我们的项目只需或者不需要做修改。这篇文章就是在这样的背景下，讨论如何设计一个良好的结构。

<!-- more -->

##我们的目标
在windows上，我们为某个软件添加一个插件，很多时候仅仅是把对应的dll放入到某个目录下，软件就会自动识别该插件并应用之。这种做法是面向用户的，用户无需从新编译我们的软件，即可使用上新的功能。

但是在android上是很难做到的（目前我是不知道有这样的做法，如果有，请告诉我），所以我们退而求其次，我们仅需在编码层做到 **即插即用** 就好了。

怎样的即插即用呢？用前面说到的服务举个例子：
>
添加一个新的服务，仅需把新代码放入某个目录，重新编译一遍，新服务就会自动按照我们的设定运行。

##需要解决的问题
* 自动发现新添加的服务
* 用户可控可设置
* service与activity解耦

###1. 自动发现新添加的服务
做法比较简单粗暴，类似windows上插件的做法：扫描一遍某个包目录，将该目录下的所有类提取出来。

网上搜索一下，会发现很多很多类似的代码，这里给出一个我感觉不错的代码：

{%highlight java %}
    public static List<Class> getClassesFromPackage(String packageName) {
        ArrayList<Class> result = new ArrayList<Class>();

        try {
            String path = context.getPackageManager().getApplicationInfo(
                    context.getPackageName(), 0).sourceDir;
            DexFile dexfile = new DexFile(path);
            Enumeration<String> entries = dexfile.entries();
            while (entries.hasMoreElements()) {
                String name = entries.nextElement();
                if (name.startsWith(packageName) && !name.contains("$"))
                    result.add(Class.forName(name));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        return result;
    }
{% endhighlight %}

注意：代码中筛选`name`的时候，我添上了这个这个过滤条件：`!name.contains("$")`，这是为了过滤掉一些注解注入时产生的类，更具体的信息可以直接查看`entries`的内容。

###2. 用户可控可设置及解耦
这里我选择了使用`SharePreference`，理由如下：

1. 便于设计
2. 跨线程
3. 简单快速
4. 全局有效

主要是考虑到，activity与service的通信问题。为了解耦，我并不打算使用Binder来完成。因为传统的方式就是通过绑定到某个服务，获取到该服务的一个实例，然后进行后续的操作。

当然我们的设计中，Binder还是有必要存在的，以便对某个特定的服务进行一些特殊操作。但是服务的控制开关不应该放在activity中。

我的想法是，在UI层，用户操作被记录到SharePreference中，而在Service中开启SharePreference的changedListener，从而做出响应。

这样，activity与service基本上解耦了。activity不需要直接操作service，service只是面向SharePreference做出响应。

<img src="/images/plugin/a.png" alt="">

当然如果activity需要获取到某个service的实例，依然需要通过Binder实现，我们需要保留这样的接口。

目前还有一个问题，就是写入SharePreference时，需要对应的key和value，而且需要把这些key与每个service对应起来。

首先肯定的是，这些key的最终源头应该是service。问题是这些key以什么样的方式提供。

* 直接定义成一个静态域，然后获取到对应的class后，反射得到该值（想想前面说到的插件，为什么不能直接获取）
* 定义一个静态方法，反射调用得到值，和第一个方法基本一样
* 在父类定义一个虚函数，由service实现。获取一个该service的实例，调用获取值

说实话这几个方法都不算太好。第一、第二个方法基本上是一样的，但是反射消耗资源太大了。

第三个方法，获取实例后（相当于new出一个对象了），由于service的特性，会直接触发`onCreate()`方法，所以获取到该值后还需要把它stop掉。消耗也是不小。

具体哪个方法最好，我也不知道= =

但是我选择了第三个方法。因为第一个第二个，当添加一个service时，如果忘了设置这么一个静态域，就不能被添加进去了。但是第三个方法不会存在这样的问题。

##解决方案
（第一次画UML图= =，图片可点击放大查看）

{% fancybox /images/plugin/uml.png %}

如图。简单解释一下各类的功能：

* DaemonService：守护服务，保证Manager的有效
* ServiceManager：通过监听SharePreference的变化，开启和关闭对应的用户服务
* ServiceKeyMap：讲Key和Service对应起来。需要注意的是，`class.newInstance()`是调用该类的默认无参构造方法

当需要添加一个新服务的时候，只需要在`user_service`包内添加一个继承自AbsService的类，然后在activity中将用户的操作映射成对应的SharePreference键值即可。
