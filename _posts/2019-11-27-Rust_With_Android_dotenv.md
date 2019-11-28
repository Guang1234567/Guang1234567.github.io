---
layout:       post
title:        "Rust 与 Android 之 .env (dotEnv)"
subtitle:     ""
date:         2019-11-27 00:00:02
author:       "Guang1234567"
header-img:   "img/home-bg-o.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
music-enable: true
categories:
    - dotEnv
    - .env
tags:
    - dotEnv
---


> 此文章不允许转载, 违者必究...

> 目的:

**目录:**

* content
{:toc}


## 简概

一般 App 会多渠道打包, 发布多个在不同配置下的App.
   
---

这些`不同配置`(其实就是一些常量)到底保存在哪里:

- Android 用 `BuildConfig`.
- IOS 用 `plist` or 写个类似于 `ios_build_config.h` 这样的头文件.

---

这些`不同配置`在什么时机生成❓

- 一般通过脚本自动生成
- 在 `编译时(compile-time)` 生成

---

多渠道分为 rust 部分和 android 部分.

假设有 rust 有 `R` 个版本, android有 `A` 个版本,

那么 存在 `R x A` 种组合.

_( ﾟДﾟ)ﾉ  WTF

---

 [可执行的正常运行的例子](https://github.com/Guang1234567/rust_android_common_lib) 在这里.

## 开始打造 rust 与 android 的 .env

### step 1: 简述一下大概的流程

首先会有一个 `.env.android` 的文本文件, 内容类似于如下:

```bash
RUST_BACKTRACE=1

DATABASE_URL=android_test.db

RUST_LOG_TAG=rust_demo_so

# log filter
# `info,database::orm=warn` means turn on global `info` logging and also `warn` for `database::orm`
RUST_LOG_FILTER=debug,greetings::database::orm=info
```   

下面会借助一些工具脚本 or 库, 将上面的 `.env.android` 分别生成 Rust 的 BuildConfig 和 Android 的 BuildConfig.


### step 2:  借助 `dotEnv.gradle` 脚本, 将 `.env.android` 转化成 Android 的 `BuildConfig.java`

**File:		`dotEnv.gradle`**

```gradle
import java.util.regex.Matcher
import java.util.regex.Pattern

def getCurrentFlavor() {
    Gradle gradle = getGradle()

    // match optional modules followed by the task
    // (?:.*:)* is a non-capturing group to skip any :foo:bar: if they exist
    // *[a-z]+([A-Za-z]+) will capture the flavor part of the task name onward (e.g., assembleRelease --> Release)
    def pattern = Pattern.compile("(?:.*:)*[a-z]+([A-Z][A-Za-z]+)")
    def flavor = ""

    gradle.getStartParameter().getTaskNames().any { name ->
        Matcher matcher = pattern.matcher(name)
        if (matcher.find()) {
            flavor = matcher.group(1).toLowerCase()
            return true
        }
    }

    return flavor
}

def loadDotEnv(flavor = getCurrentFlavor()) {
    def envFile = ".env.android"

    if (System.env['ENVFILE']) {
        envFile = System.env['ENVFILE']
    } else if (System.getProperty('ENVFILE')) {
        envFile = System.getProperty('ENVFILE')
    } else if (project.hasProperty("envConfigFiles")) {
        // use startsWith because sometimes the task is "generateDebugSources", so we want to match "debug"
        project.ext.envConfigFiles.any { pair ->
            if (flavor.startsWith(pair.key)) {
                envFile = pair.value
                return true
            }
        }
    } else if (project.hasProperty("defaultEnvFile")) {
        envFile = project.defaultEnvFile
    }

    def env = [:]
    println("Reading env from: $envFile")

    File f = new File("$project.rootDir/../../$envFile");
    if (!f.exists()) {
        f = new File("$envFile");
    }

    if (f.exists()) {
        f.eachLine { line ->
            def matcher = (line =~ /^\s*(?:export\s+|)([\w\d\.\-_]+)\s*=\s*['"]?(.*?)?['"]?\s*$/)
            if (matcher.getCount() == 1 && matcher[0].size() == 3) {
                env.put(matcher[0][1], matcher[0][2].replace('"', '\\"'))
            }
        }
    } else {
        println("**************************")
        println("*** Missing .env file ****")
        println("**************************")
    }

    project.ext.set("env", env)
}

loadDotEnv()

android {
    defaultConfig {
        project.env.each { k, v ->
            def escaped = v.replaceAll("%","\\\\u0025")
            buildConfigField "String", k, "\"$v\""
            resValue "string", k, "\"$escaped\""
        }
    }
}

tasks.whenTaskAdded { task ->
    if (project.hasProperty("envConfigFiles")) {
        project.envConfigFiles.each { envConfigName, envConfigFile ->
            if (task.name.toLowerCase() == "generate"+envConfigName+"buildconfig") {
                task.doFirst() {
                    android.applicationVariants.all { variant ->
                        def variantConfigString = variant.getVariantData().getVariantConfiguration().getFullName()
                        if (envConfigName.contains(variantConfigString.toLowerCase())) {
                            loadDotEnv(envConfigName)
                            project.env.each { k, v ->
                                def escaped = v.replaceAll("%","\\\\u0025")
                                variant.buildConfigField "String", k, "\"$v\""
                                variant.resValue "string", k, "\"$escaped\""
                            }
                        }
                    }
                }
            }
        }
    }
}


```

**File:		`BuildConfig.java`**

```java
/**
 * Automatically generated file. DO NOT MODIFY
 */
package com.rust.example.android;

public final class BuildConfig {
  public static final boolean DEBUG = Boolean.parseBoolean("true");
  public static final String APPLICATION_ID = "com.rust.example.android.debug";
  public static final String BUILD_TYPE = "debug";
  public static final String FLAVOR = "arm8";
  public static final int VERSION_CODE = 1;
  public static final String VERSION_NAME = "1.0";
  // Fields from default config.
  public static final String DATABASE_URL = "android_test.db";
  public static final String RUST_BACKTRACE = "1";
  public static final String RUST_LOG_FILTER = "debug,greetings::database::orm=info";
  public static final String RUST_LOG_TAG = "rust_demo_so";
}
```

### step 3: 借助 `dotenv` 和 `load-dotenv` rust 库, 将 `.env.android` 转化成  Rust 的 `BuildConfig`概的流程

首先在 `lib.rs` 也就是 crate root 进行进行 load dotenv 的操作

**File:		`lib.rs`**

```rust
#[cfg(target_os = "android")]

// ...

use dotenv::dotenv;
use load_dotenv::{load_dotenv, load_dotenv_from_filename};

// load your .env.android file at compile time
load_dotenv_from_filename!(".env.android");

#[no_mangle]
pub unsafe extern fn Java_com_rust_example_android_MainActivity_rustSqlite(
    env: JNIEnv,
    _: JClass,
    database_path: JString) -> jstring {
     
    // ...    

    // 这样子使用!!!
    let database_url: &str = env!("DATABASE_URL","You forgot to export DATABASE_URL path");
    let result = do_some_db_op(format!("{}_{}", input, database_url));

    // ...
}

// ...

```

如果把 `.env.android` 里的 `DATABASE_URL` 注释掉, 那么会直接出现编译错误, 如下:

```ruby
error: You forgot to export DATABASE_URL path
  --> src/lib.rs:89:30
   |
89 |     let database_url: &str = env!("DATABASE_URL","You forgot to export DATABASE_URL path");
   |                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```


## 总结

本文重点描述了如何在 Rust 与 Android 之间共享配置信息.

 [可执行的正常运行的例子](https://github.com/Guang1234567/rust_android_common_lib) 在这里.




