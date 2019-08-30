---
layout:       post
title:        "Kotlin-Coroutines 与 Rxjava2"
subtitle:     "今天开始 RxCoroutines"
date:         2019-08-23 00:00:00
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


> 此文章不允许转载, 违者必究...

* content
{:toc}

Kotlin-Coroutines 与 Rxjava2
============================

目前 kotlin 语言植根于 **JVM 生态系统**, **Native 生态系统**, **Android&IOS 生态系统** 和 **JS 生态系统**.

`Kotlin-Coroutines` 是拥有 `占用资源相对少` `更高性能`  的并发模型的一项新型JVM多线程编程技术. (就是吃得少干得多又快...)

> 还记得 Linux 环境下, 每个进程一般最多能有 1024 个文件句柄这件小事吧, 其中每个 `Thread ` 实例对应一个文件句柄, 一般新手很快就把它用光.       ￣□￣｜｜

其中 **Android生态系统** 又和 **JVM 生态系统** 有 **交集** 的部分 (JVM 部分).

  **[Kotlin-Coroutines-core/jvm](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/jvm) + [kotlinx-coroutines-android](https://github.com/Kotlin/kotlinx.coroutines/tree/master/ui/kotlinx-coroutines-android)** 就是属于这个 **生态系统的交集**. 中文一般以 **`协程`** 命名之.
  
## 目的

首先 `Kotlin-Coroutines` 与 `Rxjava2` 不是对立的关系, 虽然他们在线程那块功能有重叠.

例如可以做个`Kotlin-Coroutines `版本的 `Scheduler` (更高性能), 大概可以起个 `CoroutinesScheduler` 这样的名称吧!

所以重复三遍
> 用 `Kotlin-Coroutines ` 代替  `Rxjava` 这样的结论实在是有点白.
> <br/>
> 用 `Kotlin-Coroutines ` 代替  `Rxjava` 这样的结论实在是有点白.
> <br/>
> 用 `Kotlin-Coroutines ` 代替  `Rxjava` 这样的结论实在是有点白.
> <br/>
> <br/>
> 特别是像现在网络上那些以知识推广赚钱的那帮人.(各种瞎几把说, 各种瞎几把公众号, 博人眼球人为制造话题制造流量).

下文也会从各方面来阐述 `Kotlin-Coroutines` 与 `Rxjava2` 不是对立的关系, 是可以 `互补`的, 相互促进的.

另外这篇文章主要讲的是

```kotlin

	 Kotlin-Coroutines + Rxjava2
	 
```

的一些实际应用场景.  

它们在JVM 生态中都能适用, 因此 kotlin-backend 和 kotlin-frontend-android 的同学都可以围观.

## 正文

### BackpressureStrategy (背压)

Rxjava 已经发展到 Rxjava2 了, 但 `背压` 的解决方案还是有瑕疵. 直到 `Kotlin-Coroutines` 横空出世.

好吧直接进入主题, 来看看如何使用 `Kotlin-Coroutines` 彻底解决`背压`这个问题.

先举个例子(别问我 Controller 是什么....):

```kotlin
class DemoController() {

    fun test001() {
        // 协程 —— 在主线程的上下文中快速生成元素
    val source = Flowable.fromPublisher<Int>(rxPublisher(capacity = 0) {
        for (x in 1..7) {
            send(x) // 这是一个挂起函数
            println("Sent $x") // 在成功发送元素后打印
        }
    })
    
    // 使用 Rx 让一个处理速度很慢的订阅者在另一个线程订阅
    source
        .observeOn(Schedulers.io(), false, 1) // 指定缓冲区大小为 1 个元素
        .doOnComplete { println("Complete") }
        .subscribe { x ->
            Thread.sleep(5000) // 处理每个元素消耗 5000 毫秒
            println("Processed $x")
        }
    }
}
```

这段代码的输出更好地说明了背压是如何在协程中工作的：

```ruby
13:29:35.354  Sent 1
13:29:40.355  Processed 1
13:29:40.359  Sent 2
13:29:45.363  Processed 2
13:29:45.365  Sent 3
13:29:50.382  Processed 3
13:29:50.387  Sent 4
13:29:55.394  Processed 4
13:29:55.396  Sent 5
13:30:00.397  Processed 5
13:30:00.398  Sent 6
13:30:05.399  Processed 6
13:30:05.403  Sent 7
13:30:10.405  Processed 7
```

这段代码有两个主角 `生产者` 和 `消费者`, 其中 `消费者` 的消费速度 比`生产者`的生产速度慢, 特意用了 `Thread.sleep(5000)` 来模拟这一场景.

 当 `消费者协程` (处于 IO 线程) 消费不过来的时候,   `生产者协程` (处于 Main 线程) 会被自动 suspend (挂起), 也就是 **"暂停"** 生产 (不是 **"停止"** 哦).  当`消费者协程`消化过来后,  `生产者协程` **"恢复"** 生产.
 
是不是比 Rxjava 内置的 `背压` 策略更高效高自然:

- 实现按需生产, 再也不用像传统背压那样,申请块内存作为缓冲区 (像建设个仓库, 来存放产物).

- 不用顾虑缓冲区(仓库)满了, 是要 Drop 掉当前的呢? 还是要取 Lastest 那个呢? ...?

```java

// Rxjava 背压策略一览

package io.reactivex;

/**
 * Represents the options for applying backpressure to a source sequence.
 */
public enum BackpressureStrategy {
    /**
     * OnNext events are written without any buffering or dropping.
     * Downstream has to deal with any overflow.
     * <p>Useful when one applies one of the custom-parameter onBackpressureXXX operators.
     */
    MISSING,
    /**
     * Signals a MissingBackpressureException in case the downstream can't keep up.
     */
    ERROR,
    /**
     * Buffers <em>all</em> onNext values until the downstream consumes it.
     */
    BUFFER,
    /**
     * Drops the most recent onNext value if the downstream can't keep up.
     */
    DROP,
    /**
     * Keeps only the latest onNext value, overwriting any previous value if the
     * downstream can't keep up.
     */
    LATEST
}
```

到目前为止, 应该都想学这个背压相关的"黑魔法"吧!

这个魔法核心部分是这段代码:

```kotlin
val pulisher: Publisher =  rxPublisher {
        for (x in 1..7) {
            send(x) // 这是一个挂起函数
            println("Sent $x") // 在成功发送元素后打印
        }
    }
```

`rxPublisher` 方法产生一个 `Publisher` 实例 (Rxjava 里的一个概念, 这里先简单理解成  `生产者` ),  那么 `rxPublisher`  产生了一个什么样内部结构的 `Publisher` 呢? 看下面代码.

```kotlin
import io.reactivex.FlowableEmitter
import io.reactivex.FlowableOnSubscribe
import io.reactivex.Scheduler
import io.reactivex.disposables.Disposable
import kotlinx.coroutines.CancellationException
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.Job
import kotlinx.coroutines.ObsoleteCoroutinesApi
import kotlinx.coroutines.Runnable
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.cancel
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.channels.ProducerScope
import kotlinx.coroutines.channels.SendChannel
import kotlinx.coroutines.channels.consume
import kotlinx.coroutines.channels.produce
import kotlinx.coroutines.delay
import kotlinx.coroutines.isActive
import kotlinx.coroutines.launch
import kotlinx.coroutines.selects.SelectClause2
import kotlinx.coroutines.sync.Mutex
import org.reactivestreams.Publisher
import org.reactivestreams.Subscription
import java.util.concurrent.TimeUnit
import kotlin.coroutines.CoroutineContext
import kotlin.experimental.ExperimentalTypeInference

@JvmOverloads
@UseExperimental(ExperimentalTypeInference::class, ExperimentalCoroutinesApi::class, ObsoleteCoroutinesApi::class)
fun <E> rxPublisher(
        context: CoroutineContext = kotlin.coroutines.EmptyCoroutineContext,
        capacity: Int = Channel.RENDEZVOUS,
        @BuilderInference block: suspend ProducerScope<E>.() -> Unit
): Publisher<E> =
        Publisher { subscriber ->
            if (subscriber == null) throw NullPointerException("Subscriber cannot be null")
            val rxPublisherCoroutine = Rx2PublisherCoroutine(subscriber, context, capacity)
            subscriber.onSubscribe(rxPublisherCoroutine)
            rxPublisherCoroutine.start(block)
        }

private class Rx2PublisherCoroutine<E>(val subscriber: Subscriber<E>, context: CoroutineContext, val capacity: Int) : Subscription, CoroutineScope by CoroutineScope(context), ProducerScope<E> {

    private val mChannel = Channel<E>(capacity)

    private val mutex = Mutex(locked = true)

    fun start(block: suspend ProducerScope<E>.() -> Unit) {
        launch {
            mChannel.consumeEach { elem ->
                try {
                    if (capacity != Channel.RENDEZVOUS) {
                        mutex.lock()
                    }
                    subscriber.onNext(elem)
                } catch (e: Throwable) {
                    dispose(e)
                    throw e //rethrow
                }
            }
        }

        launch {
            block()
        }
    }

    fun dispose(cause: Throwable?) {
        try {
            cancel(if (cause is CancellationException) cause else null) // CoroutineScope.cancel
            close(cause) // this.close
            if (cause != null && cause !is CancellationException) {
                subscriber.onError(cause)
            } else {
                subscriber.onComplete()
            }
        } finally {
            mutex.unlock()
        }
    }

    override val channel: SendChannel<E>
        get() = mChannel

    @ExperimentalCoroutinesApi
    override val isClosedForSend: Boolean
        get() = channel.isClosedForSend

    @ExperimentalCoroutinesApi
    override val isFull: Boolean
        get() = mutex.isLocked

    override val onSend: SelectClause2<E, SendChannel<E>>
        get() = channel.onSend

    override fun close(cause: Throwable?): Boolean {
        return channel.close()
    }

    @ExperimentalCoroutinesApi
    override fun invokeOnClose(handler: (cause: Throwable?) -> Unit) {
        return channel.invokeOnClose(handler)
    }

    override fun offer(element: E): Boolean {
        if (capacity == Channel.RENDEZVOUS && !mutex.tryLock()) return false
        return channel.offer(element)
    }

    override suspend fun send(element: E) {
        if (capacity == Channel.RENDEZVOUS) {
            mutex.lock()
        }
        channel.send(element)
    }

    override fun cancel() {
        dispose(null)
    }

    override fun request(n: Long) {
        mutex.unlock()
    }
}
```

是不是沉迷于代码的海洋里无法自拔? 我们把上面代码的主要脉络先分离出来,并加上一点注释.

```kotlin
private class Rx2PublisherCoroutine<E>(val subscriber: Subscriber<E>, context: CoroutineContext, val capacity: Int) : Subscription, CoroutineScope by CoroutineScope(context), ProducerScope<E> {

                private val mChannel = Channel<E>(capacity)

                private val mutex = Mutex(locked = true)

                init {
                    // 消费者协程
                    launch {
                        mChannel.consume {
                            for (elem in this) {
                                    subscriber.onNext(elem)
                            }
                        }
                    }

                    // 生产者协程
                    launch {
                        block()
                    }
                }

                override val channel: SendChannel<E> = mChannel
                
                //  用作生产的方法(调用一次生产一个事件), 另外它就是上文中  send( 1 .. 7 ) 的那个 send
                override suspend fun send(element: E) {
                    // 生产前把当前协程 lock 住,  这里的目的是让当前生产的东西被消化后,才能接着生产下一个
                    mutex.lock() 
                    channel.send(element)
                }

                //  前一个 onNext 调用完毕后(前一个事件被消费了), 会调用 request 申请生产下一个事件, 这是 rxjava 方面的知识点.
                override fun request(n: Long) {
                   // 当前生产的东西经过 5000 毫秒的时长被消化掉了 (还记得 Thread.sleep(5000) 吗?)
                   // 接着 unlock 当前协程从而进行下一轮生产
                    mutex.unlock()
                }
            })
        }
```

好吧, 说到这里. 其实这个例子是我简化 `kotlin-coroutine` 一个官方库 [kotlinx-coroutines-rx2](https://github.com/hltj/kotlinx.coroutines-cn/blob/master/reactive/kotlinx-coroutines-rx2) 里一个叫 `rxFlowable` 的 api 做出来的.

顺便奉上 [kotlin-couroutine + rxjava 官方使用指南](https://github.com/Kotlin/kotlinx.coroutines/blob/master/reactive/coroutines-guide-reactive.md#backpressure)

说`Kotlin-Coroutines ` 代替  `Rxjava` 的人应该不知道官方都发布了与 `Rxjava` 相关的 `kotlin-coroutine` 库吧.

#### More

上面的那个例子都是按需生产: 生产一个 --> 消费一个 --> 生产一个 --> 消费一个 

那能不能给 `生产者`一个的缓冲区, 当缓冲区满的时候, 才 *`暂停`* 生产?

可以, 只需:

```kotlin
val source = Flowable.fromPublisher<Int>(rxPublisher(capacity = 1) {
        for (x in 1..7) {
            send(x) // 这是一个挂起函数
            println("Sent $x") // 在成功发送元素后打印
        }
    })
```

这段代码中的 `capacity = 1` 就是给 `生产者` 创建了一个大小为 `1` 的缓冲区.
我们来看看运行效果:

```ruby
13:34:11.207 Sent 1
13:34:11.209 Sent 2
13:34:11.210 Sent 3
13:34:16.211 Processed 1
13:34:16.216 Sent 4
13:34:21.217 Processed 2
13:34:21.224 Sent 5
13:34:26.220 Processed 3
13:34:26.224 Sent 6
13:34:31.226 Processed 4
13:34:31.236 Sent 7
13:34:36.234 Processed 5
13:34:41.240 Processed 6
13:34:46.245 Processed 7
```

总共还是耗时 `35 秒`, 从时间的维度上来看这段代码效率好像没提升呀.

针对这个`消费速度比生产速度慢`的例子来说, 加缓冲区意义不大. 但也有一点性能提升, 那就是 `生产者协程` 不用来回 `suspend(挂起)` `unsuspend(恢复)`那么频繁 (简单来说就是把协程切换的开销降低了).

假如将上面的例子改成 `消费速度比生产速度快`并添加`缓冲区`, 那么`缓冲区`占不满, 也没什么用.

> 上面的说法只针对上面那个例子.
>  对于某些情况, `缓冲区` 还是有用的. 如 [Buffered channels](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/channels.md#buffered-channels)


### CoroutinesScheduler

文章开头说到过,  要做一个 `CoroutinesScheduler`.

做这个之前需要用到 `kotlin-coroutine` 的世界的 `Dispatcher` 和 `CoroutineScope`

`Dispatcher` 有点类似于 java 世界的 `Executor`,  故 `Dispatcher` to `Scheduler` 代码如下:

```kotlin
private fun Job.asDisposable(): Disposable = object : Disposable {
    override fun isDisposed(): Boolean = !isActive
    override fun dispose() = cancel()
}

private class BackedWorkerCoroutineScope(coroutineContext: CoroutineContext) : Scheduler.Worker(),
        CoroutineScope by CoroutineScope(coroutineContext) {

    override fun isDisposed(): Boolean = !isActive
    override fun dispose() = cancel()

    override fun schedule(run: Runnable, delay: Long, unit: TimeUnit): Disposable =
            launch {
                if (delay > 0) {
                    delay(unit.toMillis(delay))
                }
                run.run()
            }.asDisposable()
}

fun CoroutineDispatcher.asScheduler(): Scheduler {
    return DispatcherBackedScheduler(this)
}
```

用法 (用回上面背压的例子):

```kotlin
     // 使用 Rx 让一个处理速度很慢的订阅者在另一个线程订阅
    source
        .observeOn(Dispatchers.IO.asScheduler(), false, 1) // 指定缓冲区大小为 1 个元素
        .doOnComplete { println("Complete") }
        .subscribe { x ->
            Thread.sleep(5000) // 处理每个元素消耗 5000 毫秒
            println("Processed $x")
        }
    }
```

核心代码就是 `Dispatchers.IO.asScheduler()`

接着 `CoroutineScope`, [官方文档](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/coroutine-context-and-dispatchers.md#coroutine-scope)已经解释得很清楚了.
大概意思是`协程所需的一切资源由 CoroutineScope 管理, 当 dispose CoroutineScope 的实例时, 相关的协程资源会随之释放`.

  下面是个相关的使用例子:

```kotlin
class DemoController : CoroutineScope by CoroutineScope(Dispatchers.IO) {

    fun dispose() {
        cancel() // Extension on CoroutineScope
    }
    
     fun doSomething() {
        // launch ten coroutines for a demo, each working for a different time
        repeat(10) { i ->
            launch {
                delay((i + 1) * 200L) // variable delay 200ms, 400ms, ... etc
                println("Coroutine $i is done")
            }
        }
    }
 }
 
fun main() = runBlocking<Unit> {
//sampleStart
    val controller = DemoController()
    controller.doSomething() // run test function
    println("Launched coroutines")
    delay(500L) // delay for half a second
    println("Destroying controller!")
    controller.dispose() // cancels all coroutines
    delay(1000) // visually confirm that they don't work
//sampleEnd    
}
```

The output of this example is:

```ruby
Launched coroutines
Coroutine 0 is done
Coroutine 1 is done
Destroying controller!
```

### Kotlin-Coroutines + Rxjava2 遇到 Android

这节主要用 Kotlin-Coroutines + Rxjava2  去实现 Android 上的 MVVM.

- 实现 ViewModel

```kotlin
 class DemoViewModel() : CoroutineScope by MainScope() {
    
   private val RELEASE_SWITCHER: FlowableProcessor<String> = PublishProcessor.create();    

    protected void onCleared() {
    		super.onCleared()
    		RELEASE_SWITCHER.onNext("DemoViewModel#onCleared")
    }
 
    fun doSomething() : Flowable<Int> {
        // 在 IO 协程上生产
        val source = Flowable.fromPublisher<Int>(rxPublisher(Dispatchers.IO, capacity = 0) {
            for (x in 1..7) {
                send(x) // 这是一个挂起函数
                Log.d("demo", "Sent $x") // 在成功发送元素后打印
            }
        })

        // 使用 Rx 让一个处理速度很慢的订阅者在另一个线程订阅
        return source
                .takeUntil(RELEASE_SWITCHER)  // viewmodel 被释放时, 自动把这个 rx 流释放掉
                .observeOn(Dispatchers.Main.asScheduler()) //切换到 UI 协程
    }
}
```

- 实现 Activity

```kotlin
class DemoActivity : RxAppCompatActivity() {

    companion object {
        @JvmStatic
        val TAG = "DemoActivity"
    }
    
    private val demoViewModel by lazy { ViewModelProviders.of(this@DemoActivity).get(DemoViewModel::class.java) }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_demo)
        
       demoViewModel
       .doSomething()
       .compose(bindUntilEvent(ActivityEvent.DESTROY)) // activity onDestory 时自动释放 rx 流,   这是 trello 公司 rxlifecycle3 库的 api
       .subscribe { x ->
                    textView.text = x  // 更新UI
                }

    }
}
```
### Kotlin-Coroutines + Rxjava2 遇到 Server

[ktor](https://ktor.kotlincn.net/) 是 kotlin 的 Web 框架, 它使用了先进的 `Kotlin-Coroutines`. 
类似于 java 的 spring 全家桶. 
两者功能都很全, 第三方库也很多.
个人用了一下还不错, 就是各种 DSL 写法, 感觉在写 html 代码.

至于如何跟 rxjava 关联起来? 后续再补充...
