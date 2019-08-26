---
layout:       post
title:        "使用 Haskell 与 Android NDK 进行 Linux 原生开发"
subtitle:     "用 Haskell 替代 C++ 来进行Android 平台上的原生开发"
date:         2019-08-27 00:00:00
author:       "Guang1234567"
header-img:   "img/home-bg-o.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
music-enable: true
categories:
    - Android
    - NDK
tags:
   - Android
    - NDK
---


> 此文章不允许转载, 违者必究...

**目录:**

* content
{:toc}


## 背景
Android 是 Google 公司基于 Linux 平台开发的开源手机操作系统, 自然要对 C C++ 提供原生支持. 通过 NDK, Android应用程序可以非常方便地实现 Java 与 C/C++代码的相互沟通.

随着语言的发展, 近些年来出现了一些诸如 Rust, Haskell, Go等新的系统编程语言对 C/C++ 的系统编程语言地位发起了强烈的攻击. (Kotlin-Native 这门技术也能实现 native 开发, 不过要依赖专门的垃圾回收器进行内存管理)

另外, 通过相应的交叉编译链, 使它们在 Android 平台上进行 NDK 开发成为了新的可能.

本文的主角是 Haskell, 一门纯函数式编程语言.

## 正文
本文会实现一个 `JNI` 的例子, 其中相关的 so 库是使用 `少量 C++ 代码  + Haskell` 来实现的.

### kotlin 层 (java 层)

这是 Activity 代码 (里面包含一个 JNI 接口) : 

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        var rxPermissions: RxPermissions = RxPermissions(this)

        rxPermissions
                .requestEachCombined(Manifest.permission.WRITE_EXTERNAL_STORAGE)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe({ permission ->
                    // will emit 2 Permission objects
                    if (permission.granted) {
                        // `permission.name` is granted !
                        //Toast.makeText(this@MainActivity, "已成功授予所有权限!", Toast.LENGTH_SHORT).show()
                        doSthAfterAllPermissionGranted()
                    } else if (permission.shouldShowRequestPermissionRationale) {
                        // Denied permission without ask never again
                    } else {
                        // Denied permission with ask never again
                        // Need to go to the settings
                    }
                })
    }

    private fun doSthAfterAllPermissionGranted() {
    			// 搜索 "/sdcard/*.txt" 下的 text 文档.
                 Log.w("demo", "${namesMatchingJNI("/sdcard/*.txt").joinToString()}}")
    }
    
    // 使用通配符进行模糊匹配搜索 sdcard 的相关文件
    external fun namesMatchingJNI(path: String): Array<String>
}
```

### native 层 (C++ 和 Haskell)

- 先来看相关 C++ 代码:

```cpp
#include <jni.h>

#include <unistd.h>
#include <sstream>
#include <string>

#include "ghcversion.h"
#include "HsFFI.h"
#include "Rts.h"

#include "my_log.h"
#include "Lib_stub.h"
#include "FileSystem_stub.h"
#include "ForeignUtils_stub.h"
#include "android_hs_common.h"

extern "C" {

JNIEXPORT jobjectArray
JNICALL
Java_com_xxx_yyy_MainActivity_namesMatchingJNI(
        JNIEnv *env,
        jobject thiz,
        jstring path) {

    LOG_ASSERT(NULL != env, "JNIEnv cannot be NULL.");
    LOG_ASSERT(NULL != thiz, "jobject cannot be NULL.");
    LOG_ASSERT(NULL != path, "jstring cannot be NULL.");

    const char *c_value = env->GetStringUTFChars(path, NULL);
    CStringArrayLen *cstrArrLen = static_cast<CStringArrayLen *>(namesMatching(
            const_cast<char *>(c_value)));

    char **result = cstrArrLen->cstringArray;
    jsize len = cstrArrLen->length;

    env->ReleaseStringUTFChars(path, c_value);
    jobjectArray strs = env->NewObjectArray(len, env->FindClass("java/lang/String"),
                                            env->NewStringUTF(""));
    for (int i = 0; i < len; i++) {
        jstring str = env->NewStringUTF(result[i]);
        env->SetObjectArrayElement(strs, i, str);
    }
    // freeCStringArray frees the newArray pointer created in haskell module
    freeNamesMatching(cstrArrLen);
    return strs;
}

}
```

上面这代码只是少量的 C++ 代码, 主要的功能是稍微地封装调用 `Haskell` 实现的 `namesMatching` 函数.  

因为 JNI 不能直接调用 `Haskell` 代码实现的函数, 借助 `FFI` 实现间接调用 (跟 `Rust` 一样):

```ruby

JVM -->  JNI  --> C++ -->  FFI --> Haskell

```

- 接着使用 `Haskell` 实现的 `namesMatching` 函数:

```haskell
module Android.FileSystem
    ( matchesGlob
    , namesMatching
    ) where

import Android.ForeignUtils
import Android.Log
import Android.Regex.Glob (globToRegex, isPattern)

import Control.Exception (SomeException, handle)
import Control.Monad (forM)

import Foreign
import Foreign.C

import System.Directory (doesDirectoryExist, doesFileExist, getCurrentDirectory, getDirectoryContents)
import System.FilePath ((</>), dropTrailingPathSeparator, splitFileName)

import Text.Regex.Posix ((=~))

matchesGlob :: FilePath -> String -> Bool
matchesGlob name pat = name =~ globToRegex pat

_matchesGlobC name glob = do
    name <- peekCString name
    glob <- peekCString glob
    return $ matchesGlob name glob

doesNameExist :: FilePath -> IO Bool
doesNameExist name = do
    fileExists <- doesFileExist name
    if fileExists
        then return True
        else doesDirectoryExist name

listMatches :: FilePath -> String -> IO [String]
listMatches dirName pat = do
    dirName' <-
        if null dirName
            then getCurrentDirectory
            else return dirName
    handle (const (return []) :: (SomeException -> IO [String])) $ do
        names <- getDirectoryContents dirName'
        let names' =
                if isHidden pat
                    then filter isHidden names
                    else filter (not . isHidden) names
        return (filter (`matchesGlob` pat) names')

isHidden ('.':_) = True
isHidden _ = False

listPlain :: FilePath -> String -> IO [String]
listPlain dirName baseName = do
    exists <-
        if null baseName
            then doesDirectoryExist dirName
            else doesNameExist (dirName </> baseName)
    return
        (if exists
             then [baseName]
             else [])

namesMatching :: FilePath -> IO [FilePath]
namesMatching pat
    | not $ isPattern pat = do
        exists <- doesNameExist pat
        return
            (if exists
                 then [pat]
                 else [])
    | otherwise = do
        case splitFileName pat
            -- 在只有文件名的情况下, 只在当前目录查找.
              of
            ("", baseName) -> do
                curDir <- getCurrentDirectory
                listMatches curDir baseName
            -- 在包含目录的情况下
            (dirName, baseName)
                -- 由于目录本身可能也是一个符合 glob 模式的字符床, 如(/foo*bar/far?oo/abc.txt)
             -> do
                dirs <-
                    if isPattern dirName
                        then namesMatching (dropTrailingPathSeparator dirName)
                        else return [dirName]
                -- 经过上面操作, 拿到所有符合规则的目录
                let listDir =
                        if isPattern baseName
                            then listMatches
                            else listPlain
                pathNames <-
                    forM dirs $ \dir -> do
                        baseNames <- listDir dir baseName
                        return (map (dir </>) baseNames)
                return (concat pathNames)

_namesMatchingC :: CString -> IO (Ptr CStringArrayLen)
_namesMatchingC filePath = do
    filePath' <- peekCString filePath
    pathNames <- namesMatching filePath'
    pathNames' <- forM pathNames newCString :: IO [CString]
    newCStringArrayLen pathNames'

_freeNamesMatching :: Ptr CStringArrayLen -> IO ()
_freeNamesMatching ptr = do
    cstrArrLen <- peekCStringArrayLen ptr
    let cstrArrPtr = getCStringArray cstrArrLen
    freeCStringArray cstrArrPtr
    free ptr
    return ()

foreign export ccall "matchesGlob" _matchesGlobC :: CString -> CString -> IO Bool

foreign export ccall "namesMatching" _namesMatchingC :: CString -> IO (Ptr CStringArrayLen)

foreign export ccall "freeNamesMatching" _freeNamesMatching :: Ptr CStringArrayLen -> IO ()
```
我们借助[交叉编译链](https://github.com/Guang1234567/ghc-cross-compiler-for-android/tree/v2-ghc-8.6.5-20190531-aarch64-linux-android), 把这段 Haskell 代码编译成静态库, 名为 `libHSandroid-hs-mobile-common-0.1.0.0-inplace-ghc8.6.5.a`

其中 `foreign export ccall "namesMatching" _namesMatchingC :: CString -> IO (Ptr CStringArrayLen)` 是 `FFI` 接口, 暴露给 `C++` 代码调用.

- 接着, 在 Android app 主项目的 cmake 配置文件中 link 上面这个静态库  `libHSandroid-hs-mobile-common-0.1.0.0-inplace-ghc8.6.5.a`

```cmake

add_library(lib_hs STATIC IMPORTED)
set_target_properties(lib_hs
        PROPERTIES
        IMPORTED_LOCATION $ENV{HOME}/dev_kit/src_code/android-hs-mobile-common/dist-newstyle/build/${ALIAS_1_ANDROID_ABI}/ghc-8.6.5/android-hs-mobile-common-0.1.0.0/build/libHSandroid-hs-mobile-common-0.1.0.0-inplace-ghc8.6.5.a)

target_link_libraries( # Specifies the target library.
        native-lib

        # Links the target library to the log library
        # included in the NDK.
        ...
        
        lib_hs
        
        ...
        )
```

- 最后, 编译运行 Android app 主项目:

运行的结果(通过打印 log 来呈现):

```ruby
// logcat

2019-08-26 18:14:43.662 12344-12344/com.xxx.yyy.helloworld W/demo: /sdcard/jl.txt, /sdcard/ceshitest.txt, /sdcard/treeCallBack.txt}

```

## 总结
   Android 的 NDK 开发并不是只有 C/C++, 还有别的一方天地.  
   特别是使用 C/C++ 写业务逻辑的场景, 开发效率特别低.





