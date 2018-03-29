---
layout:       post
title:        "AOP In Android"
subtitle:     "在 Android 上的面向切面(切片)编程"
date:         2018-02-03 00:00:00
author:       "Guang1234567"
header-img:   "img/home-bg-o.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
music-enable: false
categories:
    - AOP
    - AspectJ
tags:
    - AOP
    - AspectJ
---


> 此文章不允许转载, 违者必究...

> 目的:

**目录:**

* content
{:toc}


## 什么是面向切面编程?

面向切面编程（AOP是Aspect Oriented Program的首字母缩写)...  省略好多字, 请自行google.

另外面向切面编程是一种编程思想, 跟 OOP 是不冲突的, 可以互相补充, 博大家之长.

## 如何实现面向切面编程

**面向切面编程** 的实现方式多种多样

- **代理模式 + 请求链模式** : 代码完全可以自己实现, 不要依赖任何第三方库. 现实案例: 如 okhttp 的 拦截器.
- **直接修改字节码** 来进行代码注入 : 黑科技, 不过目前已经有  AspectJ, JavaAssist 等等工具能简化修改字节码的这个操作, 这些工具的原理一样, 只不过侧重点有所不同而已.

另外, 今天主题是 **如何使用 AspectJ 在 Android 平台上实现一个"打印Log和切换线程的库**

## AspectJ In Android

[代码项目](https://github.com/Guang1234567/Android_Annotation-Triggered-Logger)

项目结构:

- **app** module : 是个demo.
- **annotations** module : 一些自定义的注解放在这里, 用来配合 AspectJ
- **runtime** moudle : 打印Log和切换线程的库, 核心代码...


### 在 Android Studio 里引入 AspectJ

引入 AspectJ 需要做两个事情:

- 在Android Studio 项目上配置 AspectJ 提供的工具.
- 告诉 AspectJ 需要它处理的代码放在哪里.


**根目录下的 build.gradle**

```gradle

buildscript {
  repositories {
    jcenter()
    mavenCentral()
  }
  dependencies {
    classpath 'com.android.tools.build:gradle:2.2.0'
    classpath 'org.aspectj:aspectjtools:1.8.1' // 引入 AspectJ 工具
  }
}

```


**app 's build.gradle**

```gradle

import org.aspectj.bridge.IMessage
import org.aspectj.bridge.MessageHandler
import org.aspectj.tools.ajc.Main

apply plugin: 'com.android.application'

dependencies {
  compile project(':annotations') // 依赖 annotations 模块
  compile 'org.aspectj:aspectjrt:1.8.1' // 依赖 aspectjrt 库
}

android {
  compileSdkVersion 21
  buildToolsVersion '21.1.2'

  defaultConfig {
    applicationId 'com.XXX.YYY.ZZZ
    minSdkVersion 15
    targetSdkVersion 21
  }

  lintOptions {
    abortOnError true
  }
}

final def log = project.logger
final def variants = project.android.applicationVariants

// 告诉 AspectJ 工具, 代码在哪里...
variants.all { variant ->
  if (!variant.buildType.isDebuggable()) {
    log.debug("Skipping non-debuggable build type '${variant.buildType.name}'.")
    return;
  }

  JavaCompile javaCompile = variant.javaCompile
  javaCompile.doLast {
    String[] args = ["-showWeaveInfo",
                     "-1.5",
                     "-inpath", javaCompile.destinationDir.toString(),
                     "-aspectpath", javaCompile.classpath.asPath,
                     "-d", javaCompile.destinationDir.toString(),
                     "-classpath", javaCompile.classpath.asPath,
                     "-bootclasspath", project.android.bootClasspath.join(File.pathSeparator)]
    log.debug "ajc args: " + Arrays.toString(args)

    MessageHandler handler = new MessageHandler(true);
    new Main().run(args, handler);
    for (IMessage message : handler.getMessages(null, true)) {
      switch (message.getKind()) {
        case IMessage.ABORT:
        case IMessage.ERROR:
        case IMessage.FAIL:
          log.error message.message, message.thrown
          break;
        case IMessage.WARNING:
          log.warn message.message, message.thrown
          break;
        case IMessage.INFO:
          log.info message.message, message.thrown
          break;
        case IMessage.DEBUG:
          log.debug message.message, message.thrown
          break;
      }
    }
  }
}

```

原来要写这么多代码呀, 幸运的是github上已经有相关的 gradle 插件帮你干这个事情, 你只需只需只需

```gradel

buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'

        // 两者选一个...
        classpath 'com.hujiang.aspectjx:gradle-android-plugin-aspectjx:1.0.10' // <---
        // or
        // classpath 'com.jakewharton.hugo:hugo-plugin:1.2.1' // <--
    }
}

```
[hugo](https://github.com/JakeWharton/hugo)
[gradle-android-plugin-aspectjx](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx)


    到此, 准备工作已经做完...


### 在模块**annotations**下定义一些注解

这些注解只是起到**打个标记**和**提供参数**的作用而已, 由于需要在运行时提供参数, 故注解是 `@Retention(RUNTIME)` 级别的.

**打印 log 用的**

```java
@Target({TYPE, METHOD, CONSTRUCTOR})
@Retention(RUNTIME)
public @interface Logged {

    int level() default Log.VERBOSE;

    boolean printStack() default false;
}
```

**切换线程用的**

```java

@Documented
@Retention(RUNTIME)
@Target({METHOD,CONSTRUCTOR,TYPE})
public @interface MainThread {
}

@Documented
@Retention(RUNTIME)
@Target({METHOD,CONSTRUCTOR,TYPE})
public @interface UiThread {
}

@Documented
@Retention(RUNTIME)
@Target({METHOD,CONSTRUCTOR,TYPE})
public @interface WorkerThread {
}

```

### 在模块**runtime**下编写 AOP 相关的核心代码

针对`@Logged` 注解的处理逻辑如下, 只展示出核心部分.

另外 `@MainThread @UiThread @WorkerThread` 大同小异, 处理逻辑在这里[ThreadAspectConfig.java](https://github.com/Guang1234567/Android_Annotation-Triggered-Logger/blob/master/runtime/src/main/java/eu/f3rog/apt/runtime/ThreadAspectConfig.java)

```java
@Aspect
public class LoggedAspectConfig {
    @Pointcut("within(@com.XXX.YYY.Logged *)")
    public void withinAnnotatedClass() {
    }

    @Pointcut("execution(!synthetic * *(..)) && withinAnnotatedClass()")
    public void methodInsideAnnotatedType() {
    }

    @Pointcut("execution(!synthetic *.new(..)) && withinAnnotatedClass()")
    public void constructorInsideAnnotatedType() {
    }

    @Pointcut("execution(@com.XXX.YYY.Logged * *(..)) || methodInsideAnnotatedType()")
    public void method() {
    }

    @Pointcut("execution(@com.XXX.YYY.Logged *.new(..)) || constructorInsideAnnotatedType()")
    public void constructor() {
    }

    @Around("method() || constructor()")
    public Object logAndExecute(ProceedingJoinPoint joinPoint) throws Throwable {
        enterMethod(joinPoint);

        long startNanos = System.nanoTime();
        Object result = joinPoint.proceed();
        long stopNanos = System.nanoTime();
        long lengthMillis = TimeUnit.NANOSECONDS.toMillis(stopNanos - startNanos);

        exitMethod(joinPoint, result, lengthMillis);

        return result;
    }

    private static void enterMethod(JoinPoint joinPoint) {
        // 在方法体执行前调用...
        // 省略了一些代码...
    }

    private static void exitMethod(JoinPoint joinPoint, Object result, long lengthMillis) {
        // 在方法体调用后但在返回值前调用...
        // 省略了一些代码...
    }
}
```

重要的是下面这几个注解, 它们的作用直接看注释吧. 另外查阅 AspectJ 的官方文档可更为详细信息.

```java

/*
@Pointcut("within(@com.XXX.YYY.Logged com.abc.*)"),

意思是匹配com.abc包下所有使用@Logged注解的类；

下面@Pointcut("within(@com.XXX.YYY.Logged *)")
指的是匹配所有使用@Logged注解的类.
*/
@Pointcut("within(@com.XXX.YYY.Logged *)")
public void withinAnnotatedClass() {
}

/*
execution(!synthetic * *(..))

其中 synthetic 是 java 的关键字(很冷门吧, 详细了解移步: https://www.cnblogs.com/bethunebtj/p/7761596.html)
这句大概的意思是, 把"由java编译器生成的方法(如存在内外部类时,产生的 package 访问权限的 access 方法)"排除掉

execution(!synthetic * *(..)) && withinAnnotatedClass()  这个组合应该不用说也是知道是用来干什么的了吧!
*/
@Pointcut("execution(!synthetic * *(..)) && withinAnnotatedClass()")
public void methodInsideAnnotatedType() {
}

/*
    execution(!synthetic *.new(..))

    排除"由java编译器生成的构造方法(如默认构造方法)"排除掉

    execution(!synthetic *.new(..)) && withinAnnotatedClass()  这个组合应该不用说也是知道是用来干什么的了吧!
*/
@Pointcut("execution(!synthetic *.new(..)) && withinAnnotatedClass()")
public void constructorInsideAnnotatedType() {
}

/*
    execution(@com.XXX.YYY.Logged * *(..))  || methodInsideAnnotatedType()

    指的是匹配所有使用@Logged注解的方法 or 使用@Logged注解的类所具有的一些 !synthetic 方法
*/
@Pointcut("execution(@com.XXX.YYY.Logged * *(..)) || methodInsideAnnotatedType()")
public void method() {
}

/*
    execution(@com.XXX.YYY.Logged * *(..))  || methodInsideAnnotatedType()

    指的是匹配所有使用@Logged注解的构造方法 or 使用@Logged注解的类所具有的一些 !synthetic 构造方法
*/
@Pointcut("execution(@com.XXX.YYY.Logged *.new(..)) || constructorInsideAnnotatedType()")
public void constructor() {
}

/*
满足上面所有规则的方法, 插入 logAndExecute(...) 到字节码文件相应的位置中去
*/
@Around("method() || constructor()")
public Object logAndExecute(ProceedingJoinPoint joinPoint) throws Throwable {
    // ...
}

```

### 用法

```java
public class XXXYYYActivity extends Activity {
  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    //...
  }

  @Logged
  @WorkerThread
  private void printArgs(String... args) {
    for (String arg : args) {
      Log.i("Args", arg);
    }
  }

  @Logged
  @MainThread
  private int fibonacci(int number) {
    if (number <= 0) {
      throw new IllegalArgumentException("Number must be greater than zero.");
    }
    if (number == 1 || number == 2) {
      return 1;
    }
    // NOTE: Don't ever do this. Use the iterative approach!
    return fibonacci(number - 1) + fibonacci(number - 2);
  }

  private void startSleepyThread() {
    new Thread(new Runnable() {
      private static final long SOME_POINTLESS_AMOUNT_OF_TIME = 50;

      @Override public void run() {
        sleepyMethod(SOME_POINTLESS_AMOUNT_OF_TIME);
      }

      @Logged
      private void sleepyMethod(long milliseconds) {
        SystemClock.sleep(milliseconds);
      }
    }, "I'm a lazy thr.. bah! whatever!").start();
  }

  @Logged
  @MainThread
  static class Greeter {
    private final String name;

    Greeter(String name) {
      this.name = name;
    }

    private String sayHello() {
      return "Hello, " + name;
    }
  }

  @Logged
  @WorkThread
  static class Charmer {
    private final String name;

    private Charmer(String name) {
      this.name = name;
    }

    public String askHowAreYou() {
      return "How are you " + name + "?";
    }
  }
}
```

### @Logged @WorkThread 同时标注同一个方法时的作用顺序.

一般希望打印日志的代码和业务逻辑代码运行在同一个线程, 故 @ThreadAspectConfig 要排在最外层,
而 @Logged 排在最内层.

```
   void foo(输入参数) {

       输入参数 ---> @ThreadAspectConfig --->  @Logged  --->  业务逻辑 ---> @Logged ---> @ThreadAspectConfig --->  

   }

```


解决方法:

```Java

/**
 * 对多个@Aspect的进行排序顺序
 */
@Aspect
@DeclarePrecedence(
        "com.XXX.YYY.ThreadAspectConfig, com.XXX.YYY.LoggedAspectConfig")
class Precedence {

}

```
