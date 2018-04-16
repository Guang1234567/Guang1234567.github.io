---
layout:       post
title:        "Js 与 Android 之间是如何交流的"
subtitle:     "W 两个世界"
date:         2018-03-29 00:00:00
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

> 目的: 介绍诸如 JsBridge 库所用到的最核心的 API 和流程.

**目录:**

* content
{:toc}


## 第一种方式

Js 通过调用

```JavaScript
window.top.prompt("A_Request");
```

把 A_Request 发送到 Android 端, 此时下面 Android 端的接口

```java
WebChromeClient#onJsPrompt("A_Request") {

    // 处理 A_Request 得到 A_Response
}
```

被调用, 在此方法内填写处理 A_Request 得到 A_Response的逻辑代码.  接着通过下面 Android 端的接口

```java
android.webkit.WebView#loadUrl(A_Response)
```

把 A_Response 返回给 Js 端.




## 第二种方式

Js 通过调用

```JavaScript
function _createQueueReadyIframe(doc) {
        messagingIframe = doc.createElement('iframe');
        messagingIframe.style.display = 'none';
        doc.documentElement.appendChild(messagingIframe);
    }
var messagingIframe = _createQueueReadyIframe(document);

messagingIframe.src = "A_Request";
```

把 A_Request 发送到 Android 端, 此时下面 Android 端的接口

```java
android.webkit.WebViewClient#shouldOverrideUrlLoading(android.webkit.WebView, java.lang.String)("A_Request") {

    // 处理 A_Request 得到 A_Response
}
```

被调用, 在此方法内填写处理 A_Request 得到 A_Response的逻辑代码.  接着通过下面 Android 端的接口

```java
android.webkit.WebView#loadUrl(A_Response)
```

把 A_Response 返回给 Js 端.


## 第三种方式

通过 @JavascriptInterface注解, 不过有安全隐患一般都不会用的了...

1) js调用android中的方法

```java
private void init(Context context){
        WebSettings setting =getSettings();
        setting.setJavaScriptEnabled(true);//支持js
        setWebViewClient(new WebClient(this));
        setWebChromeClient(new WebChromeClient());
        //添加js调用android的方法这是关键 前者是个对象，后者是个字符串 在js中是在window.android可以直接获取到
        addJavascriptInterface(new JavaScriptinterface(context,this),
                "android");
        //这是开启js的调试下面再讲
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            setWebContentsDebuggingEnabled(true);
        }
    }


  public class JavaScriptinterface{
    Context context;

    public JavaScriptinterface(Context c) {
        context = c;
    }
    //这个注解可以点击进去看官方描述 是 带有此注释的标记可用于JavaScript代码
      @JavascriptInterface
      public void toastMessage(String message) {
          Toast.makeText(getApplicationContext(), message, Toast.LENGTH_LONG).show();
      }
  }
```

```JavaScript
<script>
function showToast() {

    android.toastMessage('hhh');
}
</script>
```


2) android 调用 js 的方法 (如何把结果返回到 js)

```javascript
<script>
function jsalert(data) {
    alert(data);
}
</script>
```

```java
//java这样调用即可 方法前需要加javascript
webview.loadUrl('javascript:jsalert('111')')
```

## 第四种方式

> 一句话概括: 把android端当成服务器来接受并处理来自 Js 端的请求

启动一个本地server，端口号是：`8888`，那么在手机上，网页就可以通过：`http://127.0.0.1:8888` 访问这个server，server接收到请求就可以进行一些native的操作，对于需要回调数据的，就通过返回请求内容来执行，比如：

获取个定位信息，js执行`$.get('http://127.0.0.1:8888/getGeoLocation?callback=cbname')`
server收到请求之后，调用native方法，获取GPS的定位信息，然后将数据通过response：`window.cbname&&cbname({xxx})`给页面返回定位数据


## 第五种方式

https://developer.android.com/topic/security/best-practices.html#webview

#### Use WebView objects carefully

Whenever possible, load only whitelisted content in
`[WebView](https://developer.android.com/reference/android/webkit/WebView.html)` objects. In other words, the
`[WebView](https://developer.android.com/reference/android/webkit/WebView.html)` objects in your app shouldn't allow users to
navigate to sites that are outside of your control.

In addition, you should never enable
[JavaScript interface
support](https://developer.android.com/guide/webapps/webview.html#UsingJavaScript) unless you completely control and trust the content in your app's
`[WebView](https://developer.android.com/reference/android/webkit/WebView.html)` objects.

#### Use HTML message channels

If your app must use JavaScript interface support on devices running Android
6.0 (API level 23) and higher, use HTML message channels instead of
`[evaluateJavascript()](https://developer.android.com/reference/android/webkit/WebView.html#evaluateJavascript(java.lang.String,%20android.webkit.ValueCallback%3Cjava.lang.String%3E))` to
communicate between a website and your app, as shown in the following code
snippet:

```java
WebView myWebView = (WebView) findViewById(R.id.webview);  
  // messagePorts[0] and messagePorts[1] represent the two ports.  // They are already tangled to each other and have been started.  WebMessagePort[] channel = myWebView.createWebMessageChannel();  
  // Create handler for channel[0] to receive messages.  
channel[0].setWebMessageCallback(new WebMessagePort.WebMessageCallback() {  
    @Override  
    public void onMessage(WebMessagePort port, WebMessage message) {  
         Log.d(TAG, "On port " + port + ", received this message: " + message);  
    }  });  
  // Send a message from channel[1] to channel[0].  
channel[1].postMessage(new WebMessage("My secure message"));
```

**Related info:**

* `[WebMessage](https://developer.android.com/reference/android/webkit/WebMessage.html)`
* `[WebMessagePort](https://developer.android.com/reference/android/webkit/WebMessagePort.html)`

* `[App Sample](https://commonsware.com/blog/2017/01/23/replacing-addjavascriptinterface-html-message-channels.html)`



## 其他方式

应该还有其他方式, 我目前还不知道, 之后看到就再补充

