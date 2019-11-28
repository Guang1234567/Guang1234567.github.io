---
layout:       post
title:        "Rust 与 Android 之断点调试"
subtitle:     ""
date:         2019-11-27 00:00:04
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

> 目的:

**目录:**

* content
{:toc}


## 简概

是不是找遍了网站都没找到呀? 哈哈.

 [可执行的正常运行的例子](https://github.com/Guang1234567/rust_android_common_lib) 在这里.

## 开始

直接使用例子来演示,  lldb gdb 都支持, 相关步骤已经写成脚本了!

位于

```bash
 root_of_project  $  tree -L 5
.
├── Cargo.lock
├── Cargo.toml
├── README.md
├── build_for_android.sh
├── diesel.toml
├── lldb
│   ├── bin
│   │   ├── clion_remote_debug_cfg.png
│   │   ├── gdb_rust_arm64-v8a.sh
│   │   ├── gdb_rust_x86_64.sh
│   │   ├── lldb_rust_arm64-v8a.sh
│   │   ├── resolve_android_waiting_for_debugger.sh
│   │   ├── start_gdb_server_arm64-v8a.sh
│   │   └── start_gdb_server_x86_64.sh
```

### gdb

- 第一步: 已 debug 模式启动例子 app

```bash
adb shell am start -n "com.rust.example.android.debug/com.rust.example.android.MainActivity" -a android.intent.action.MAIN -c android.intent.category.LAUNCHER -D
```

注意那个重要的 `-D` 参数 (⊙﹏⊙)b)


- 第二步:  在手机的 android os 上运行 gdb-server 

```bash

cd  lldb/bin

./gdb_rust_arm64-v8a.sh  "com.rust.example.android.debug"

```

- 第三步:  配置 CLion的 `gdb remote debug`

![配置 CLion的 `gdb remote debug`](https://raw.githubusercontent.com/Guang1234567/rust_android_common_lib/master/lldb/bin/clion_remote_debug_cfg_ gdb.png)


- 第四步:  运行第三步的配置, 直到 Clion 的 "debug panel" 显示 "Connected" 之类的字样,  不是 "Connecting" !   "Connecting" 的话请耐心等待!


- 第五步:  手机其实一直处于 "waiting for debugger attach" 的状态, 看到这个对话框的吧.

此时运行下面的脚本让被 debug block 住的 app 继续运行下去.

```bash
	
	cd   lldb/bin

	./resolve_android_waiting_for_debugger.sh    "com.rust.example.android.debug"
	
```

里面的端口号写死成 `8600`, 有时候会变的, 此时在 android studio 里打开 `example/android`, 然后 debug, 此时在 `debug pannel` 会看到这个端口号.  


最终效果图:

![](https://raw.githubusercontent.com/Guang1234567/rust_android_common_lib/master/lldb/bin/clion_remote_debug_gdb_final.png)

### lldb

- 第一步: 已 debug 模式启动例子 app

```bash
adb shell am start -n "com.rust.example.android.debug/com.rust.example.android.MainActivity" -a android.intent.action.MAIN -c android.intent.category.LAUNCHER -D
```

注意那个重要的 `-D` 参数 (⊙﹏⊙)b)


- 第二步:  在手机的 android os 上运行 lldb-server 

```bash
cd  lldb/bin

./lldb_rust_arm64-v8a.sh "com.rust.example.android.debug" "com.rust.example.android"
```

- 第三步:  有点跟 gdb 不太一样.

先在 CLion 装个叫 `Android Native Debug` 的插件

配置 CLion的 `android native debug`

![配置 CLion的 `android native debug`](https://raw.githubusercontent.com/Guang1234567/rust_android_common_lib/master/lldb/bin/clion_remote_debug_cfg_lldb.png)


> unix-abstract-connect:///com.rust.example.android.debug-0/platform-debug.sock
>  不用手打这句

- 第四步:  运行第三步的配置, 直到 Clion 的 "debug panel" 显示 "Connected" 之类的字样,  不是 "Connecting" !   "Connecting" 的话请耐心等待!


- 第五步:  手机其实一直处于 "waiting for debugger attach" 的状态, 看到这个对话框的吧.

此时运行下面的脚本让被 debug block 住的 app 继续运行下去.

```bash
	
	cd   lldb/bin

	./resolve_android_waiting_for_debugger.sh    "com.rust.example.android.debug"
	
```

里面的端口号写死成 `8600`, 有时候会变的, 此时在 android studio 里打开 `example/android`, 然后 debug, 此时在 `debug pannel` 会看到这个端口号.  

## 总结





