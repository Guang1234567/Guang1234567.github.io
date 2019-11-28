---
layout:       post
title:        "搭建 Rust 与 Android 的项目"
subtitle:     ""
date:         2019-11-27 00:00:01
author:       "Guang1234567"
header-img:   "img/home-bg-o.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
music-enable: true
categories:
    - ndk
    - rust
    - android
tags:
    - ndk
    - rust
    - android

---


> 此文章不允许转载, 违者必究...

**目录:**

* content
{:toc}


## 简概
   文章主要描述如何搭建 `Rust` 与 `Android` 项目.
   
   其实就是 Android NDK 开发, 只不过把 `C++` 换成 `Rust`, 编译并使用一个 `Rust` 写的 `librust.so` 动态库.
   
   其中搭建过程中会用到一些 `Rust 生态`中的库来辅助搭建, 但由于要移植到 `Android 平台`, 某些库会被 fork 下来进行修改一番.

   [可执行的正常运行的例子](https://github.com/Guang1234567/rust_android_common_lib) 在这里.
   
## 开始

### Step 1:  如何分别创建 Android 和 Rust 的项目

"创建分别创建 Android 和 Rust 项目",  这个 google 一下.

参考 [可执行的正常运行的例子](https://github.com/Guang1234567/rust_android_common_lib)

```bash
# 目录结构如下:

root_of_rust_project $   tree -L 1
.
├── Cargo.lock
├── Cargo.toml
├── README.md
├── build_for_android.sh
├── diesel.toml
├── example
│   └── android							#####    android project root dir
│       ├── README.md
│       ├── android.iml
│       ├── app
│       ├── build
│       ├── build.gradle
│       ├── gradle
│       ├── gradle.properties
│       ├── gradlew
│       ├── gradlew.bat
│       ├── local.properties
│       └── settings.gradle
│   └── ios                                              #####    ios project root dir
├── lldb
├── migrations
├── rust_android_common_lib.iml
├── src
├── target
└── third_part_libs
```

用过 `ReactNative` 和 `Flutter` 的同学应该感觉项目的目录结构似曾相识.

---

**开发环境:**

- OSX 10.14.X   (用 Linux 也可以, 不推荐 Windows)
- Android studio 3.4
- Clion 2018 + `Jetbrain 官方 rust 插件`
- rust 1.40.0-nightly  (直接 nightly)

```bash
root_of_rust_project $   rustc --version
rustc 1.40.0-nightly (7979016af 2019-10-20)
```
### Step 2: 添加用于编译 rust 库的 gradle task

  修改 example/android/app/build.gradle 
  
```gradle
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions'

// ...

android {
    
// ...

    def buildType = "debug"
    applicationVariants.all { variant ->
        buildType = variant.buildType.name // Sets the current build type
    }

    //copying so files

    task compileNativeLib(type: Exec) {
        doFirst {
            println("\n============================== Start Compile Native Lib ==============================")
            println("buildType = ${buildType}")
            workingDir "./../../.."
            if (buildType.equalsIgnoreCase("debug")) {
                commandLine "./build_for_android.sh"
            } else {
                commandLine "./build_for_android.sh", "--${buildType}"
            }
        }
        doLast {
            println("\n============================== Finish Compile Native Lib ==============================\n")
        }
    }

    task copyArm8NativeLib() {
        doLast {
            copy {
                from "./../../../target/aarch64-linux-android/${buildType}/libgreetings.so"
                into "src/main/jniLibs/arm64-v8a"
            }
        }
    }

    task copyArm7NativeLib() {
        doLast {
            copy {
                from "./../../../target/armv7-linux-androideabi/${buildType}/libgreetings.so"
                into "src/main/jniLibs/armeabi-v7a"
            }
        }
    }

    task copyX86_64NativeLib() {
        doLast {
            copy {
                from "./../../../target/x86_64-linux-android/${buildType}/libgreetings.so"
                into "src/main/jniLibs/x86_64"
            }
        }
    }

    task copyNativeLib() {

        dependsOn compileNativeLib

        dependsOn copyArm8NativeLib
        dependsOn copyArm7NativeLib
        dependsOn copyX86_64NativeLib
    }

    //add dependencies
    tasks.withType(JavaCompile) {
        compileTask -> compileTask.dependsOn copyNativeLib
    }
}

// ...
```

上面脚本主要是把 so 拷贝到 `jniLibs` 中去, 如下所示

``` bash

./example
└── android
    ├── app
    │   └── src
    │       ├── main
    │       │   ├── AndroidManifest.xml
    │       │   ├── cpp
    │       │   ├── java
    │       │   ├── jniLibs
    │       │   │   ├── arm64-v8a
    │       │   │   │   └── libgreetings.so                 ###  here
    │       │   │   ├── armeabi-v7a
    │       │   │   │   └── libgreetings.so                 ###  here
    │       │   │   └── x86_64
    │       │   │       └── libgreetings.so                 ###  here
```

### Step 2:  新建 `build_for_android.sh` 给上一步的 gradle task 调用, 用于编译 rust 库

**File:		`build_for_android.sh`**

```bash
#!/bin/bash

echo -e "\n*****************    cargo +nightly build --target aarch64-linux-android $*     ******************\n"

cargo +nightly build --target aarch64-linux-android $*

echo -e "\n*****************     cargo +nightly build --target x86_64-linux-android $*    ******************\n"

cargo +nightly build --target x86_64-linux-android $*

echo -e "\n*****************    cargo +nightly build --target armv7-linux-androideabi $*   ******************\n"

cargo +nightly build --target armv7-linux-androideabi $*

```

### Step 3:  开始编写相应的 rust 代码和 kotlin 代码

**File (rust):		`lib.rs`**   

用于演示,详情  [可执行的正常运行的例子](https://github.com/Guang1234567/rust_android_common_lib).

```rust

#[cfg(target_os = "android")]
#[allow(non_snake_case)]
// extern crate android_log;
extern crate android_logger;
#[macro_use]
extern crate diesel;
#[macro_use]
extern crate diesel_migrations;
extern crate dotenv;
extern crate jni;
#[macro_use]
extern crate lazy_static;
#[macro_use]
extern crate log;
extern crate log_panics;


use std::error::Error;

use dotenv::dotenv;
use jni::JNIEnv;
use jni::objects::{JClass, JString};
use jni::sys::jstring;

use database::orm::do_some_db_op;
use database::sqlite::SqliteHelper;
use load_dotenv::{load_dotenv, load_dotenv_from_filename};
use logger::MyLogger;


// load your .env.android file at compile time
load_dotenv_from_filename!(".env.android");

#[no_mangle]
pub unsafe extern fn Java_com_rust_example_android_MainActivity_rustSqlite(
    env: JNIEnv,
    _: JClass,
    database_path: JString) -> jstring {

    //dotenv().ok();

    let result = MyLogger::init("app_rust_sql_123");
    match result {
        Ok(_) => {
            error!("MyLogger::init success !!!");
            warn!("MyLogger::init success !!!");
            info!("MyLogger::init success !!!");
            debug!("MyLogger::init success !!!");
            trace!("MyLogger::init success !!!");
        }
        Err(err) => error!("{}", err.description()),
    }

    let input: String =
        env.get_string(database_path).expect("Couldn't get java string!").into();

    let database_url: &str = env!("DATABASE_URL","You forgot to export DATABASE_URL path");
    let result = do_some_db_op(format!("{}_{}", input, database_url));
    match result {
        Ok(_) => info!("do_some_db_op:  success !!!!!!!!!!!!!"),
        Err(err) => error!("do_some_db_op: {}", err.description()),
    }

    let output = env.new_string(format!("Hello, {} from Rust!", input))
        .expect("Couldn't create java string!");
    output.into_inner()
}
```  

**File:		`MainActivity.kt`

```kotlin
package com.rust.example.android

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import kotlinx.android.synthetic.main.activity_main.*

class MainActivity : AppCompatActivity() {

    // ...
   
    external fun rustSqlite(dbPath: String): String
    
    // ...

    companion object {
        // Used to load the 'native-lib' library on application startup.
        init {
            // <= 5.0
            System.loadLibrary("greetings")
            // >= 6.0
            System.loadLibrary("native-lib")
        }
    }
}
```

## 总结
   
描述 rust 和 android 如何结合构建的一些重要部分.