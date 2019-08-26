---
layout:       post
title:        "Android NDK  打印 logger"
subtitle:     " 如何在 native code 打印 log 到 Android Logcat?"
date:         2019-08-26 00:00:00
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
    - Android
    - NDK
    - Logcat
---


> 此文章不允许转载, 违者必究...

> 目的:

**目录:**

* content
{:toc}

## (一) 直接上代码

```cpp
#pragma once

#include <jni.h>
#include <android/log.h>


#define __FILENAME__ (strrchr(__FILE__, '/') ? strrchr(__FILE__, '/') + 1 : __FILE__)


#ifndef MY_LOG_TAG
#   define MY_LOG_TAG __FILENAME__
#endif


#ifndef MY_LOG_LEVEL
#   define MY_LOG_LEVEL ANDROID_LOG_VERBOSE
#endif


#define LOG_NOOP (void) NULL


#define LOG_PRINT_TAG(level, tag, fmt, ...)    \
    if(tag == NULL || tag[0] == '\0') {     \
        __android_log_print(level, MY_LOG_TAG, "(%s:%u) %s: " fmt,  \
            __FILE__, __LINE__, __PRETTY_FUNCTION__, ##__VA_ARGS__); \
    } else {                                \
        __android_log_print(level, tag, "(%s:%u) %s: " fmt,         \
            __FILE__, __LINE__, __PRETTY_FUNCTION__, ##__VA_ARGS__); \
    }


#define LOG_PRINT(level, fmt, ...) \
    LOG_PRINT_TAG(level, MY_LOG_TAG, fmt, ##__VA_ARGS__)


#if ANDROID_LOG_VERBOSE >= MY_LOG_LEVEL
#   define LOG_V(tag, fmt, ...) \
        LOG_PRINT_TAG(ANDROID_LOG_VERBOSE, tag, fmt, ##__VA_ARGS__)
#else
#   define LOG_V(...) LOG_NOOP
#endif


#if ANDROID_LOG_DEBUG >= MY_LOG_LEVEL
#   define LOG_D(tag, fmt, ...) \
        LOG_PRINT_TAG(ANDROID_LOG_DEBUG, tag, fmt, ##__VA_ARGS__)
#else
#   define LOG_D(...) LOG_NOOP
#endif


#if ANDROID_LOG_INFO >= MY_LOG_LEVEL
#   define LOG_I(tag, fmt, ...) \
        LOG_PRINT_TAG(ANDROID_LOG_INFO, tag, fmt, ##__VA_ARGS__)
#else
#   define LOG_I(...) LOG_NOOP
#endif


#if ANDROID_LOG_WARN >= MY_LOG_LEVEL
#   define LOG_W(tag, fmt, ...) \
        LOG_PRINT_TAG(ANDROID_LOG_WARN, tag, fmt, ##__VA_ARGS__)
#else
#   define LOG_W(...) LOG_NOOP
#endif


#if ANDROID_LOG_ERROR >= MY_LOG_LEVEL
#   define LOG_E(tag, fmt, ...) \
        LOG_PRINT_TAG(ANDROID_LOG_ERROR, tag, fmt, ##__VA_ARGS__)
#else
#   define LOG_E(...) LOG_NOOP
#endif


#if ANDROID_LOG_FATAL >= MY_LOG_LEVEL
#   define LOG_FATAL(tag, fmt, ...) \
        LOG_PRINT_TAG(ANDROID_LOG_FATAL, tag, fmt, ##__VA_ARGS__)
#else
#   define LOG_FATAL(...) LOG_NOOP
#endif


#if ANDROID_LOG_FATAL >= MY_LOG_LEVEL
#   define LOG_ASSERT(expression, fmt, ...) \
        if (!(expression)) { \
            __android_log_assert(#expression, MY_LOG_TAG, "(%s:%u) %s: " fmt, \
        __FILE__, __LINE__, __PRETTY_FUNCTION__, ##__VA_ARGS__); \
    }
#else
#   define LOG_ASSERT(...) LOG_NOOP
#endif
```

## (二) 调整打印 Log 的 Level

请看上面代码的 

  ```cpp
  
#ifndef MY_LOG_TAG
#   define MY_LOG_TAG __FILENAME__
#endif

  ```
  
另外 android log level 源码

```cpp

typedef enum android_LogPriority {
    ANDROID_LOG_UNKNOWN = 0,
    ANDROID_LOG_DEFAULT = 1,    /* only for SetMinPriority() */
    ANDROID_LOG_VERBOSE = 2,
    ANDROID_LOG_DEBUG = 3,
    ANDROID_LOG_INFO = 4,
    ANDROID_LOG_WARN = 5,
    ANDROID_LOG_ERROR = 6,
    ANDROID_LOG_FATAL = 7,
    ANDROID_LOG_SILENT = 8,     /* only for SetMinPriority(); must be last */
} android_LogPriority;

```
  
- 假如使用  makefile(Android.mk) 来构建

```make
# 定义基于构建类型的默认日志等级
ifeq ($(APP_OPTIM),release)
	MY_LOG_LEVEL := 4
else
	MY_LOG_LEVEL := 2
endif
```

# 追加编译标记
```make
LOCAL_CFLAGS += -DMY_LOG_LEVEL=${MY_LOG_LEVEL}
```


- 假如使用 CMAKE 来构建 (优先使用这种构建方式)

```cmake
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -DMY_LOG_LEVEL=2")
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -DMY_LOG_LEVEL=4")
```

[CMAKE_C_FLAGS_XXX 的文档链接](https://cmake.org/cmake/help/v3.7/variable/CMAKE_BUILD_TYPE.html#variable:CMAKE_BUILD_TYPE)

## (三) 延伸

- 如何在 cpp 文件中区分 release or debug ?

Begin in NDK r8b, you can check

```cpp
#ifdef NDEBUG
// this is "release"
#else
// this is "debug"
#endif
```






