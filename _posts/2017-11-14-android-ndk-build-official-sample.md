---
title: 使用 android ndk 去编译官方的sample
layout: post
categories: Android
tags: Android_NDK
---

> 此文章不允许转载, 违者必究...

> 目的: 如何使用 android ndk 去编译官方的 `android_ndk/samples/gles3jni`

* content
{:toc}





## 开发环境
* windows7
* babun  (相当于 cygwin 一键安装, github 上有自己去搜索)

## 例子在哪里?

```
Administrator@Guang1234567-Win7 [01:11:39] [/cygdrive/d/dev_kit/android_ndk/samples/gles3jni]
-> % tree -L 2
.
├── AndroidManifest-11.xml
├── AndroidManifest-18.xml
├── jni
│   ├── Android-11.mk
│   ├── Android-18.mk
│   ├── Application.mk
│   ├── gl3stub.c
│   ├── gl3stub.h
│   ├── gles3jni.cpp
│   ├── gles3jni.h
│   ├── RendererES2.cpp
│   └── RendererES3.cpp
├── libs
├── README
├── res
└── src

```

## 如何编译它?

### Step 1) 打开 cygwin 终端

进入 smaple 的 jni 子目录: /cygdrive/d/dev_kit/android_ndk/samples/gles3jni/jni

```
Administrator@Guang1234567-Win7 [01:06:15] [/]
-> % cd /cygdrive/d/dev_kit/android_ndk/samples/gles3jni/jni
```

### Step 2) 设置 NDK_PROJECT_PATH 环境变量

```
Administrator@Guang1234567-Win7 [01:07:26] [/cygdrive/d/dev_kit/android_ndk/samples/gles3jni/jni]
-> % export NDK_PROJECT_PATH=/cygdrive/d/dev_kit/android_ndk/samples/gles3jni

```

### Step 3) 执行下面的 ndk-build 命令

> 运行在 >= android 11

```
Administrator@Guang1234567-Win7 [01:07:26] [/cygdrive/d/dev_kit/android_ndk/samples/gles3jni/jni]
-> % ndk-build.cmd APP_BUILD_SCRIPT=./Android-11.mk -B
```

> 运行在 >= android 18

```
Administrator@Guang1234567-Win7 [01:07:26] [/cygdrive/d/dev_kit/android_ndk/samples/gles3jni/jni]
-> % ndk-build.cmd APP_BUILD_SCRIPT=./Android-18.mk -B
```