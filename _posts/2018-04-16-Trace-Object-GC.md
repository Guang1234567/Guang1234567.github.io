---
layout:       post
title:        "用虚引用来跟踪对象被垃圾回收器回收的活动"
subtitle:     ""
date:         2018-04-03 00:00:00
author:       "Guang1234567"
header-img:   "img/home-bg-o.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
music-enable: true
categories:
    - GC
tags:
    - GC
---


> 原创,不得转载

> 目的: 用虚引用来跟踪对象被垃圾回收器回收的活动)

**目录:**

* content
{:toc}


# 为什么需要虚引用?

我们常常需要在某个对象在被GC之前做某些释放资源的操作.
一般我们在子类覆盖 `android.database.sqlite.SQLiteCursor#finalize()` 方法来实现这个目的, 


- 一般情况下,我们应该避免这种做法, 原因: 下面的博客算是写的比较清楚.

https://blog.csdn.net/aitangyong/article/details/39450341

另外, 列举Android 源代码里使用 `android.database.sqlite.SQLiteCursor#finalize()` 的例子

```java

package android.database.sqlite;

/**
 * A Cursor implementation that exposes results from a query on a
 * {@link SQLiteDatabase}.
 *
 * SQLiteCursor is not internally synchronized so code using a SQLiteCursor from multiple
 * threads should perform its own synchronization when using the SQLiteCursor.
 */
public class SQLiteCursor extends AbstractWindowedCursor {

    @Override
    public void close() {
        super.close();
        synchronized (this) {
            mQuery.close();
            mDriver.cursorClosed();
        }
    }

    /**
     * Release the native resources, if they haven't been released yet.
     */
    @Override
    protected void finalize() {
        try {
            // if the cursor hasn't been closed yet, close it first
            if (mWindow != null) {
                if (mStackTrace != null) {
                    String sql = mQuery.getSql();
                    int len = sql.length();
                    StrictMode.onSqliteObjectLeaked(
                        "Finalizing a Cursor that has not been deactivated or closed. " +
                        "database = " + mQuery.getDatabase().getLabel() +
                        ", table = " + mEditTable +
                        ", query = " + sql.substring(0, (len > 1000) ? 1000 : len),
                        mStackTrace);
                }
                close();
            }
        } finally {
            super.finalize();
        }
    }
}

```

- [适合使用finalize的一些场景](https://blog.csdn.net/q397739000/article/details/52489345) , 上面的 `SQLiteCursor`就是其中一种使用场景.


# 如何使用虚引用跟踪对象的回收时机?

- 直接使用 `sun.misc.Cleaner` 来帮助我们跟踪回收时机.

使用实例:

```java

public class LargeMemoryObj
{
    private long mNativeAddress = 0;

    public LargeMemoryObj()
    {
           mNativeAddress =  allowLargeMemoryFromNative
    }

    public static void main(String[] args)
    {
        while (true)
        {
            System.gc();
            LargeMemoryObj heap = new LargeMemoryObj();
            
            
            // 使用 Cleaner 释放资源, 没错就是下面这个语句.
            Cleaner.create(heap, new Runnable() {
                @override
                public void run() {
                    deleteNativeMemory(mNativeAddress);
                }
            
            );
        }
    }
}

```

- `sun.misc.Cleaner` 在 Android 上是没有的! 怎么办? 我参照源码写了一个(原理请google之), 另外建议不要在正式环境用, 因为有些Android手机不起作用,如我的honor8(╬￣皿￣).

```java

import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;
import java.util.Collections;
import java.util.LinkedList;
import java.util.List;
import java.util.Objects;

/**
 * Like {@link java.lang.ref.Cleaner}, but compat for Android.
 *
 * <pre>{@code
 * public class CleaningExample implements AutoCloseable {
 *        // A cleaner, preferably one shared within a library
 *        private static final Cleaner cleaner = <cleaner>;
 *
 *        static class State implements Runnable {
 *
 *            State(...) {
 *                // initialize State needed for cleaning action
 *            }
 *
 *            public void run() {
 *                // cleanup action accessing State, executed at most once
 *            }
 *        }
 *
 *        private final State;
 *        private final Cleaner.Cleanable cleanable
 *
 *        public CleaningExample() {
 *            this.state = new State(...);
 *            this.cleanable = cleaner.register(this, state);
 *        }
 *
 *        public void close() {
 *            cleanable.clean();
 *        }
 *    }
 * }</pre>
 *
 * @author Guang1234567
 * @date 2018/4/16 13:50
 */

public class Cleaner {

    private final ReferenceQueue<Object> mDummyQueue;

    private final List<PhantomCleanable> mPhantomCleanableList;

    private Cleaner() {
        mDummyQueue = new ReferenceQueue();
        mPhantomCleanableList = Collections.synchronizedList(new LinkedList<>());
    }

    public static Cleaner create() {
        Cleaner cleaner = new Cleaner();
        cleaner.start();
        return cleaner;
    }

    private void start() {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent())
            ;
        Thread handler = new ReferenceHandler(tg, "a.b.c.Cleaner for Android");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();
    }

    private class ReferenceHandler extends Thread {


        public ReferenceHandler(ThreadGroup tg, String s) {
            super(tg, s);
        }

        @Override
        public void run() {
            while (true) {
                try {
                    System.out.println("Awaiting for GC");
                    Cleanable cleanable = (Cleanable) mDummyQueue.remove();
                    //Log.e("lhg", String.valueOf("cleaner = " + cleaner));
                    if (cleanable != null) {
                        cleanable.clean();
                    }
                    System.out.println("Referenced GC'd");
                } catch (Throwable ignore) {
                    ignore.printStackTrace();
                }
            }
        }
    }

    public Cleanable register(Object obj, Runnable action) {
        Objects.requireNonNull(obj, "obj");
        Objects.requireNonNull(action, "action");
        PhantomCleanable cleanable = new PhantomCleanable(obj, action);
        return cleanable;
    }

    public interface Cleanable {

        /**
         * Unregisters the cleanable and invokes the cleaning action.
         * The cleanable's cleaning action is invoked at most once
         * regardless of the number of calls to {@code clean}.
         */
        void clean();
    }

    private class PhantomCleanable<T> extends PhantomReference<T>
            implements Cleanable {

        private Runnable mAction;

        private PhantomCleanable(T referent, Runnable action) {
            super(referent, mDummyQueue);
            mAction = action;
            mPhantomCleanableList.add(this);
        }

        @Override
        public void clean() {
            if (mAction != null) {
                mAction.run();
                mAction = null;
            }
            mPhantomCleanableList.remove(this);
        }
    }
}

```

