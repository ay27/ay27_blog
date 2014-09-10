---
layout: post
title: "android软键盘弹出时控件的遮挡和自调整问题"
date: 2014-09-10 23:09:54 +0800
comments: true
categories: 
---

当我们的layout包含有EditText时，EditText获取焦点软键盘弹出时，会对布局产生挤压或遮蔽，这篇文章主要就是解决如何使得我们的控件能够跟随软键盘的弹出或收起而动态调整位置。

<!--more-->


###需要实现的目标界面
注意到底下的两个按钮的位置，是会跟随软键盘的。

<img src="/images/post/simple_problem.png" alt="">

###实现很简单，但是没人能说清楚
就这么一个简单的跟随，我花了半天时间，确实恼火。

一开始是不知道怎么描述，搜索的关键字是什么？软键盘？软键盘跟随？菜单栏浮动？菜单栏跟随软键盘？输入框遮挡？软键盘遮挡？

然后在搜索的过程中总结出实现这个效果的两种方法，发现都有所缺漏，而且都只是说了一半。

现在我把这两种方法贴出：

1. 在整个Activity的layout里边，在最外层加上 ` ScrollView ` ，使得当软键盘弹出时可以把ScrollView往上推，从而显示出被遮挡的控件。实践发现，只有焦点处的控件不会被遮挡，但是底下的两个按钮还是被遮挡了。
2. 单独设置AndroidManifest.xml里的 ` android:windowSoftInputMode ` 参数，或者和上面的方法混合使用。效果依然不理想。

这个帖子对于这两种方法的解释还可以：[【Android】防止UI界面被输入法遮挡(画面随输入法自适应)](http://blog.csdn.net/feng88724/article/details/6186037)，但是我依照他说的方法修改参数，发现问题依然存在，可见原作者还是没能真的解决这个问题。

###我的实现步骤
1. 将layout的最外层用 ` RelativeLayout ` ，而不是ScrollView或者LinearLayout
2. 将需要软键盘跟随的控件放置在RelativeLayout的底部
3. 让其他控件的位置根据这个控件来设置相对值
4. android:windowSoftInputMode="adjustResize"

其中第一、二、四步是关键，缺一不可。

第三步是为了防止软键盘弹出时挤压界面，导致控件重叠。

###我的猜测
网上很多人对于这个问题的说法和解决办法，基本都没有提到 ` RelativeLayout ` 这一项的。

我的想法是：RelativeLayout在这里的作用：RelativeLayout是相对布局，控件的位置都是动态可变的，当软键盘弹出，占据了屏幕的一块空间后，RelativeLayout将重新对子控件进行测量，重绘，放置。而其他的布局相对偏静态，子控件的位置相对固定。

###最后，贴出我的layout，仅供参考
{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>

<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
        >

    <LinearLayout
            android:orientation="horizontal"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_alignParentBottom="true"
            android:id="@+id/bottom_layout"
            >
        <Button android:layout_width="fill_parent" android:layout_height="wrap_content"
                android:text="链接"
                android:id="@+id/button_link"
                android:layout_weight="1"
                />
        <Button android:layout_width="fill_parent" android:layout_height="wrap_content"
                android:text="图片"
                android:id="@+id/button_pic"
                android:layout_weight="1"/>
    </LinearLayout>

    <EditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:id="@+id/editText2"
            android:layout_below="@+id/editText"
            android:gravity="left|top"
            android:layout_alignParentTop="true"
            android:layout_above="@id/bottom_layout"
            />

</RelativeLayout>

{% endhighlight %}

如果你在开发过程中遇到了问题，随时可以联系我:)
