---
layout:       post
title:        Android-Trello-Rxlifecycle2
subtitle:     "没写完, 待填坑"
date:         2017-12-06 07:07:07
author:       "Guang1234567"
header-img:   "img/post-bg-android.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
categories: 
    - Android
tags:
    - Android
    - RxAndroid
    - rxlifecycle
    - trello
---

> 此文章不允许转载, 违者必究...

> 目的: trello 团队研发的 [Rxlifecycle2](https://github.com/trello/RxLifecycle) 库(基于[Rxjava2](https://github.com/ReactiveX/RxJava)) 到底用来解决什么问题的? 另外怎么去用它.
>> 简单说一下:
>> 
>> 通常 `Activity#onDestory` 的时候, 要反注册某些 Listener, 释放并停止正在后台运行的 WorkerThread 等资源.
>>
>> 如果你的 Android app 使用 [Rxjava2](https://github.com/ReactiveX/RxJava) 写的, 那么[Rxlifecycle2](https://github.com/trello/RxLifecycle) 库提供了一套 APIs 去帮你简化这些释放资源的操作和统一释放资源的方式.

**目录:**

* content
{:toc}





