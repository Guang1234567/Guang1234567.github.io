---
layout:       post
title:        "Android Cipher SQLite API Based On SqlBrite"
subtitle:     "基于 SqlBrite 添加支持加密的 API"
date:         2017-12-27 00:00:00
author:       "Guang1234567"
header-img:   "img/post-bg-android.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
music-enable: true
categories:
    - Database
    - Sqlite
    - SqlBrite
    - Android
    - Rxjava
tags:
    - Database
    - Sqlite
    - SqlBrite
    - Android
    - Rxjava
---


> 此文章不允许转载, 违者必究...

> 最近  <i class="fa fa-github fa-lg"></i> [square/sqlbrite] 出 `3.x` 版本. 
> 该版本基于 google 官方的 `android.arch.persistence:db:1.0.0` 重写了 API.
> 
> `android.arch.persistence:db:1.0.0` 是一套 `android api for sqlite`,
> 目的是提供一套规范化的接口来隐藏各个厂商的 `Sqlite实现` 的具体细节, 从而降低开发者在持久化层投入的成本.
> 
> 下面我将要基于 `android.arch.persistence:db:1.0.0` 将 [android-database-sqlcipher][] 和 [wcdb][] 这两个拥有`加密` 特性的 `Sqlite实现` 集成到 [square/sqlbrite].

**目录:**

* content
{:toc}


## Github 项目

[Guang1234567/sqlbrite][] <i class="fa fa-hand-o-left fa-lg"></i>

**USAGE**

------

<pre class="line-numbers" data-start="1" data-line="88-97,100"><code class="language-java">

package com.example.sqlbrite.todo.db;

import android.app.Application;
import android.arch.persistence.db.SupportSQLiteOpenHelper;
import android.arch.persistence.db.SupportSQLiteOpenHelper.Configuration;
import android.arch.persistence.db.SupportSQLiteOpenHelper.Factory;
import android.arch.persistence.db.framework.FrameworkSQLiteOpenHelperFactory;
import android.arch.persistence.db.sqlcipher.SqlcipherSQLiteOpenHelperFactory;
import android.arch.persistence.db.wcdb.WcdbSQLiteOpenHelperFactory;

import com.squareup.sqlbrite3.BriteDatabase;
import com.squareup.sqlbrite3.SqlBrite;

import dagger.Module;
import dagger.Provides;
import io.reactivex.schedulers.Schedulers;

import javax.inject.Singleton;

import timber.log.Timber;

@Module
public final class DbModule {
    @Provides
    @Singleton
    SqlBrite provideSqlBrite() {
        return new SqlBrite.Builder()
                .logger(new SqlBrite.Logger() {
                    @Override
                    public void log(String message) {
                        Timber.tag("Database").v(message);
                    }
                })
                .build();
    }

    @Provides
    @Singleton
    BriteDatabase provideDatabase(SqlBrite sqlBrite, Application application) {

        // 1) android native sqlite, no cipher
        /*
        Configuration configuration = Configuration.builder(application)
            .name("todo.db")
            .callback(new DbCallback())
            .build();

        Factory factory = new FrameworkSQLiteOpenHelperFactory();
        SupportSQLiteOpenHelper helper = factory.create(configuration);

        BriteDatabase db = sqlBrite.wrapDatabaseHelper(helper, Schedulers.io());
        db.setLoggingEnabled(true);
        //*/


        // 2) SQLCipher is an SQLite extension that provides 256 bit AES encryption of database files.
        /*
        Configuration configuration_sqlcipher = Configuration.builder(application)
                .name("todo_sqlcipher.db")
                .callback(new DbCallback())
                .build();

        SqlcipherSQLiteOpenHelperFactory factory_sqlcipher = new SqlcipherSQLiteOpenHelperFactory();
        SupportSQLiteOpenHelper helper_sqlcipher = factory_sqlcipher.create(configuration_sqlcipher, "Passsword_1234567");

        BriteDatabase db_sqlcipher = sqlBrite.wrapDatabaseHelper(helper_sqlcipher, Schedulers.io());
        db_sqlcipher.setLoggingEnabled(true);
        //*/


        // 3) wcdb base on SQLCipher, no cipher
        /*
        Configuration configuration_wcdb = Configuration.builder(application)
                .name("todo_wcdb.db")
                .callback(new DbCallback())
                .build();

        WcdbSQLiteOpenHelperFactory factory_wcdb = new WcdbSQLiteOpenHelperFactory();
        SupportSQLiteOpenHelper helper_wcdb = factory_wcdb.create(configuration_wcdb);

        BriteDatabase db_wcdb = sqlBrite.wrapDatabaseHelper(helper_wcdb, Schedulers.io());
        db_wcdb.setLoggingEnabled(true);
        //*/


        // 4) wcdb base on SQLCipher, has cipher
        ///*
        Configuration configuration_wcdb_cipher = Configuration.builder(application)
                .name("todo_wcdb_cipher.db")
                .callback(new DbCallback())
                .build();

        WcdbSQLiteOpenHelperFactory factory_wcdb_cipher = new WcdbSQLiteOpenHelperFactory();
        SupportSQLiteOpenHelper helper_wcdb_cipher = factory_wcdb_cipher.create(configuration_wcdb_cipher, "Passsword_7654321");

        BriteDatabase db_wcdb_cipher = sqlBrite.wrapDatabaseHelper(helper_wcdb_cipher, Schedulers.io());
        db_wcdb_cipher.setLoggingEnabled(true);
        //*/

        return db_wcdb_cipher;
    }
}

</code></pre>
## 如何集成?

参照 `android.arch.persistence:db-framework:1.0.0` 的源码（照葫芦画瓢...）

`android.arch.persistence:db-framework:1.0.0` 是 google 基于 `android.arch.persistence:db:1.0.0` 封装 `package android.database.*` 下面所有类的一个 Android Library. 


## 集成 android-database-sqlcipher

### Step 1

在 Android Studio 新建 android library module, 接着把 `android.arch.persistence:db-framework:1.0.0` clone 一份.

### Step 2

将包名 `android.database.` 替换成 `net.sqlcipher.database.` 

### Step 3

处理不兼容的接口, 怎么处理? 抛个异常...(如下面代码的高亮行)

<pre class="line-numbers" data-start="1" data-line="11"><code class="language-java">

package android.arch.persistence.db.sqlcipher;

public class SqlcipherSQLiteDatabase{
    
    // ...
    
    @Override
    @RequiresApi(api = Build.VERSION_CODES.JELLY_BEAN)
    public Cursor query(final SupportSQLiteQuery supportQuery,
                        CancellationSignal cancellationSignal) {
        throw new UnsupportedOperationException(); // <-- android-database-sqlcipher 没有这个接口...
    }
    
    // ...
}

</code></pre>

<p class="tip-green">
  <i class="fa fa-flag fa-2x pull-left faa-float animated"></i>o(╯□╰)o<br>毕竟是两套接口, 厂商不想兼容你这也是没办法的事情呀!
</p>


## 集成 wcdb

跟 [集成 android-database-sqlcipher](#集成-android-database-sqlcipher) 差不多, 只不过包名要替换成 `com.tencent.wcdb.database.`


[square/sqlbrite]: https://github.com/square/sqlbrite
[android-database-sqlcipher]: https://github.com/sqlcipher/android-database-sqlcipher
[wcdb]: https://github.com/Tencent/wcdb
[Guang1234567/sqlbrite]: https://github.com/Guang1234567/sqlbrite


