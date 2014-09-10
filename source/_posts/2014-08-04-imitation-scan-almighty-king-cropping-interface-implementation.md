---
layout: post
title: '模仿“扫描全能王”裁剪界面的一种实现方式'
date: 2014-08-04 21:07
comments: true
categories: 
---
先看扫描全能王的裁剪界面：

<!--more-->

![Screenshot_2014-08-04-21-05-43.png](http://user-image.logdown.io/user/8984/blog/8874/post/216251/qjwsjrC7TXq8WcYeF58V_Screenshot_2014-08-04-21-05-43.png)

当手指触控到一个控制点时，屏幕的左上角或者右上角会显示一个小圈标示当前控制点的放大区。

现在我要模仿这个界面，做一个demo出来。


##先设计大概的界面
我想象的界面是这样的：

![image1.jpg](http://user-image.logdown.io/user/8984/blog/8874/post/216251/VpTzf2sRjSxH2VXKQoQ4_image1.jpg)

为什么把放大区放那么大呢？是觉得扫描全能王那个小圈实在太小，看起来怪吃力的，不方便。

##触控图片区，放大区内的图片跟随移动
我的做法是，在图片区捕捉到触摸动作，把动作的位置(x,y)对应到图片上，然后再根据图片上的相对位置对应到放大区上，使得触摸点中心与放大区中心的图像是对应的。

这里需要解决的问题是：图片区的图片是被压缩的，而放大区的图片又是被放大的。怎么做才能保证放大区的图片跟随触控点移动，使得放大区中心对齐触控点中心。

首先明确一点，放大区的` 放大系数 `默认是` 2 `.

```
设触控点坐标(x, y)，
则对应到图片的坐标是(x/pictureViewWidth * pictureWidth, y/pictureHeight * pictureHeight),
为方便起见，下面的图片坐标用(imageX, imageY)表示
把放大区的图片平移一个向量使得中心对齐(imageX, imageY)，这个向量是这样算的：
(imageX - zoomViewWidth/2, imageY-zoomViewHeight/2)
为什么是/2呢？因为要把放大区中心对齐嘛，放大区中心就是(zoomViewWidth/2, zoomViewHeight/2)


```

这样就能把图片区的触控点位置与放大区的中心点对齐了。

##图片区控制点的绘制即操作
为了做无关区域的半透明化和相关区域的全透明化，我在一个` ImageView `上放一个自定义的` View `用于描绘这几个点和区域的透明化。

明确一点：不能直接简单的把这个自定义的View背景设置成半透明，然后中间区域用代码描绘成全透明。这样做的后果是：代码描绘全透明效果是没有的。

我的做法是：把4个控制点与边界连接起来分成4个区域，把这4个区域分别描绘成半透明即可。

这里是用了` FrameLayout` 把两个View重叠在一起，注意需要用代码来动态的把顶层的View的位置和大小设置成与底部的ImageView是完全对应的，不然在做onTouch方法时会比较麻烦。同时记得把顶层的View的` TouchEvent `投递到底部ImageView中，让ImageView能够根据触控点位置来操纵放大区。

##触控点合理性的检测
1. 变态的角度肯定是没法裁剪的
2. 触控点位置错乱，比如左上角的触控点拉去了右上角，形成不了区域的情况，也是无法裁剪的
3. 暂时就这两种

角度的检测，我用了余弦定理进行计算，如果` abs(cosX)<0.707 `就无法裁剪。为什么是这样呢，画个cos图就知道了。

位置错乱就更好整了，检测一下4个点各自的x,y坐标就行了。

##看一下我做的结果
![Screenshot_2014-08-04-21-40-07.png](http://user-image.logdown.io/user/8984/blog/8874/post/216251/emm8tZdbQcifFzw0Qjtz_Screenshot_2014-08-04-21-40-07.png)

图片是随便找的，不要介意:)


最后附上这个小项目的github链接：<https://github.com/ay27/ImageDragZoomTest>