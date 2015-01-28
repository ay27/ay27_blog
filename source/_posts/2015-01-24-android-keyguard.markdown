---
layout: post
title: android系统锁屏实现
date: 2015-01-24 13:25:48 +0800
comments: true
categories: 
---

最近打算做一个个性一点的锁屏应用。原以为锁屏的开发应该难度不大，后来发现太天真了。倒腾了好几天，总算是把锁屏做出来了。特写下此文，向各位展示一种比较巧妙的方法。

<!-- more -->

##难点
锁屏，在android系统里边也称为keyguard。一个锁屏应用，在锁屏状态下，为了防止我们的锁屏界面被绕过，应该做到以下几点：

* 屏蔽home、back、menu键
* 屏蔽status bar，或者说屏蔽利用状态栏进行转跳
* 监听系统屏幕的开启和关闭
* 开机自启动

其中，难点是前面两点，后面两点实现方法非常多，在此不赘述。

首先明确一点，如果问题是放在android2.x时代，那么锁屏的话直接声明我们的activity是一个锁屏即可，上面的几点要求就都不是事了。像下面这样的做法：

{% highlight java %}
@Override
public void onAttachedToWindow()
{  //HOMEBUTTON
    if(OnLockMode())
    {   
        this.getWindow().setType(WindowManager.LayoutParams.TYPE_KEYGUARD);
          super.onAttachedToWindow();
    }
    else
    {
        this.getWindow().setType(WindowManager.LayoutParams.TYPE_APPLICATION); 
        super.onAttachedToWindow();
    }
}
{% endhighlight %}

但是，在android4.x之后，TYPE_KEYGUARD被弃用了。也就是说，我们再也不能简单的替换掉系统的锁屏了。所以只能用另外的办法解决。

##屏蔽home、back、menu键
屏蔽back和menu键是相当简单的，以下代码可以非常好的工作在任何android系统下（覆盖activity的方法）：

{% highlight java %}
    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        switch (keyCode) {
            case KeyEvent.KEYCODE_BACK:
            case KeyEvent.KEYCODE_MENU:
                return true;
        }
        return super.onKeyDown(keyCode, event);
    }
{% endhighlight %}

问题是屏蔽home键不好处理。因为home键直接在framework层处理的，application层只能监听，但是这个监听需要用一个BroadcastReceiver来监听：

{% highlight java %}
public class HomeWatcherReceiver extends BroadcastReceiver {
    private static final String LOG_TAG = "HomeReceiver";
    private static final String SYSTEM_DIALOG_REASON_KEY = "reason";
    private static final String SYSTEM_DIALOG_REASON_RECENT_APPS = "recentapps";
    private static final String SYSTEM_DIALOG_REASON_HOME_KEY = "homekey";
    private static final String SYSTEM_DIALOG_REASON_LOCK = "lock";
    private static final String SYSTEM_DIALOG_REASON_ASSIST = "assist";

    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        Log.i(LOG_TAG, "onReceive: action: " + action);
        if (action.equals(Intent.ACTION_CLOSE_SYSTEM_DIALOGS)) {
            // android.intent.action.CLOSE_SYSTEM_DIALOGS
            String reason = intent.getStringExtra(SYSTEM_DIALOG_REASON_KEY);
            Log.i(LOG_TAG, "reason: " + reason);
            if (SYSTEM_DIALOG_REASON_HOME_KEY.equals(reason)) {
                // 短按Home键
                Log.i(LOG_TAG, "homekey");
            } else if (SYSTEM_DIALOG_REASON_RECENT_APPS.equals(reason)) {
                // 长按Home键 或者 activity切换键
                Log.i(LOG_TAG, "long press home key or activity switch");
            } else if (SYSTEM_DIALOG_REASON_LOCK.equals(reason)) {
                // 锁屏
                Log.i(LOG_TAG, "lock");
            } else if (SYSTEM_DIALOG_REASON_ASSIST.equals(reason)) {
                // samsung 长按Home键
                Log.i(LOG_TAG, "assist");
            }

        }
    }
}
{% endhighlight %}


---

明确一点，既然不能屏蔽home键，那么最简单的想法是顺着home键往下做。

那么很容易想到，我们构建一个自己的home桌面，当按下home键时，开启的是我们自己的桌面，而不是系统的桌面。虽然home键的消息还是没能传递过来，但是我们桌面的启动只有两种可能：一种是home键被按下了，一种是在其他应用里退回来的。

那么，想法已经很清晰了。我们设置一个“桌面”，这个桌面就是我们的锁屏界面。

1. 当监听到屏幕关闭或开启的消息时，设置一个flag，然后转跳到这个“桌面”里
2. 在这个桌面启动的时候检测flag，如果发现被设置了，那么说明此时我们的“桌面”是一个“锁屏”
3. 如果flag没有被设置，说明此时我们的“桌面”就是桌面，直接把自己kill掉，然后转跳到真正的系统桌面

实验证明，这种方法是可以屏蔽掉home键的，前提是要把系统默认桌面设置成我们自己的“桌面”。。。

可是，在有“应用抽屉（二级桌面）”的系统里边，当我们当前处于应用抽屉里边的时候，home键就真的是失效了。传统的在应用抽屉里按home键应该是会退到主桌面，但是使用了我们的锁屏后，在应用抽屉里是无法回退的。

##屏蔽status bar，或者说屏蔽利用状态栏进行转跳
如果在锁屏状态下，我们利用状态栏进行转跳且成功的话，那么我们的锁屏就是个摆设而已。

但是同样的，status bar我们同样没有权限进行操控。如果我们的应用是system app，那一切都好办，因为status bar里边有setEnable()和collapsePanels()两个函数。但是这两个函数没有在sdk里开放，只有查看源码才会发现这两个函数。

如果我们把我们的应用伪装成system app，然后利用反射得到这两个函数，也是可行的。但是伪装的方法只有三种：

1. 放在有系统源码的环境下编译出apk
2. 使用对应系统的签名对我们的应用进行签名
3. 利用root提权

但是，我们仅仅只是想做一个锁屏应用而已。。。。

其实有一个很撇的方法，就是直接监听topActivity。如果当前是在锁屏状态下，监听到topActivity改变了且锁定标志依然生效，那么就说明我们的锁屏将要被绕过。此时我们再直接转跳会自己的锁屏界面即可。

但是能够想象，在利用状态栏进行绕过的时候，会有两次动画：一次是从锁屏转跳到某个app，第二次是从那个app转跳回来。

想想都醉了。。。。

##另谋出路
走投无路之下，逆了好几个锁屏的app，它们的共同特点是：

* 不需要设置桌面
* 能够屏蔽掉home键和status bar
* 利用status bar转跳的时候，是转跳成功了，但是会被锁屏界面遮挡（注意是遮挡）
* 某些app在锁屏的情况下，锁屏背后的界面依然可以响应点击动作
* 都使用了一个这样的权限：`android.permission.SYSTEM_ALERT_WINDOW`

对，就是遮挡，而不是屏蔽。（在这里感谢好基友s117的合作）

它们更像是一个系统的dialog，能够在任何地方显示。

虽然知道`TYPE_KEYGUARD_DIALOG`是一个可以屏蔽掉home键的dialog，但是无法屏蔽转跳。

然后，我们目前的工作变成了寻找一个全局有效的，不会被遮挡的dialog，这个dialog需要有以下特性：

* 一定是全局有效，不能被其他的应用遮挡
* 这个dialog对home键无响应
* 这个dialog不能被取消（普通的dialog点击屏幕空白处是可以取消的）

##廓然开朗
先不讲上面的dialog是什么，我们先看看android系统是怎么描绘一个窗体的。

描绘一个view，我们首先想到的是将这个view布局到某个activity中。但是如果被布局到activity，它就一定是可以被遮挡的。有没有办法绕过activity直接在屏幕上显示呢？

我们看看activity是怎么显示出来的：

<img src="/images/keyguard/1360501355_1966.jpg" alt="">

更具体的可以查看其他资料，这里不赘述。

也就是说，我们利用WindowManager是可以做到，绕过activity直接在屏幕上显示内容。

想想各大安全软件，都喜欢在屏幕上显示一个小泡泡，无时无刻在刷存在感有木有= =

好了，直接给出最后这个“dialog”的实现代码：

{% highlight java %}
public class MySystemDialog extends Activity {   
    @Override   
    public void onCreate(Bundle savedInstanceState) {   
        super.onCreate(savedInstanceState);   
        setContentView(R.layout.main);   
        Button bb=new Button(getApplicationContext());   
        bb.setText("hahaha");
        WindowManager wm=(WindowManager)getApplicationContext().getSystemService("window");   
        WindowManager.LayoutParams wmParams = new WindowManager.LayoutParams();   
        
        wmParams.type = WindowManager.LayoutParams.TYPE_SYSTEM_ERROR;
        wmParams.format = PixelFormat.OPAQUE;
        wmParams.flags=WindowManager.LayoutParams.FLAG_DISMISS_KEYGUARD                 | WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN
                | WindowManager.LayoutParams.FLAG_LAYOUT_INSET_DECOR;;   
        wmParams.width=400;   
        wmParams.height=400;   
        wm.addView(bb, wmParams);
    }   
}
{% endhighlight %}

然后你会发现这个感人肺腑的“hahaha”：

<img src="/images/keyguard/screen.jpg" alt="">

注意哦，这个hahaha是全局的，无论你怎么转跳，它都在那里↖(^ω^)↗

对了，如果要做成锁屏，只需要在上面给出的代码里边修改一下wmParams的一些参数就行了，具体不赘述，自己找sdk源码看一下就知道有什么参数了。只要参数设置好了，连长按关机键弹出的关机窗口都可以遮挡住。。。而背后遮挡的界面的响应也是可以kill掉的。。。

至此，无论是屏蔽home键也好，屏蔽status bar转跳也好，我们其实都不需要做，而是用了一个很巧妙的方法绕过去。

对了，给出一个忠告，记得在开发时，在锁屏界面设定一个关闭按钮，不然很蛋疼。有一次错误的把一个无法解锁的apk发给他人，把手机锁死了，只能扣电池了，幸亏还没做开机锁o(╯□╰)o