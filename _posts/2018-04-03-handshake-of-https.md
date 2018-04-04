---
layout:       post
title:        "由 Https 握手引发的一系列..."
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


> 请勿转载...

> 目的: 了解 Https 握手的过程...

**目录:**

* content
{:toc}

## 背景知识

### OSI 和 TCP/IP

- **OSI** 是 **Open System Interconnection**的缩写，意为**开放式系统互联**。国际标准化组织（ISO）制定了OSI模型，该模型定义了不同计算机互联的标准，是设计和描述计算机网络通信的基本框架。OSI模型把网络通信的工作分为7层，分别是物理层、数据链路层、网络层、传输层、会话层、表示层和应用层。

![]({{ "/img/in-post/2018-04-03-handshake-of-https/705728-20160424234824085-667046040.png" | prepend: site.baseurl }})

        各层功能定义
                这里我们只对OSI各层进行功能上的大概阐述，不详细深究，因为每一层实际都是一个复杂的层。后面我也会根据个人方向展开部分层的深入学习。这里我们就大概了解一下。我们从最顶层——应用层 开始介绍。整个过程以公司A和公司B的一次商业报价单发送为例子进行讲解。
        <1>    应用层
                OSI参考模型中最靠近用户的一层，是为计算机用户提供应用接口，也为用户直接提供各种网络服务。我们常见应用层的网络服务协议有：HTTP，HTTPS，FTP，POP3、SMTP等。
                实际公司A的老板就是我们所述的用户，而他要发送的商业报价单，就是应用层提供的一种网络服务，当然，老板也可以选择其他服务，比如说，发一份商业合同，发一份询价单，等等。
        <2>    表示层
                表示层提供各种用于应用层数据的编码和转换功能,确保一个系统的应用层发送的数据能被另一个系统的应用层识别。如果必要，该层可提供一种标准表示形式，用于将计算机内部的多种数据格式转换成通信中采用的标准表示形式。数据压缩和加密也是表示层可提供的转换功能之一。
                由于公司A和公司B是不同国家的公司，他们之间的商定统一用英语作为交流的语言，所以此时表示层（公司的文秘），就是将应用层的传递信息转翻译成英语。同时为了防止别的公司看到，公司A的人也会对这份报价单做一些加密的处理。这就是表示的作用，将应用层的数据转换翻译等。
        <3>    会话层
                会话层就是负责建立、管理和终止表示层实体之间的通信会话。该层的通信由不同设备中的应用程序之间的服务请求和响应组成。      
                会话层的同事拿到表示层的同事转换后资料，（会话层的同事类似公司的外联部），会话层的同事那里可能会掌握本公司与其他好多公司的联系方式，这里公司就是实际传递过程中的实体。他们要管理本公司与外界好多公司的联系会话。当接收到表示层的数据后，会话层将会建立并记录本次会话，他首先要找到公司B的地址信息，然后将整份资料放进信封，并写上地址和联系方式。准备将资料寄出。等到确定公司B接收到此份报价单后，此次会话就算结束了，外联部的同事就会终止此次会话。
        <4>   传输层
                传输层建立了主机端到端的链接，传输层的作用是为上层协议提供端到端的可靠和透明的数据传输服务，包括处理差错控制和流量控制等问题。该层向高层屏蔽了下层数据通信的细节，使高层用户看到的只是在两个传输实体间的一条主机到主机的、可由用户控制和设定的、可靠的数据通路。我们通常说的，TCP UDP就是在这一层。端口号既是这里的“端”。
                传输层就相当于公司中的负责快递邮件收发的人，公司自己的投递员，他们负责将上一层的要寄出的资料投递到快递公司或邮局。
        <5>   网络层
               本层通过IP寻址来建立两个节点之间的连接，为源端的运输层送来的分组，选择合适的路由和交换节点，正确无误地按照地址传送给目的端的运输层。就是通常说的IP层。这一层就是我们经常说的IP协议层。IP协议是Internet的基础。
                网络层就相当于快递公司庞大的快递网络，全国不同的集散中心，比如说，从深圳发往北京的顺丰快递（陆运为例啊，空运好像直接就飞到北京了），首先要到顺丰的深圳集散中心，从深圳集散中心再送到武汉集散中心，从武汉集散中心再寄到北京顺义集散中心。这个每个集散中心，就相当于网络中的一个IP节点。
        <6>   数据链路层 
                将比特组合成字节,再将字节组合成帧,使用链路层地址 (以太网使用MAC地址)来访问介质,并进行差错检测。
             数据链路层又分为2个子层：逻辑链路控制子层（LLC）和媒体访问控制子层（MAC）。

                MAC子层处理CSMA/CD算法、数据出错校验、成帧等；LLC子层定义了一些字段使上次协议能共享数据链路层。 在实际使用中，LLC子层并非必需的。

                这个没找到合适的例子

        <7>  物理层     
                实际最终信号的传输是通过物理层实现的。通过物理介质传输比特流。规定了电平、速度和电缆针脚。常用设备有（各种物理设备）集线器、中继器、调制解调器、网线、双绞线、同轴电缆。这些都是物理层的传输介质。
                 快递寄送过程中的交通工具，就相当于我们的物理层，例如汽车，火车，飞机，船。

- **TCP/IP** 是 **Transmission Control Protocol/Internet Protocol** 的简写，中译名为**传输控制协议/因特网互联协议**，又名**网络通讯协议**，是Internet最基本的协议、Internet国际互联网络的基础，由网络层的IP协议和传输层的TCP协议组成。TCP/IP 定义了电子设备如何连入因特网，以及数据如何在它们之间传输的标准。协议采用了4层的层级结构，每一层都呼叫它的下一层所提供的协议来完成自己的需求。通俗而言：TCP负责发现传输的问题，一有问题就发出信号，要求重新传输，直到所有数据安全正确地传输到目的地。而IP是给因特网的每一台联网设备规定一个地址。

![]({{ "/img/in-post/2018-04-03-handshake-of-https/705728-20160424234825491-384470376.png" | prepend: site.baseurl }})

### **OSI** VS **TCP/IP**

**TCP/IP**模型实际上是 **OSI** 模型的一个浓缩版本，它只有四个层次：

1. 应用层，对应着OSI的应用层、表示层、会话层
2. 传输层，对应着OSI的传输层
3. 网络层，对应着OSI的网络层
4. 网络接口层，对应着OSI的数据链路层和物理层

OSI模型的网络层同时支持面向连接和无连接的通信，但是传输层只支持面向连接的通信；

TCP/IP模型的网络层只提供无连接的服务，但是传输层上同时提供两种通信模式。


### 协议层的一些实现举例

每层协议都有一些具体实现, 直接看下面的表格和图片.

OSI中的层 | 功能                                                                   | TCP/IP协议族                            
------ | -------------------------------------------------------------------- | -------------------------------------
应用层    | 文件传输，电子邮件，文件服务，虚拟终端                                                  | TFTP，HTTP，SNMP，FTP，SMTP，DNS，Telnet 等等
表示层    | 数据格式化，代码转换，数据加密                                                      | 没有协议                                 
会话层    | 解除或建立与别的接点的联系                                                        | 没有协议                                 
传输层    | 提供[端对端](https://baike.baidu.com/item/%E7%AB%AF%E5%AF%B9%E7%AB%AF)的接口 | TCP，UDP                              
网络层    | 为[数据包](https://baike.baidu.com/item/%E6%95%B0%E6%8D%AE%E5%8C%85)选择路由 | IP，ICMP，OSPF，EIGRP，IGMP              
数据链路层  | 传输有地址的帧以及错误检测功能                                                      | SLIP，CSLIP，PPP，MTU                   
物理层    | 以二进制数据形式在物理媒体上传输数据                                                   | ISO2110，IEEE802，IEEE802.2            


![]({{ "/img/in-post/2018-04-03-handshake-of-https/705728-20160424234827195-1493107425.png" | prepend: site.baseurl }})

![]({{ "/img/in-post/2018-04-03-handshake-of-https/555379-20160210231238370-230545857.png" | prepend: site.baseurl }})

### 支撑起协议层的一些硬件设备

在每一层都工作着不同的设备，比如我们常用的交换机就工作在数据链路层的，一般的路由器是工作在网络层的。

![]({{ "/img/in-post/2018-04-03-handshake-of-https/705728-20160424234826351-1957282396.png" | prepend: site.baseurl }})


### Http 协议

**HTTP**是一种 **应用层协议**(看上面的表格).

HTTP（HyperText Transfer Protocol)超文本传输协议是互联网上应用最为广泛的一种网络协议。
由于信息是明文传输，所以被认为是不安全的。

用来干什么? 看 **OSI** 截图(最顶的三层)


### TCP 协议

**TCP** 是一种 **传输层协议**, 支持面向连接的通信协议

### UDP 协议

**UDP** 是一种 **传输层协议**, 支持面向无连接的通信协议

### SSL/TLS 协议

**SSL/TLS** 是一种 **传输层协议**.

SSL(Secure Sockets Layer 安全套接层),及其继任者传输层安全（Transport Layer Security，TLS）是为网络通信提供安全及数据完整性的一种安全协议。TLS与SSL在传输层对网络连接进行加密。

#### SSL/TLS的历史

互联网加密通信协议的历史，几乎与互联网一样长。

        1994年，NetScape公司设计了SSL协议（Secure Sockets Layer）的1.0版，但是未发布。
        1995年，NetScape公司发布SSL 2.0版，很快发现有严重漏洞。
        1996年，SSL 3.0版问世，得到大规模应用。
        1999年，互联网标准化组织ISOC接替NetScape公司，发布了SSL的升级版TLS 1.0版。
        2006年和2008年，TLS进行了两次升级，分别为TLS 1.1版和TLS 1.2版。最新的变动是2011年TLS 1.2的修订版。
        目前，应用最广泛的是TLS 1.0，接下来是SSL 3.0。但是，主流浏览器都已经实现了TLS 1.2的支持。

TLS 1.0通常被标示为SSL 3.1，TLS 1.1为SSL 3.2，TLS 1.2为SSL 3.3。


#### 代码举例:

Android 的 Okhttp 是一个支持 https 的客户端实现. 
另外下面的代码是极其不严谨的(因为验证逻辑一点都没写), 请勿直接copy用于工作环境.

```java
    /**
     * 创建使用 SSL 协议的 http client.
     */
    public static OkHttpClient cloneOkHttpsClient() {
        // 创建使用 SSL 协议的 http client.
        SSLContext sslContext = null;
        try {
            sslContext = SSLContext.getInstance("TLS"); // <------ here
            X509TrustManager tm = new X509TrustManager() {
                public void checkClientTrusted(X509Certificate[] chain, String authType) throws CertificateException {
                }

                public void checkServerTrusted(X509Certificate[] chain, String authType) throws CertificateException {
                }

                public X509Certificate[] getAcceptedIssuers() {
                    return new X509Certificate[0];
                }
            };
            sslContext.init(null, new TrustManager[]{tm}, null);
        } catch (Exception e) {
            Logger.e(LOG_TAG, "#cloneOkHttpsClient : ", e);
        }

        HostnameVerifier hostnameVerifier = new HostnameVerifier() {
            @Override
            public boolean verify(String hostname, SSLSession session) {
                return true;
            }
        };

        OkHttpClient clone = getOkHttpClient().newBuilder()
                .sslSocketFactory(sslContext.getSocketFactory())
                .hostnameVerifier(hostnameVerifier)
                .build();
        return clone;
    }
```

- https://baike.baidu.com/item/ssl/320778
- https://baike.baidu.com/item/TLS/2979545?fr=aladdin
- http://www.cnblogs.com/zhuqil/archive/2012/10/06/ssl_detail.html
- http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html

### IP 协议

**IP** 是一种 **网络层协议**

### 常挂在嘴边的 Http 协议是? 

    网络层(用IP协议) + 传输层(用TCP协议) + 应用层(用 Http 协议) 的一种组合结构...(真相总是残酷的 ε(┬┬﹏┬┬)3 )
    
    
### 常挂在嘴边的 Https 协议是? 

    网络层(用IP协议) + 传输层(用SSL/TLS协议) + 应用层(用 Http 协议) 的一种组合结构...(真相总是残酷的 ε(┬┬﹏┬┬)3 )
    
    
### 嘴边的 Https VS http

常说 Https 比 Http 安全, 是因为 Https 传输层所用的协议是 **SSL/TLS**, 数据是被加密的.


### CA证书是什么？

CA（Certificate Authority）是负责管理和签发证书的第三方权威机构，是所有行业和公众都信任的、认可的。

CA证书，就是CA颁发的证书，可用于验证网站是否可信（针对HTTPS）、验证某文件是否可信（是否被篡改）等，也可以用一个证书来证明另一个证书是真实可信，最顶级的证书称为根证书。除了根证书（自己证明自己是可靠），其它证书都要依靠上一级的证书，来证明自己。

> 补充: CA 是要收取服务费用的, 因为它会提供**证书认证服务器**来供客户端验证证书的合法性.


## HTTP三次握手

HTTP（HyperText Transfer Protocol)超文本传输协议是互联网上应用最为广泛的一种网络协议。由于信息是明文传输，所以被认为是不安全的。而关于HTTP的三次握手，其实就是使用三次TCP握手确认建立一个HTTP连接。

如下图所示，SYN（synchronous）是TCP/IP建立连接时使用的握手信号、Sequence number（序列号）、Acknowledge number（确认号码），三个箭头指向就代表三次握手，完成三次握手，客户端与服务器开始传送数据。

![]({{ "/img/in-post/2018-04-03-handshake-of-https/555379-20160210231251448-1547962527.jpg" | prepend: site.baseurl }})

第一次握手：客户端发送syn包(syn=j)到服务器，并进入SYN_SEND状态，等待服务器确认；

第二次握手：服务器收到syn包，必须确认客户的SYN（ack=j+1），同时自己也发送一个SYN包（syn=k），即SYN+ACK包，此时服务器进入SYN_RECV状态；

第三次握手：客户端收到服务器的SYN＋ACK包，向服务器发送确认包ACK(ack=k+1)，此包发送完毕，客户端和服务器进入ESTABLISHED状态，完成三次握手。

## HTTPS握手

摘抄至 http://www.cnblogs.com/zhuqil/archive/2012/07/23/2604572.html

HTTPS其实是有两部分组成：HTTP + SSL / TLS，也就是在HTTP上又加了一层处理加密信息的模块。服务端和客户端的信息传输都会通过TLS进行加密，所以传输的数据都是加密后的数据。具体是如何进行加密，解密，验证的，且看下图。

![]({{ "/img/in-post/2018-04-03-handshake-of-https/2012072310244445.png" | prepend: site.baseurl }})

- 1 客户端发起HTTPS请求

这个没什么好说的，就是用户在浏览器里输入一个https网址，然后连接到server的443端口。

> 补充:
>
>把客户端所支持的 SSL(TLS) 的最高版本也发给服务器

- 2 服务端的配置

采用HTTPS协议的服务器必须要有一套数字证书，可以自己制作，也可以向组织申请。区别就是自己颁发的证书需要客户端验证通过，才可以继续访问，而使用受信任的公司申请的证书则不会弹出提示页面(startssl就是个不错的选择，有1年的免费服务)。这套证书其实就是一对公钥和私钥。如果对公钥和私钥不太理解，可以想象成一把钥匙和一个锁头，只是全世界只有你一个人有这把钥匙，你可以把锁头给别人，别人可以用这个锁把重要的东西锁起来，然后发给你，因为只有你一个人有这把钥匙，所以只有你才能看到被这把锁锁起来的东西。

- 3 传送证书

这个证书其实就是公钥，只是包含了很多信息，如证书的颁发机构，过期时间等等。

- 4 客户端解析证书

这部分工作是有客户端的TLS来完成的，首先会验证公钥是否有效，比如颁发机构，过期时间等等，如果发现异常，则会弹出一个警告框，提示证书存在问题。如果证书没有问题，那么就生成一个随机值(Password)。然后用证书对该随机值进行加密。就好像上面说的，把随机值用锁头锁起来，这样除非有钥匙，不然看不到被锁住的内容。

> 补充: 
>
> 如果证书是由CA签发的话, 客户端还要去CA证书服务器验证证书的合法性.
>
> 如果证书是自己制作的话, 客户端需要导入跟服务器一模一样的证书, 来跟服务器传输过来的证书做对比验证.

- 5 传送加密信息

这部分传送的是用证书加密后的随机值，目的就是让服务端得到这个随机值，以后客户端和服务端的通信就可以通过这个随机值来进行加密解密了。

- 6 服务段解密信息

服务端用私钥解密后，得到了客户端传过来的随机值(Password)，然后把内容通过该值进行对称加密。所谓对称加密就是，将信息和随机值(Password)通过某种算法混合在一起，这样除非知道随机值(Password)，不然无法获取内容，而正好客户端和服务端都知道这个随机值(Password)，所以只要加密算法够彪悍，私钥够复杂，数据就够安全。

- 7 传输加密后的信息

这部分信息是服务段用随机值(Password)加密后的信息，可以在客户端被还原

- 8 客户端解密信息

客户端用之前生成的随机值(Password)解密服务段传过来的信息，于是获取了解密后的内容。整个过程第三方即使监听到了数据，也束手无策。


## 参考

https://www.cnblogs.com/qishui/p/5428938.html
