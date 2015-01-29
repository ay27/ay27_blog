---
layout: post
title: 一次奇妙的修bug经历
date: 2015-01-29 18:03:44 +0800
comments: true
categories: 
---

今天在做android上配置文件处理。当向SharePreference插入字符串时，只要插入值与已存储的值不同，就会抛出ClassCastException, String can not be cast to Boolean. 纳闷了好些时间，特写下此文以做纪念。

<!-- more -->

##出现问题的代码
{% highlight java %}

public static void write(KeySet key, String value) {
	PreferenceUtils instance = new PreferenceUtils();
	instance.editor.putString(key.name(), value);
	instance.editor.apply();
}

{% endhighlight %}

很标准的写法，但是每次调用这里，都会抛出一个强制转换异常，String不能强制转换成Boolean。问题是，这里跟Boolean有一毛线关系啊= =

通俗的说，就是系统要我提供一个苹果给他，然后我给了一个苹果给他，他告诉我，我想要的是雪梨，你给我个苹果干嘛╭(╯3╰)╮

##经历
一开始一直以为是系统抽风了，肯定是我API没用对吧。但是翻看了以前写的一些代码，几乎是一模一样啊，这怎么可能啊。

没办法，直接调进去看看系统都做了些啥吧。

{% highlight java %}
public void apply() {
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
        public void run() {
            try {
                mcr.writtenToDiskLatch.await();
            } catch (InterruptedException ignored) {
            }
        }
    };

    QueuedWork.add(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
        public void run() {
            awaitCommit.run();
            QueuedWork.remove(awaitCommit);
        }
    };

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    notifyListeners(mcr);
}
{% endhighlight %}

这是apply函数的代码，第二行的`commitToMemory()`就是把更新值写入到内存中；之后等待写入锁；获取到写入锁后调用`enqueueDiskWrite`把更新值写入到存储设备中；最后通知各监听器。

可以看到，我们调用`putString`时所作的更改，会在`commitToMemory()`里写入到内存。

现在可以猜测，跟进去`commitToMemory()`应该就能看到问题所在了吧↖(^ω^)↗

{% highlight java %}
// Returns true if any changes were made
private MemoryCommitResult commitToMemory() {
   ...
    for (Map.Entry<String, Object> e : mModified.entrySet()) {
        String k = e.getKey();
        Object v = e.getValue();
        if (v == this) {  // magic value for a removal mutation
            if (!mMap.containsKey(k)) {
                continue;
            }
            mMap.remove(k);
        } else {
            boolean isSame = false;
            if (mMap.containsKey(k)) {
                Object existingValue = mMap.get(k);
                if (existingValue != null && existingValue.equals(v)) {
                    continue;
                }
            }
            mMap.put(k, v);
        }
        mcr.changesMade = true;
        if (hasListeners) {
            mcr.keysModified.add(k);
        }
    }
    ...
}
{% endhighlight %}

关键代码就这么多，不就是把更新值与原值比对一下，如果相同就不更新，不相同才写入到`mcr`中进行下一步的更新嘛。不可能存在任何类型问题啊。

##关键
多次调试后，发现只有当需要更新值时才会抛出强转错误，但是即使是抛出强转错误，新值也是正常写入到了配置文件中的。

想起韩老师的《老码识途》一书中的一句话：
>
当我们关注案发现场时，元凶很有可能转移场地了。

此时我们应该肯定的是我们的调用方法应该是没有错的。这里有一个关键点：错误只会在需要更新值时才会抛出。

到这里其实已经比较清晰了，更新值会产生两种动作：1. 写入配置文件；2. 通知监听器。

##解决

既然写入文件是系统代理了的，问题基本不可能出现在这里，所以可以明确的是：错误应该是发生在监听器中。

把整个项目的监听器找了个遍，竟然发现了以下这样的二逼代码：

{% highlight java %}
@Override
public void onSharedPreferenceChanged(SharedPreferences sharedPreferences, String _key) {
	KeySet key = KeySet.valueOf(_key);
	boolean enable = PreferenceUtils.read(key, false);
	if (enable) {
		doSomething();
	} else {
		doOtherthing();
	}
}
{% endhighlight %}

发现了没，竟然没有做任何的类型检测就把值转换成了boolean，怎么可能不错= =

##后记
静下来想想，出现这样的情况，主要是两个原因：

* 一开始时想着配置文件只存boolean值，不存其他值。但是在后续开发时忘记了之前的约定，把一个String写入了配置文件
* 由于前面的约定，就在所有用到配置文件取值的地方省去了类型检测
* 调试的时候不够冷静，其实直接把异常catch后查看其StackTrace就很容易找到问题所在，毕竟还是太年轻了

可以想象，这么一个个人项目，都可能发生违反约定的情况，何况多人合作的项目呢。

其实对于第一二点，解决也不难，就是在所有函数的入口都做有效性检测，不能相信任何传入值的正确性。