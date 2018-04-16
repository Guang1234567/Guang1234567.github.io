---
layout:       post
title:        "Android热更新方案Robust"
subtitle:     ""
date:         2018-04-03 00:00:00
author:       "Guang1234567"
header-img:   "img/home-bg-o.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
music-enable: true
categories:
    - Android
    - HotFix
tags:
    - Android
    - HotFix
---


> 转载自 https://tech.meituan.com/android_robust.html

> 目的: 看了那么多热修复的方案, 还是美团的方案稍微靠谱一点...

**目录:**

* content
{:toc}

美团•大众点评是中国最大的O2O交易平台，目前已拥有近6亿用户，合作各类商户达432万，订单峰值突破1150万单。美团App是平台主要的入口之一，O2O交易场景的复杂性决定了App稳定性要达到近乎苛刻的要求。用户到店消费买优惠券时死活下不了单，定外卖一个明显可用的红包怎么点也选不中，上了一个新活动用户一点就Crash……过去发生过的这些画面太美不敢想象。客户端相对Web版最大的短板就是有发版的概念，对线上事故很难有即时生效的解决方式，每次发版都如临深渊如履薄冰，毕竟就算再完善的开发测试流程也无法保证不会将Bug带到线上。  
从去年开始，Android平台出现了一些优秀的热更新方案，主要可以分为两类：一类是基于multidex的热更新框架，包括Nuwa、Tinker等；另一类就是native hook方案，如阿里开源的Andfix和Dexposed。这样客户端也有了实时修复线上问题的可能。但经过调研之后，我们发现上述方案或多或少都有一些问题，基于native hook的方案：需要针对dalvik虚拟机和art虚拟机做适配，需要考虑指令集的兼容问题，需要native代码支持，兼容性上会有一定的影响；基于Multidex的方案，需要反射更改DexElements，改变Dex的加载顺序，这使得patch需要在下次启动时才能生效，实时性就受到了影响，同时这种方案在android N [speed-profile]编译模式下可能会有问题，可以参考[Android N混合编译与对热补丁影响解析](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286341&idx=1&sn=054d595af6e824cbe4edd79427fc2706&scene=0#wechat_redirect)。考虑到美团Android用户机型分布的碎片化，很难有一个方案能覆盖所有机型。  
去年底的Android Dev Summit上，Google高调发布了Android Studio 2.0，其中最重要的新特性Instant Run，实现了对代码修改的实时生效（热插拔）。我们在了解Instant Run原理之后，实现了一个兼容性更强的热更新方案，这就是产品化的hotpatch框架－－Robust。

# 原理

Robust插件对每个产品代码的每个函数都在编译打包阶段自动的插入了一段代码，插入过程对业务开发是完全透明。如State.java的getIndex函数：

```java
public long getIndex() {
        return 100;
    }
```

被处理成如下的实现：

```java
public static ChangeQuickRedirect changeQuickRedirect;
    public long getIndex() {
        if(changeQuickRedirect != null) {
            //PatchProxy中封装了获取当前className和methodName的逻辑，并在其内部最终调用了changeQuickRedirect的对应函数
            if(PatchProxy.isSupport(new Object[0], this, changeQuickRedirect, false)) {
                return ((Long)PatchProxy.accessDispatch(new Object[0], this, changeQuickRedirect, false)).longValue();
            }
        }
        return 100L;
    }
```

可以看到Robust为每个class增加了个类型为ChangeQuickRedirect的静态成员，而在每个方法前都插入了使用changeQuickRedirect相关的逻辑，当 changeQuickRedirect不为null时，可能会执行到accessDispatch从而替换掉之前老的逻辑，达到fix的目的。  
如果需将getIndex函数的返回值改为return 106，那么对应生成的patch，主要包含两个class：PatchesInfoImpl.java和StatePatch.java。  
PatchesInfoImpl.java:

```java
public class PatchesInfoImpl implements PatchesInfo {
    public List<PatchedClassInfo> getPatchedClassesInfo() {
        List<PatchedClassInfo> patchedClassesInfos = new ArrayList<PatchedClassInfo>();
        PatchedClassInfo patchedClass = new PatchedClassInfo("com.meituan.sample.d", StatePatch.class.getCanonicalName());
        patchedClassesInfos.add(patchedClass);
        return patchedClassesInfos;
    }
}
```

StatePatch.java：

```java
public class StatePatch implements ChangeQuickRedirect {
    @Override
    public Object accessDispatch(String methodSignature, Object[] paramArrayOfObject) {
        String[] signature = methodSignature.split(":");
        if (TextUtils.equals(signature[1], "a")) {//long getIndex() -> a
            return 106;
        }
        return null;
    }

    @Override
    public boolean isSupport(String methodSignature, Object[] paramArrayOfObject) {
        String[] signature = methodSignature.split(":");
        if (TextUtils.equals(signature[1], "a")) {//long getIndex() -> a
            return true;
        }
        return false;
    }
}
```

客户端拿到含有PatchesInfoImpl.java和StatePatch.java的patch.dex后，用DexClassLoader加载patch.dex，反射拿到PatchesInfoImpl.java这个class。拿到后，创建这个class的一个对象。然后通过这个对象的getPatchedClassesInfo函数，知道需要patch的class为com.meituan.sample.d（com.meituan.sample.State混淆后的名字），再反射得到当前运行环境中的com.meituan.sample.d class，将其中的changeQuickRedirect字段赋值为用patch.dex中的StatePatch.java这个class new出来的对象。这就是打patch的主要过程。通过原理分析，其实Robust只是在正常的使用DexClassLoader，所以可以说这套框架是没有兼容性问题的。

大体流程如下：  
![](https://tech.meituan.com/img/android_robust/patching.png)

# 插件的问题

OK，到这里Robust原理就介绍完了。很简单是不是？而且sample这个例子中也验证成功了。难道一切这么顺利？其实现实并不是这样，我们将这套实现用到美团的主App时，问题出现了：

```java
Conversion to Dalvik format failed:Unable to execute dex: method ID not in [0, 0xffff]: 65536
```

居然不能打出包来了！从原理上分析，除了引入的patch过程aar外，我们这套实现是不会增加别的方法的，而且引入的那个aar的方法才100个左右，怎么会造成美团的mainDex超过65536呢？进一步分析，我们一共处理7万多个函数，导致最后方法数总共增加7661个。这是为什么呢？

看下patch前后的dex对比：  
![](https://tech.meituan.com/img/android_robust/compare_patch_dex.png)

针对com.meituan.android.order.adapter.OrderCenterListAdapter.java分析一下，发现进行hotpatch之后增加了如下6个方法：

```java
public boolean isEditMode() {
        return isEditMode;
    }
private int incrementDelCount() {
        return delCount.incrementAndGet();
    }
private boolean isNeedDisplayRemainingTime(OrderData orderData) {
        return null != orderData.remindtime && getRemainingTimeMillis(orderData.remindtime) > 0;
    }
private boolean isNeedDisplayUnclickableButton(OrderData orderData) {
        return null != orderData.remindtime && getRemainingTimeMillis(orderData.remindtime) <= 0;
    }
private boolean isNeedDisplayExpiring(boolean expiring) {
        return expiring && isNeedDisplayExpiring;
    }
private View getViewByTemplate(int template, View convertView, ViewGroup parent) {
        View view = null;
        switch (template) {
            case TEMPLATE_DEFALUT:
            default:
                view = mInflater.inflate(R.layout.order_center_list_item, null);
        }
        return view;
    }
```

但是这些多出来的函数其实就在原来的产品代码中，为什么没有Robust的情况下不见了，而使用了插件后又出现在最终的class中了呢？只有一个可能，就是ProGuard的内联受到了影响。使用了Robust插件后，原来能被ProGuard内联的函数不能被内联了。看了下ProGuard的Optimizer.java的相关片段：

```java
if (methodInliningUnique) {
    // Inline methods that are only invoked once.
    programClassPool.classesAccept(
        new AllMethodVisitor(
        new AllAttributeVisitor(
        new MethodInliner(configuration.microEdition,
                          configuration.allowAccessModification,
                          true,
                          methodInliningUniqueCounter))));
}
if (methodInliningShort) {
    // Inline short methods.
    programClassPool.classesAccept(
        new AllMethodVisitor(
        new AllAttributeVisitor(
        new MethodInliner(configuration.microEdition,
                          configuration.allowAccessModification,
                          false,
                          methodInliningShortCounter))));
}
```

通过注释可以看出，如果只被调用一次或者足够小的函数，都可能被内联。深入分析代码，我们发现确实如此，只被调用了一次的私有函数、只有一行函数体的函数（比如get、set函数等）都极可能内联。前面com.meituan.android.order.adapter.OrderCenterListAdapter.java多出的那6个函数也证明了这一点。知道原因了就能有解决问题的思路。  
其实仔细思考下，那些可能被内联的只有一行函数体的函数，真的有被插件处理的必要吗？别说一行代码的函数出问题的可能性小，就算出问题了也可以通过patch内联它的那个函数来解决问题，或者patch这一行代码调用的那个函数。只调用了一次的函数其实是一样的。所以通过分析，这样的函数其实是可以不被插件处理的。那么有了这个认识，我们对插件做了处理函数的判断，跳过被ProGuard内联可能性比较大的函数。重新在团购试了一次，这次apk顺利的打包出来了。通过对打出来apk中的dex做分析，发现优化后的插件还是影响了内联效果，不过只导致方法数增加了不到1000个，所以算是临时简单的解决了这个问题。

# 影响

原理上，Robust是为每个函数都插入了一段逻辑，为每个class插入了ChangeQuickRedirect的字段，所以最终肯定会增加apk的体积。以美团主App为例，平均一个函数会比原来增加17.47个字节，整个App中我们一共处理了6万多个函数，导致包大小由原来的19.71M增加到了20.73M。有些class没有必要添加ChangeQuickRedirect字段，以后可以通过将这些class过滤掉的方式来做优化。  
Robust在每个方法前都加上了额外的逻辑，那对性能上有什么影响呢？  
![](https://tech.meituan.com/img/android_robust/effectiveness1.png)  
从图中可以看到，对一个只有内存运算的函数，处理前后分别执行10万次的时间增加了128ms。这是在华为4A上的测试结果。  
对启动速度上的影响：  
![](https://tech.meituan.com/img/android_robust/effectiveness2.png)  
在同一个机器上的结果，处理前后的启动时间相差了5ms。

# 补丁的问题

再来看看补丁本身。要制作出补丁，我们可能会面临如下两个问题：

```
1. 如何解决混淆问题？
2. 被补的函数中使用了super相关的调用怎么办？
```

其实混淆的问题比较好处理。先针对混淆前的代码生成patch.class，然后利用生成release包时对应的mapping文件中的class的映射关系，对patch.class做字符串上的处理，让它使用线上运行环境中混淆的class。  
被补的函数中使用了super相关的调用怎么办？比如某个Activity的onCreate方法中需要调用super.onCreate，而现在这个bad.Class的badMethod就是这个Activity的onCreate方法，那么在patched.class的patchedMethod中如何通过这个Activity的对象，调用它父类的onCreate方法呢？通过分析Instant Run对这个问题的处理，发现它是在每个class中都添加了一个代理函数，专门来处理super的问题的。为每个class都增加一个函数无疑会增加总的方法数，这样做肯定会遇到65536这个问题。所以直接使用Instant Run的做法显然是不可取的。  
在Java中super是个关键字，也无法通过别的对象来访问到。看来，想直接在patched.java代码中通过Activity的对象调用到它父类的onCreate方法有点不太可能了。不过通过对class文件做分析，发现普通的函数调用是使用JVM指令集的invokevirtual指令，而super.onCreate的调用使用的是invokesuper指令。那是不是将class文件中这个调用的指令改为invokesuper就好了？看如下的例子：  
产品代码SuperClass.java：

```java
public class SuperClass {
    String uuid;
    public void setUuid(String id) {
        uuid = id;
    }
    public void thisIsSuper() {
        Log.d("SuperClass", "thisIsSuper "+uuid);
    }
}
```

产品代码TestSuperClass.java：

```java
public class TestSuperClass extends SuperClass{
    String subUuid;
    public void setSubUuid(String id) {
        subUuid = id;
    }

    @Override
    public void thisIsSuper() {
        Log.d("TestSuperClass", "thisIsSuper no call");
    }
}
```

TestSuperPatch.java是DexClassLoader将要加载的代码：

```java
public class TestSuperPatch {
    public static void testSuperCall() {
        TestSuperClass testSuperClass = new TestSuperClass();
        String t = UUID.randomUUID().toString();
        Log.d("TestSuperPatch", "UUID " + t);
        testSuperClass.setUuid(t);
        testSuperClass.thisIsSuper();
    }
}
```

对TestSuperPatch.class的testSuperClass.thisIsSuper()调用做invokesuper的替换，并且将invokesuper的调用作用在testSuperClass这个对象上，然后加载运行：

```java
Caused by: java.lang.NoSuchMethodError: No super method thisIsSuper()V in class Lcom/meituan/sample/TestSuperClass; or its super classes (declaration of 'com.meituan.sample.TestSuperClass' appears in /data/app/com.meituan.robust.sample-3/base.apk)
```

报错信息说在TestSuperClass和TestSuperClass的父类中没有找到thisIsSuper()V函数！但是实际上TestSuperClass和父类中是存在thisIsSuper()V函数的，而且通过apk反编译看也确实存在的，那怎么就找不到呢？分析invokesuper指令的实现，发现系统会在执行指令所在的class的父类中去找需要调用的方法，所以要将TestSuperPatch跟TestSuperClass一样作为SuperClass的子类。修改如下：

```java
public class TestSuperPatch extends SuperClass {
    ...
}
```

然后再做一次尝试：

```java
08-11 09:12:03.012 1787-1787/? D/TestSuperPatch: UUID c5216480-5c3a-4990-896d-58c3696170c5
08-11 09:12:03.012 1787-1787/? D/SuperClass: thisIsSuper c5216480-5c3a-4990-896d-58c3696170c5
```

看一下testSuperCall的实现，将UUID.randomUUID().toString()的结果，通过setUuid赋值给了testSuperClass这个对象的父类的uuid字段。从日志可以看出，对testSuperClass.thisIsSuper处理后，确实是调用到了testSuperClass这个对象的super的thisIsSuper函数。OK，super的问题看来解决了，而且这种方式不会增加方法数。

# 上线后的效果

Robust 靠谱吗？  
![](https://tech.meituan.com/img/android_robust/result.png)  
尝试修个线上的问题，我们是在07.14下午17:00多的时候上线的补丁，我们可以看到接下来的几天一直到07.17号将补丁下线，这个线上问题得到了明显的修复，补丁下线后看到07.18号这个问题又明显上升了。直到07.18号下班前又重新上线补丁。

补丁的兼容性和成功率如何？通过以上的理论分析，可以看到这套实现基本没有兼容性问题，实际上线的数据如下：  
![](https://tech.meituan.com/img/android_robust/successRate.png)

先简单解释下这几个指标：  
补丁列表拉取成功率=拉取补丁列表成功的用户/尝试拉取补丁列表的用户  
补丁下载成功率=下载补丁成功的用户/补丁列表拉取成功的用户  
patch应用成功率=patch成功的用户/补丁下载成功的用户

通过这个表能够看出，我们的patch信息拉取的成功最低，平均97%多，这是因为实际的网络原因，而下载成功后的patch成功率是一直在99.8%以上。而且我们做的是无差别下发，服务端没有做任何针对机型版本的过滤，线上的结果再次证明了Robust的高兼容性。

# 总结

目前业界已有的Android App热更新方案，包括Multidesk和native hook两类，都存在一些兼容性问题。为此我们借鉴Instant Run原理，实现了一个兼容性更强的热更新方案－－Robust。Robust除了高兼容性之外，还有实时生效的优势。so和资源的替换目前暂时未做实现，但是从框架上来说未来是完全有能力支持的。当然，这套方案虽然对开发者是透明的，但毕竟在编译阶段有插件侵入了产品代码，对运行效率、方法数、包体积还是产生了一些副作用。这也是我们下一步努力的方向。

# 参考文献

* Instant Run, Android Tools Project Site, http://tools.android.com/tech-docs/instant-run.
* Oracle, The Java Virtual Machine Instruction Set, https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html.
* Oracle, ClassLoader, https://docs.oracle.com/javase/7/docs/api/java/lang/ClassLoader.html).
* ltshddx, https://github.com/ltshddx/jaop).
* w4lle, [Android热补丁之AndFix原理解析](http://w4lle.github.io/2016/03/03/Android%E7%83%AD%E8%A1%A5%E4%B8%81%E4%B9%8BAndFix%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/).
* shwenzhang, [Android N混合编译与对热补丁影响解析](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=2649286341&idx=1&sn=054d595af6e824cbe4edd79427fc2706&scene=0#wechat_redirect).
