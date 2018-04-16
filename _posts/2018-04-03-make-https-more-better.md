---
layout:       post
title:        "HTTPS系列干货（二）：突破这5个技术难点，HTTPS会好用到飞起来"
subtitle:     ""
date:         2018-04-03 00:00:00
author:       "Guang1234567"
header-img:   "img/home-bg-o.jpg"
header-mask:  0.3
catalog:      true
multilingual: false
music-enable: true
categories:
    - Https
    - http
    - osi
    - TCP_IP
tags:
    - Https
    - http
    - osi
    - TCP_IP
---


> 转载自 http://support.upyun.com/hc/kb/article/1058927/

> 目的: 让 https 更好用更安全(但不是绝对安全...)

**目录:**

* content
{:toc}

随着互联网用户对安全性、保密性的需求日益提升，大洋彼岸的Google、Apple带头强推HTTPS，将网站升级成为HTTPS已经如风潮般席卷了国内外互联网厂商，《Google透明度报告：各大热门网站的HTTPS使用情况》中指出，相当多数量的网站已经转向HTTPS协议。

然而，对于互联网厂商来说，并不是部署了HTTPS就可以高枕无忧，要让HTTPS发挥最大作用，往往还需要与HSTS、HTTP 2.0、OSCP stapling等其它技术一起使用。本文罗列了网站在实现HTTPS中遇到的5个技术难点，帮助广大需要进行HTTPS升级和已经升级为HTTPS的互联网厂商呈现出升级HTTPS后更好的用户体验。

## HSTS

即使我们将一个HTTP网站升级为HTTPS网站，当我们在浏览器中键入以www为开头的网址时，网页并不会自动跳转为HTTPS网站，因为浏览器默认打开HTTP网站，基于此，我们就需要对HTTP的访问在服务器端做301、302或307重定向，使之跳转到HTTPS网站。

其中最常用的就是302重定向了：

302重定向是一条对网站浏览器的指令，来使浏览器显示被要求显示的URL，当一个网页经历短期的URL变化时使用。

301重定向是302重定向的升级版，301重定向是永久的重定向，搜索引擎在抓取新内容的同时也将旧的网址交换为重定向之后的网址。

但是它们存在一个共同的弱点：

当使用301、302跳转时，只要修改location指令，网站就会被劫持。

对此，国际互联网工程组织IETE推行了一种新的Web安全协议：HSTS，即307跳转。

采用HSTS协议的网站将保证浏览器始终连接到该网站的HTTPS加密版本，不需要用户手动在URL地址栏中输入加密地址。帮助网站采用全局加密，用户看到的就是该网站的安全版本。

HSTS的作用还包括阻止基于SSL Strip的中间人攻击，万一证书有错误，则显示错误，用户不能回避警告。从而能够更加有效安全的保障用户的访问。

目前主流的浏览器大多已支持HSTS，但IE 11以下版本还不支持HSTS。

## **HTTP/2**

HTTP/2即超文本传输协议2.0，是HTTP协议的升级版。由互联网工程任务组的Hypertext Transfer Protocol Bis工作小组进行开发，以SPDY为原型，经过两年多的讨论和完善最终确定。

SPDY是一种HTTP兼容协议，由Google发起，Chrome、Opera、Firefox以及Amazon Silk等浏览器均已提供支持，并对当前的HTTP协议有4个改进：

* 多路复用请求
* 对请求划分优先级
* 压缩HTTP头
* 服务器推送流（即Server Push技术）

HTTP/2相比于SPDY，优势更加明显：

* HTTP/2采用二进制格式传输数据，其在协议的解析和优化扩展上带来更多的优势和可能
* HTTP/2对消息头采用HPACK进行压缩传输，能够节省消息头占用的网络的流量
* Server Push的服务端能够更快地把资源推送给客户端。

即便如此，HTTP/2在往返时延（RTT）上仍存在问题，尤其是在移动网络方面。所以谷歌正在测试另一项协议，即QUIC；QUIC在TCP到UDP的网络转换上更加流畅。

更多特性可见：[一文读懂 HTTP/2 特性](https://link.zhihu.com/?target=http%3A//support.upyun.com/hc/kb/article/1048799/)

## OSCP stapling

在HTTPS通信过程时，浏览器通过CRL和OCSP来验证服务器端下发的证书链是否已经被撤销。

CA机构在CRL中列举在有效期内但是被撤销的数字证书，CA机构会维护并定期更新CRL列表，但随着时间的推移，体现出两个不足之处：

* CRL列表越来越大
* 浏览器更新不及时的时候，会造成误判

OCSP是实时证书在线验证协议，用来弥补CRL机制的上述不足之处，通过OCSP浏览器可以实时向CA机构验证证书。但OCSP也有自身的弊端：

* 要求CA机构实时全球高可用
* 客户端的访问隐私可能被泄露
* 增加浏览器的握手延时

于是，OCSP Stapling技术被开发出来，作为对OCSP协议缺陷的弥补，这个技术实现了服务器可以事先模拟浏览器对证书链进行验证，并将带有CA机构签名的OCSP验证结果响应保存到本地。等到真正的握手阶段，再将OCSP响应和证书链一起下发给浏览器，以此避免增加浏览器的握手延时。由于浏览器不需要直接向 CA 站点查询证书状态，这个功能对访问速度的提升非常明显。

<figure>![](https://pic1.zhimg.com/80/v2-15bbe255227b2812f325b52a30ed4dab_hd.jpg)</figure>

又拍云CDN SSL握手时间

如上图所示，使用OCSP Stapling可以在SSL握手阶段节约90+毫秒， 性能提升了34.08%左右。

## Session ID

HTTPS访问过程是在TCP三次握手之后进行SSL的四次握手，如果一个业务请求包含多条的加密流，反复的握手请求势必会导致较高的延迟。并且如果出于某种原因，对话中断，就需要重新握手，增加了用户访问时间。

Session ID是一种在网络通信中使用（通常在一块数据的HTTP）来识别一个会话，具有一系列相关的信息交流。SessionID属性用于唯一地标识在服务器上包含会话数据的浏览器。

SessionID值由[http://ASP.NET](https://link.zhihu.com/?target=http%3A//ASP.NET)随机生成，并存储在浏览器的不过期Cookie中。每次向[http://ASP.NET](https://link.zhihu.com/?target=http%3A//ASP.NET)应用程序发送请求时，SessionID值便被发送到Cookie中。因此，session ID可以有效的减少多次访问时的握手次数，降低访问延迟。

## SNI技术

过去，只有独立IP的虚拟主机或VPS才能享有SSL/TLS连接服务。在SSL握手的过程中，不会传递域名信息，所以服务器端通常返回的是配置中的第一个可用证书；如果要使用多个证书，就只能配置不同的SSL端口或增加IP地址，或者花重金使用一个“多域名SSL证书”或一个“通配型证书”来达到相同效果。

随着SNI技术的出现，在一个IP地址上可以为不同域名分配使用不同SSL证书的技术得以实现；这同时意味着，共享IP的虚拟主机也可实现SSL/TLS连接。

SNI技术的实现是用于改善SSL/TLS的技术，它可以在SSLv3/TLSv1中被启用。当SNI技术实现时，客户端可以在发起SSL握手请求时提交请求的Host信息，服务器接收到信息后，能够切换到正确的域，再返回相应的证书。

要使用SNI，需要客户端和服务器端同时满足条件，对于浏览器来说，大部分都支持SSLv3/TLSv1，所以都可以享受SNI带来的便利。对于服务器来说，Nginx在以SNI为支持的OpenSSL的陪同下即可实现。

以上就是实现HTTPS的5个技术难点，希望大家在进行HTTPS升级的时候，可以顺利实现。当然，大家也可以选择使用[又拍云CDN](https://link.zhihu.com/?target=https%3A//www.upyun.com/products/cdn)，又拍云已经完全攻克了上面提及的技术难点，只需要提供域名，即可轻松升级为HTTPS~

**推荐阅读：**

[免费](https://link.zhihu.com/?target=https%3A//www.upyun.com/https) [SSL](https://link.zhihu.com/?target=https%3A//www.upyun.com/https) [证书申请入口](https://link.zhihu.com/?target=https%3A//www.upyun.com/https)
