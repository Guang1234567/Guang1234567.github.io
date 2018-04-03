---
layout:       post
title:        "Gradle 依赖配置 api VS implementation"
subtitle:     ""
date:         2018-04-03 00:00:00
author:       "Guang1234567"
header-img:   "img/home-bg-o.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
music-enable: true
categories:
    - Gradle
    - Android
tags:
    - Gradle
    - Android
---


> 转载自  (于卫国)http://yuweiguocn.github.io/gradle-new-dependency-configurations/

> 目的: 简单了解一下.

**目录:**

* content
{:toc}

## 正文

本文介绍了Gradle 3.4新引入的依赖配置以及 `api` 和 `implementation` 之间的区别。

>《赠汪伦》
>
>
> 李白乘舟将欲行，忽闻岸上踏歌声。
>
> 桃花潭水深千尺，不及汪伦送我情。
>
>
> —唐，李白

Gradle 3.4 引入了新的依赖配置，新增了 `api` 和 `implementation` 来代替 `compile` 依赖配置。其中 `api` 和以前的 compile 依赖配置是一样的。使用 `implementation` 依赖配置，会显著提升构建时间。

接下来，我们举例说明 `api` 和 `implementation` 的区别。

假如我们一个名 MyLibrary 的 module 类库和一个名为 InternalLibrary 的 module 类库。里面的代码类似这样：

```gradle
//internal library module
public class InternalLibrary {
    public static String giveMeAString(){
        return "hello";
    }
}


//my library module
public class MyLibrary {
    public String myString(){
        return InternalLibrary.giveMeAString();
    }
}
```

MyLibrary 中 build.gradle 对 InternalLibrary 的依赖如下：

```gradle
dependencies {
    api project(':InternalLibrary')
}
```

然后在主 module 的 build.gradle 添加对 MyLibrary 的依赖：

```gradle
dependencies {
    api project(':MyLibrary')
}
```

在主 module 中，使用 `api` 依赖配置 MyLibrary 和 InternalLibrary 都可以访问：

```gradle
//so you can access the library (as it should)
MyLibrary myLib = new MyLibrary();
System.out.println(myLib.myString());

//but you can access the internal library too (and you shouldn't)
System.out.println(InternalLibrary.giveMeAString());
```

使用这种方法，会泄露一些不应该被使用的实现。


为了阻止这种情况，Gradle 新增了 `implementation` 配置。如果我们在 MyLibrary 中使用 `implementation` 配置：

```gradle
dependencies {
    implementation project(':InternalLibrary')
}
```

然后在主 module 的 build.gradle 文件中使用 `implementation` 添加对 MyLibrary 的依赖：

```gradle
dependencies {
    implementation project(':MyLibrary')
}
```

使用这个 `implementation` 依赖配置在应用中无法调用 `InternalLibrary.giveMeAString()`。如果 MyLibrary 使用 `api` 依赖 InternalLibrary，无论主 module 使用 `api` 还是 `implementation` 依赖配置，主 module 中都可以访问 `InternalLibrary.giveMeAString()`。

使用这种封箱策略，如果你只修改了 InternalLibrary 中的代码，Gradle 只会重新编译 MyLibrary，它不会触发重新编译整个应用，因为你无法访问 InternalLibrary。当你有大量的嵌套依赖时，这个机制会显著提升构建速度。

其它配置说明如下表所示。

|新配置	| 废弃配置	| 说明
| :-----:   | :-----:   | :---- |
|compileOnly	| provided	| gradle 添加依赖到编译路径，编译时使用。（不会打包到APK）|
|runtimeOnly	| apk	| gradle 添加依赖只打包到 APK，运行时使用。（不会添加到编译路径）|

## 总结

- 当你切换到新的 Android gradle plugin 3.x.x，你应用使用 `implementation` 替换所有的 `compile` 依赖配置。然后尝试编译和测试你的应用。如果没问题那样最好，如果有问题说明你的依赖或使用的代码现在是私有的或不可访问——来自 Android Gradle plugin engineer Jerome Dochez 的建议。

- 如果你是一个lib库的维护者，对于所有需要公开的 API 你应该使用 `api` 依赖配置，测试依赖或不让最终用户使用的依赖使用 `implementation` 依赖配置。


## 参考

* https://stackoverflow.com/a/44419574/7161403





