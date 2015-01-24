---
layout: post
title: "YUV420SP图像的旋转"
date: 2014-10-09 11:39:52 +0800
comments: true
categories: 
---

###源起
最近在做android上摄像头实时数据的处理，由于从系统获取的图像格式是YUV420SP，且是横屏模式的图像，我需要对它进行顺时针旋转90度。网上搜索一番，发现网上一广为流传的旋转代码，问题很多。故写下此文。

<!--more-->

###YUV420SP格式
YUV，分为三个分量，“Y”表示明亮度（Luminance或Luma），也就是灰度值；而“U”和“V” 表示的则是色度（Chrominance或Chroma），作用是描述影像色彩及饱和度，用于指定像素的颜色。

<img src="/images/post/yuv420.png" alt="">



###网上广为流传的一个错误范例：

{% highlight java %}

public static void rotateYUV240SP(byte[] src,byte[] des,int width,int height)
 {
    
  int wh = width * height;
  //旋转Y
  int k = 0;
  for(int i=0;i<width;i++) {
   for(int j=0;j<height;j++) 
   {
               des[k] = src[width*j + i];   
         k++;
   }
  }
  
  for(int i=0;i<width;i+=2) {
   for(int j=0;j<height/2;j++) 
   { 
               des[k] = src[wh+ width*j + i]; 
               des[k+1]=src[wh + width*j + i+1];
         k+=2;
   }
  }
 }
 
 {% endhighlight %}
 
 ---
 
###为什么是错误的？
 
 观察第一个循环，其实就是对图像矩阵进行了一次转置。注意，转置与旋转不一样！
 转置后从图像上看，就是产生了一个镜面对称的图。
 
###栗子：
 
 ```
 
 1   2   3  4
 5   6   7  8
 9  10  11 12
 
 转置如下：
 1   5   9
 2   6  10
 3   7  11
 4   8  12
 
 顺时针旋转如下：
 9   5   1
 10  6   2
 11  7   3
 12  8   4
 
 ```
 
 So, 上面的那个栗子已经是错误了。而第二个循环也是同样的错误。
 
 
###提供我写的一个解决方案
  
 
{% highlight java %}

    private static void rotate(int width, int height, byte[] src, byte[] dst) {
        int delta = 0;
        for (int x = 0; x < width; x++) {
            for (int y = height - 1; y >= 0; y--) {
                dst[delta] = src[y*width + x];
                delta++;
            }
        }

        int wh = width * height;
        int ww = width / 2, hh = height / 2;

        for (int i = 0; i < ww; i++) {
            for (int j = 0; j < hh; j++) {
                dst[delta] = src[wh + width * (hh - j - 1) + i];
                dst[delta + 1] = src[wh + width * (hh - j - 1) + i + 1];
                delta += 2;
            }
        }
    }


 {% endhighlight %}
 
###简单解释一下：
 
 1. 第一个循环是旋转Y值，注意到第二层循环是倒序
 2. 第二个循环是旋转UV值，你可以简单画个矩阵，看一下坐标的对应关系即可得出
 3. 注意到这个版本的代码其实效率很低，发现了没？大量的乘法！
 
###给出效率好些的版本：

 
{% highlight java %}


    private static void rotate(int width, int height, byte[] src, byte[] dst) {
        int delta = 0;
        for (int x = 0; x < width; x++) {
            int tmp = (height-1)*width;
            for (int y = height - 1; y >= 0; y--) {
                dst[delta] = src[tmp + x];
                tmp -= width;
                delta++;
            }
        }

        int wh = width * height;
        int ww = width / 2, hh = height / 2;

        for (int i = 0; i < ww; i++) {
            int tmp = width * (hh - 1);
            for (int j = 0; j < hh; j++) {
                dst[delta] = src[wh + tmp + i];
                dst[delta + 1] = src[wh + tmp + i + 1];
                delta += 2;
                tmp -= width;
            }
        }
    }
 

 {% endhighlight %}
 
####当然，效率最好的方法是，不能传送两个byte数组，直接定义成全局的，省去传参的时间和空间。这里仅仅是为了说明一种可行方案！