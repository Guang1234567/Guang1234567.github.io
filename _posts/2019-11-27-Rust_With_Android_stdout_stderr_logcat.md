---
layout:       post
title:        "Rust 与 Android 之重定向 stdout stderr 到 logcat"
subtitle:     ""
date:         2019-11-27 00:00:03
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

当在 Android 平台上运行 Rust 二进制代码时, 在 logcat 打印 rust 代码的崩溃堆栈信息和在 rust 代码中打印 log 到 logcat 都是不可避免的.

由于 rust 默认使用 `stdout  stderr` 来 输出这些信息, so 本文描述如何在 rust 中重定向 stdout stderr 到 logcat.

 [可执行的正常运行的例子](https://github.com/Guang1234567/rust_android_common_lib) 在这里.

## 开始

### step 1:  依赖 `log`, `log-panic`, `android_logger` 这三个 rust 库.

**File:  `Cargo.toml`**

```toml
[package]
name = "rust_android_common_lib"
version = "0.1.0"
authors = ["Guang1234567 <abc@163.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
log = {version = "0.4", features = ["std"]}
log-panics = {version = "2.0.0", features = ["with-backtrace"]}

[target.'cfg(target_os="android")'.dependencies]
android_logger = "0.8"

[profile.release]
lto = true

[lib]
# 编译的动态库名字
name = "greetings"
# 编译类型 cdylib 指定为动态库
crate-type = ["cdylib", "staticlib"]
```

---
简要说明:

```toml
[dependencies]
log = {version = "0.4", features = ["std"]}
log-panics = {version = "2.0.0", features = ["with-backtrace"]}

[target.'cfg(target_os="android")'.dependencies]
android_logger = "0.8"
```

`features = ["std"]` 和 `features = ["with-backtrace"]` 是什么👻❓

rust 库其实是可以包含多个 feature (功能),  然后其中某几个 feature 可以人为地自由组合成一些 "features 组合方案", so `with-backtrace` 就是某几个 feature 的组合方案名.

这有点类似于Android平台的 `库本身的多渠道`打包.

### step 2:   借助上面的库在 rust 中往 logcat 打印.

**File: 	`logger.rs`**

```rust
use std::io::{Result as IoResult, Write};
use crate::error::LibResult;
//use android_log::AndroidLogger;
use log::Level;
use android_logger::{Config, FilterBuilder};

pub struct MyLogger {}

impl MyLogger {
    pub fn init<S: Into<Vec<u8>>>(tag: S) -> LibResult<()> {
        //MY_ANDROID_LOGGER.init()?;
        android_logger::init_once(
            Config::default()
                .with_min_level(Level::Trace) // limit log level
                .with_tag(tag) // logs will show under mytag tag
                .with_filter( // configure messages for specific crate
                              FilterBuilder::new()
                                  .parse(env!("RUST_LOG_FILTER","You forgot to export RUST_LOG_FILTER")) // `greetings` is dynamic-share-lib-name in Cargo.toml
                                  .build())
        );

        log_panics::init();
        Ok(())
    }

    pub fn new() -> Self {
        Self {}
    }
}

impl Write for MyLogger {
    fn write(&mut self, buf: &[u8]) -> IoResult<usize> {
        info!("{}", String::from_utf8_lossy(buf));
        Ok(buf.len())
    }

    fn flush(&mut self) -> IoResult<()> {
        log::logger().flush();
        Ok(())
    }
}
``` 

如何使用?

```rust
	    let result = MyLogger::init(env!("RUST_LOG_TAG","You forgot to export RUST_LOG_TAG"));
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
```

还记得那个 `.env.android` 吗? 里面有这么一个配置:

```bash
# ...

# log filter
# `info,database::orm=warn` means turn on global `info` logging and also `warn` for `database::orm`
RUST_LOG_FILTER=debug,greetings::database::orm=info
```

所以上面的 `trace!("MyLogger::init success !!!");` 不会打印到 logcat !


### step3:  将崩溃堆栈信息打印到 logcat

借助 `log-panics = {version = "2.0.0", features = ["with-backtrace"]}`

使用方法:

下面代码 `log_panics::init(); ` 与 logger 一起初始化了.

``` rust
impl MyLogger {
    pub fn init<S: Into<Vec<u8>>>(tag: S) -> LibResult<()> {
        //MY_ANDROID_LOGGER.init()?;
        android_logger::init_once(
            Config::default()
                .with_min_level(Level::Trace) // limit log level
                .with_tag(tag) // logs will show under mytag tag
                .with_filter( // configure messages for specific crate
                              FilterBuilder::new()
                                  .parse(env!("RUST_LOG_FILTER","You forgot to export RUST_LOG_FILTER")) // `greetings` is dynamic-share-lib-name in Cargo.toml
                                  .build())
        );

        log_panics::init();   //   here
        Ok(())
    }

    pub fn new() -> Self {
        Self {}
    }
}
```

原理简要说明一下:

一般在 android 会这样子主动打印堆栈信息 :

```java
Log.e("demo", "主动打印堆栈信息", new Exception());
```

其中 `new Exception()` 是重点! 借助它可以在`runtime(运行时)`获取堆栈信息!

那么 rust 对应的东西就是  `backtrace` 库.

通过 `Backtrace::new()` 可以拿到堆栈信息, 然后借助上面的 `android-logger` 打印到 logcat.

所以 `log-panics = {version = "2.0.0", features = ["with-backtrace"]}` 会有一个 `features = ["with-backtrace"]` .

```rust
// log-panic 源码

pub fn init() {
    panic::set_hook(Box::new(|info| {
        let backtrace = Backtrace::new();      // here

        let thread = thread::current();
        let thread = thread.name().unwrap_or("unnamed");

        let msg = match info.payload().downcast_ref::<&'static str>() {
            Some(s) => *s,
            None => match info.payload().downcast_ref::<String>() {
                Some(s) => &**s,
                None => "Box<Any>",
            },
        };

        match info.location() {
            Some(location) => {
                error!(
                    target: "panic", "thread '{}' panicked at '{}': {}:{}{:?}",
                    thread,
                    msg,
                    location.file(),
                    location.line(),
                    Shim(backtrace)
                );
            }
            None => {
                error!(
                    target: "panic",
                    "thread '{}' panicked at '{}'{:?}",
                    thread,
                    msg,
                    Shim(backtrace)         //here
                )
            }
        }
    }));

```

其中 `Shim(backtrace)`  是一个 "打印动作的 Wrapper", 类似于 haskell 的 newtype,  这里不展开描述了.

``` rust
struct Shim(Backtrace);

impl fmt::Debug for Shim {

    #[cfg(feature = "with-backtrace")]
    fn fmt(&self, fmt: &mut fmt::Formatter) -> fmt::Result {
        write!(fmt, "\n{:?}", self.0)
    }

    #[cfg(not(feature = "with-backtrace"))]
    fn fmt(&self, _: &mut fmt::Formatter) -> fmt::Result {
        Ok(())
    }
}
```
另外 `#[cfg(feature = "with-backtrace")]`  和  `#[cfg(not(feature = "with-backtrace"))]` 这两个宏的作用是根据 `log-panics = {version = "2.0.0", features = ["with-backtrace"]}` 有没有 `features = ["with-backtrace"]` 来选择编译哪个方法.

## 总结

在 rust 代码中, 如何重定向 stdout stderr 到 logcat 应该知道了吧.




