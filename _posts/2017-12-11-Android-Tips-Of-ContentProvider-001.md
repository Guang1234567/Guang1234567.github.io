---
layout:       post
title:        "借助 ContentProvider 简化模块化编程"
subtitle:     "ContentProvider 的小技巧"
date:         2018-01-24 00:00:00
author:       "Guang1234567"
header-img:   "img/home-bg-o.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
music-enable: true
categories:
    - Android
    - SDK
    - ContentProvider
tags:
    - Android
    - SDK
    - ContentProvider
---


> 此文章不允许转载, 违者必究...

> 目的:
> 开发 Android App 一般会依赖别的第三方库(如 okhttp) or 同一公司别的小组提供的 Android 库,
> 这些库在使用之前都要进行初始化和配置相关参数, 我们一般会放在 `Application#onCreate()` 里执行.
> 结果导致 `Application#onCreate()` 一大堆配置的代码, 下面来借助 ContentProvider 来简化这个情况.

**目录:**

* content
{:toc}


## demo 地址
   (https://github.com/Guang1234567/Android-Debug-Database


## 讲解

### demo 结构概述

(https://github.com/Guang1234567/Android-Debug-Database/tree/master/debug-db  是一个 Android library 模块

(https://github.com/Guang1234567/Android-Debug-Database/tree/master/app 是一个 Application 模块, 依赖 `debug-db`

### 新旧两种集成 debug-db 库的方式对比

#### 旧的集成方式

在 `app模块` 下新建个 android.app.Application 的子类, 然后在 `app模块的AndroidManifest.xml`中声明它, 如下代码所示:

- Add this to your `app's build.gradle`

```gradle

dependencies {
    debugCompile project(':debug-db')
}

```


- 在 `DemoApplication#onCreate()` 里初始化 debug-db 模块

```java

import android.app.Application;
import com.amitshekhar.DebugDB;

public class DemoApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        DebugDB.initialize(this); // <----- 初始化 debug-db 模块
    }
}

```


- 在 `app's AndroidManifest.xml` 声明 `DemoApplication`

```xml

<!-- app's AndroidManifest.xml -->

<application
    android:name=".DemoApplication">
</application>

```

#### 新的集成方式

**新的集成方式特点**
- 不需要在 `DemoApplication#onCreate()` 里初始化
- 不需要在 `app's AndroidManifest.xml` 里声明 DemoApplication

**改为**
1. 在 `debug-db 模块` 新建一个名为 `DebugDBInitProvider` 的 ContentProvider 子类
2. 在 `demo's AndroidManifest.xml` 里声明 DebugDBInitProvider

**代码如下:**

- Add this to your `app's build.gradle`

```gradle

dependencies {
    debugCompile project(':debug-db')
}

```


- 在 `debug-db 模块` 新建 `DebugDBInitProvider`

```java

public class DebugDBInitProvider extends ContentProvider {
    @Override
    public boolean onCreate() {
        DebugDB.initialize(getContext());
        return true;
    }
}

```


- 在 `debug-db's AndroidManifest.xml` 声明 `DebugDBInitProvider`

```xml

<!-- debug-db's AndroidManifest.xml -->

<application>
    <provider
        android:authorities="${applicationId}.DebugDBInitProvider"
        android:exported="false"
        android:enabled="true"
        android:name=".DebugDBInitProvider" />
</application>

```

**对比结果**

虽然代码没少写, 但以 [新的集成方式](#新的集成方式) , 初始化代码全部封装在库的内部, `app` 只需:

```gradle

// Add this to your `app's build.gradle`

dependencies {
    debugCompile project(':debug-db')
}

```

初始化细节都不用处理 \(^o^)/

## 仍需解决的问题

### 问题一: 配置所需的参数怎么传递到 `debug-db`?

答案: 借助 android 的 resource overlay, 只需在两个gradle文件里配置参数即可.
但这个只能在编译阶段生效, 在运行时无法使用这种方式传递参数(如需要从远程服务器获取本地数据库密钥, 然后作为库的配置参数传递进去).
具体如下代码

- debug-db's build.gradle

```gradle

android {
    //...
    defaultConfig {
        // 配置默认参数
        resValue("string", "PORT_NUMBER", "8080")
    }
    //...
}

```

- app's build.gradle

```gradle

android {
    buildTypes {
        debug {
            // 覆盖默认值
            resValue("string", "PORT_NUMBER", "7788")
        }
    }
}

```

### 问题二: 存在多个依赖库, 并且这些依赖库有顺序关系?

答案: 无法解决这个问题, ContentProvider 与  ContentProvider 被创建的时机是无序的, 并不会因为在 AndroidManifest.xml 的位置靠向而先创建.

## 代码调用顺序

```
Application->attachBaseContext
        |
        ↓
ContentProvider->onCreate
        |
        ↓
Application->onCreate
        |
        ↓
Activity->onCreate
```

这样我们可以在  ContentProvider#onCreate 里进行`库的初始化`,
在 Application#onCreate 直接放心使用 `已被初始化的库所提供的相关 API`, 这也是 [新的集成方式](#新的集成方式) 需要遵循的一个原则:

> ContentProvider#onCreate 初始化库, Application#onCreate 里直接使用库
