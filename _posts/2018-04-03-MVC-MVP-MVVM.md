---
layout:       post
title:        "浅谈MVC、MVP、MVVM架构模式的区别和联系"
subtitle:     ""
date:         2018-04-03 00:00:00
author:       "Guang1234567"
header-img:   "img/home-bg-o.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
music-enable: true
categories:
    -
    -
tags:
    -
---


> 转载自 http://www.cnblogs.com/guwei4037/p/5591183.html

> 目的: 简单了解一下.

**目录:**

* content
{:toc}


# [浅谈MVC、MVP、MVVM架构模式的区别和联系](http://www.cnblogs.com/guwei4037/p/5591183.html)

MVC、MVP、MVVM这些模式是为了解决开发过程中的实际问题而提出来的，目前作为主流的几种架构模式而被广泛使用。

## 一、MVC（Model-View-Controller）

MVC是比较直观的架构模式，用户操作->View（负责接收用户的输入操作）->Controller（业务逻辑处理）->Model（数据持久化）->View（将结果反馈给View）。

MVC使用非常广泛，比如JavaEE中的SSH框架（Struts/Spring/Hibernate），Struts（View, STL）-Spring（Controller, Ioc、Spring MVC）-Hibernate（Model, ORM）以及ASP.NET中的ASP.NET MVC框架，xxx.cshtml-xxxcontroller-xxxmodel。（实际上后端开发过程中是v-c-m-c-v，v和m并没有关系，下图仅代表经典的mvc模型）

![]({{ "/img/in-post/2018-04-03-MVC-MVP-MVVM/112316-20160616144231401-1133255130.png" | prepend: site.baseurl }})

## 二、MVP（Model-View-Presenter）

MVP是把MVC中的Controller换成了Presenter（呈现），目的就是为了完全切断View跟Model之间的联系，由Presenter充当桥梁，做到View-Model之间通信的完全隔离。

.NET程序员熟知的ASP.NET webform、winform基于事件驱动的开发技术就是使用的MVP模式。控件组成的页面充当View，实体数据库操作充当Model，而View和Model之间的控件数据绑定操作则属于Presenter。控件事件的处理可以通过自定义的IView接口实现，而View和IView都将对Presenter负责。

![]({{ "/img/in-post/2018-04-03-MVC-MVP-MVVM/112316-20160616144243917-1180649863.png" | prepend: site.baseurl }})

## 三、MVVM（Model-View-ViewModel）

如果说MVP是对MVC的进一步改进，那么MVVM则是思想的完全变革。它是将“数据模型数据双向绑定”的思想作为核心，因此在View和Model之间没有联系，通过ViewModel进行交互，而且Model和ViewModel之间的交互是双向的，因此视图的数据的变化会同时修改数据源，而数据源数据的变化也会立即反应到View上。

这方面典型的应用有.NET的WPF，js框架Knockout、AngularJS等。

![]({{ "/img/in-post/2018-04-03-MVC-MVP-MVVM/112316-20160616144255948-1076854192.png" | prepend: site.baseurl }})

参考资料：

http://www.ruanyifeng.com/blog/2015/02/mvcmvp_mvvm.html





