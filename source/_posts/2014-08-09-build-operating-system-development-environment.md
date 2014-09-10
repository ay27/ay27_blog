---
layout: post
title: '搭建操作系统开发环境'
date: 2014-08-09 00:07
comments: true
categories: os
---

##为什么有这篇文章
很多程序猿，都有一个梦想，在自己的电脑上跑自己的操作系统，用自己设计的语言编写代码，用自己的编译器编译。
我目前只简单涉猎了一下操作系统，后边的东西迟早会做，时间问题而已。

<!--more-->

前两天，实验室的一哥们问我如何搭建操作系统的开发环境。正好我之前写过一些文档，直接扔给他就跑起来了。
现在我想，肯定还有人想要类似这样的文档。但是我们的目标是开发操作系统，而开发环境只是一些工具，没必要花太多时间在这上面（说实话，一开始时我还真找不到好的教程来教我搭建一个环境，在这上面花了相当多的时间啊）。

##需要的工具
####linux x86 + gcc + nasm + bochs + gdb + make + eclipse(可选) + vim or other editor
系统推荐使用` 32位linux `，不推荐用windows。原因如下：linux的命令行强大方便，环境搭建方便，代码编写简洁；相反，windows上你可能还需要Cygwin来进行编译链接啥的，开发一般是在VS上，配置麻烦。

##多种方式
1. bochs + bochs GUI debugger
2. bochs + gdb + eclipse
3. bochs + bochs debugger

第一种是我推荐的做法。这是效果图：

![无标题1.png](http://user-image.logdown.io/user/8984/blog/8874/post/218131/86Ttec9vTyPlmcYeOhYn_%E6%97%A0%E6%A0%87%E9%A2%981.png)

这种做法的特点是：全程汇编调试。

从booter，到loader，再到内核，全都是汇编级调试，即使你用了C来写。

但是这种方法有个优点：非常方便的查看段信息，内存信息，寄存器信息等等，因为这个GUI debugger就是bochs附带的 :)


第二种方法的特点是：内核的代码可以用eclipse调试。

但是booter，loader只能用gdb调试，到了内核才能用eclipse调试，而且段信息，内存信息，寄存器信息等查看不是很方便。但是很明显的，在内核后边的代码，你将获得一个非常好的调试环境（虽然eclipse有各种不是，但是相对其他方法来说，eclipse还是太方便了点）。

第三种方法的特点是：全程用命令来调试。

这个是最蛋疼的方法了，估计用的人不会太多。注意我们的目标是开发操作系统，而不是在工具上浪费时间。无疑这一种方法是最低效率的故不太推荐。

## 安装过程（基于ubuntu14.04 32位 + bochs2.6.2）
* nasm汇编语言的安装

{% codeblock %}
sudo apt-get install nasm
{% endcodeblock %}

* bochs依赖环境（依赖包可能版本会有所更新，问题不大）

```
sudo apt-get install gcc g++ gdb xorg-dev libgtk2.0-dev build-essential libc6-dev libwxgtk2.8-dev libx11-xcb-dev vgabios bximage
```

* 配置bochs的configure脚本

```
（方法一）
./configure --with-x11 --with-wx --enable-disasm --enable-all-optimizations --enable-readline  --enable-debugger-gui
（方法二）
./configure --enable-gdb-stub --enable-disasm --enable-all-optimizations --enable-readline
（方法三）
./configure --enable-debugger --enable-disasm
` 注意：如果是64位linux，需要加上：--enable-long-phy-address `
```

* 安装bochs

```
make
sudo make install
```

假如make阶段出错，那就需要修改Makefile了：

打开Makefile，在第92行处：

 ![无标题2.png](http://user-image.logdown.io/user/8984/blog/8874/post/218131/tEoCve2cSuWF3gMiBBNM_%E6%97%A0%E6%A0%87%E9%A2%982.png)
 
在最后的位置加上 ` -lpthread ` ，结果如图：

 ![无标题3.png](http://user-image.logdown.io/user/8984/blog/8874/post/218131/k9mxHq3LSUulSMg7Icpw_%E6%97%A0%E6%A0%87%E9%A2%983.png)
 
再次尝试安装。

至此，安装过程结束，下面就是配置使用了。

##方法一的配置
方法一几乎不需要什么配置，就是需要在你的源代码目录写上一个bochsrc文件，然后在这个目录里边命令行启动bochs就可以了。我这里提供一个我自己写的一个bochsrc文件，仅供参考。

```

###############################################################
# Configuration file for Bochs
###############################################################

# how much memory the emulated machine will have
megs: 32

# filename of ROM images
romimage: file=/usr/share/bochs/BIOS-bochs-latest
#vgaromimage: file=/usr/share/vgabios/vgabios.bin

# what disk images will be used
floppya: 1_44=y.img, status=inserted
#floppyb: 1_44=pm.img, status=inserted

# choose the boot disk.
boot: a

# where do we send log messages?
#log: log.txt

# disable the mouse
mouse: enabled=0

# enable key mapping, using US layout as default.
keyboard_mapping: enabled=1, map=/usr/share/bochs/keymaps/x11-pc-us.map

# magic_break是开启写在代码里边的断点，代码里边的断点是：xchg bx, bx
magic_break:enabled=1

# 这里是重点，不加上这点就只能用方法三来调试
display_library: x, options="gui_debug"

```

##方法二的配置
这里的思维是：用gdb远程连接bochs gdb-stub接口，然后用eclipse远程连接到gdb上。注意这里的顺序！

* 在bochsrc文件中加入这一行：

```
# port可以随便写，就是使用了一个端口来进行通信
gdbstub: enabled=1, port=1234, text_base=0, data_base=0, bss_base=0
```

* 在命令行里打开bochs，正常情况应该会显示这样的结果（注意最后一行）：

![无标题4.png](http://user-image.logdown.io/user/8984/blog/8874/post/218131/AjUY2SK8RlqoCbOt5dmU_%E6%97%A0%E6%A0%87%E9%A2%984.png)

* 打开elcipse，新建一个这样的工程：` Makefile Project with existing code `，Toolchain选` none `

![111.png](http://user-image.logdown.io/user/8984/blog/8874/post/218131/Hgs1mZFoTbykmvLpaZuj_111.png)

在工具栏中有这两个东西：

![112.png](http://user-image.logdown.io/user/8984/blog/8874/post/218131/QzhDNgMIR4TFSkbJVFHy_112.png)

左边是debug，右边是run。

现先配置run，再配置debug。

run的配置：注意到类型是` Remote Application `，中间一定要写上` Kernel.bin `文件的完整路径。

![113.png](http://user-image.logdown.io/user/8984/blog/8874/post/218131/GKO0XZRhRF2GQ5NzJE8l_113.png)

debug的配置：

![114.png](http://user-image.logdown.io/user/8984/blog/8874/post/218131/KxOe8ysaRSCpiu58f0Q4_114.png)

一样是` remote application `，写上` kernel.bin `文件的位置，记得选上` disable auto build `！！看到底下的` Using GDB(DSF) ….. `，点开` Select other… `，选上手动启动调试选项：

![115.png](http://user-image.logdown.io/user/8984/blog/8874/post/218131/raRABI1eSOWUVZYfhSZz_115.png)

回到debug的配置界面，打开debugger选项卡，填上调试目的地：

![116.png](http://user-image.logdown.io/user/8984/blog/8874/post/218131/4AyJx3KKRR2CyCwRuwg5_116.png)

在某个内核文件里边打个断点，开始调试：

![117.png](http://user-image.logdown.io/user/8984/blog/8874/post/218131/9dRM6YEXSEKnrdv1OE6y_117.png)


方法二的配置至此结束。

##方法三的配置
这个最简单了，没啥好讲的。

##另外一种
bochs+gdb

这也是一种比较不错的方法，毕竟大名鼎鼎的` gdb `，名声在外，还是有它的优点的。这里简单说一下怎么用gdb调试。

配置方法跟方法二类似，还是需要` gdb-stub `来进行gdb的远程连接。
在方法二的第二步之后，打开gdb：

```
gdb
>target remote localhost:1234
>file KERNEL.BIN
>list file:linenumber
```

gdb的一些小技巧：

```
汇编相关：
>display /I $pc
>si
>ni
>ci

修改寄存器：
>info registers
>set $eax=0

查看内存和修改内存：
>x 0x7c00
>set $0x7c00 value
```

如果嫌默认的AT&T语法的汇编蛋疼，可以在命令行这样干：` sudo echo "set disassembly-flavor intel"> ~/.gdbinit `.

##接下来，就快乐的撸代码吧:-)