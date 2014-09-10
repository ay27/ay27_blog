---
layout: post
title: 'spydroid项目中的两个bug'
date: 2014-08-03 21:30
comments: true
categories: 
---
spydroid项目：<https://code.google.com/p/spydroid-ipcamera/>

<!--more-->

第一个bug出现在rtsp控制流这一块。当客户端发送一个describe请求时，spydroid返回如下信息：

```
     RTSP/1.0 200 OK
     Server: MajorKernelPanic RTSP Server
     Cseq: 3
     Content-Length: 270
     Content-Base: 192.168.1.108:8086/
     Content-Type: application/sdp
     v=0
     o=- 0 0 IN IP4 192.168.1.108
     s=Unnamed
     i=N/A
     c=IN IP4 192.168.1.107
     t=0 0
     a=recvonly
     m=video 5006 RTP/AVP 96
     a=rtpmap:96 H264/90000
     a=fmtp:96 packetization-mode=1;profile-level-id=42000a;sprop-parameter-               sets=Z0IACpZUBQHogA==,aM44gA==;
     a=control:trackID=1
```

注意到m=video 5006 RTP/AVP 96这一句，由于端口5006已经被写死了，无法更改。这一句的作用是要求客户端用5006端口，以RTP/AVP的方式接收数据。当用VLC播放一个spydroid视频时，一般5006端口都是可用的，那么一般是能够正常播放的。然后，如果在同一台PC上播放两个spydroid视频时，问题出现了！！由于它们都是需要使用5006端口，但是这个端口已经被第一次播放的VLC使用了，第二个VLC无法使用，导致第二个播放窗口不能正常播放。

其实把这个端口写死是一种很诡异的逻辑：服务器这边无法知道客户端有哪些端口是没有被占用的，但是却指定了一个端口，要求客户端一定要用这个端口来链接服务器。这种做法逻辑上都是很诡异的。正规的rtsp通信规则中，这个端口应该是写0，就是说客户端随便用什么端口来链接我都可以。那么选择权就交还给了客户端，逻辑上也是非常正常的。

这个bug已经报告给了spydroid的作者FyHertz，估计会在后续版本中修复的。

其实rtsp这一块还有一个地方写得不好，就是程序把session id写死了。不过经过测试，session id好像在这里不起作用。。。所以这个可以改也可以不改吧，但是正规的做法应该是生成一个唯一的id给每一个会话。

第二个bug：当使用spydroid录像的时间较长时，比如说10分钟，就会出现无法退出程序的情况。时间较短时是没有问题的，如果把录制时的分辨率，码率等降低，那么时间会长一点，可能会到15分钟过的样子，但是还是会无法退出。目前我的猜测是：spydroid内存泄漏了！

虽说java虚拟机确实可以回收不可用的内存，降低了内存泄漏的风险，但是有些情况下还是会内存泄漏的。最经典的例子就是，用java写的程序，在长时间运行时很可能出现各种无响应的问题。

目前我尚未找到有效的解决办法，只是简单的杀进程退出而已。

如果确实需要找原因的话，估计应该从把视频数据压入rtp流这一块入手。