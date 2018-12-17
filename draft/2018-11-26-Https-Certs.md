---
layout:     post
title:      Https-TLS/SSL机制详解
subtitle:    ""
date:       2018-12-15
author:     guoqing
header-img: img/posts/post-bg-hacker.jpg
catalog: true
tags:
    - Https
    - NetWork
---
### 目录
> HTTPS  
> TLS/SSL  
> TLS/SSL机制  
### HTTPS  

HTTPS协议是HTTP协议的安全版，s代表着secure即所有使用HTTPS协议通信的服务都是加密的。由于HTTP协议是明文的，因此client与server之间传输的数据容易被修改。为了解决这一问题，HTTPS协议在TCP网络模型中传输层与应用层之间加了一层TLS/SSL为数据通信提供安全支持。HTTPS的机制就是指TSL/SSL的机制。
### TSL/SSL  
TLS(Transport layer secure)是基于SSL(Secure Socket Layer)发展而来。他们都提供了数据的加密及认证，因此使用这个协议可以确保：
- 任何人都无法读取你得数据
- 任何人都无法修改你得数据
- 你可以确定你所通信的服务是可信的  
而做到这两点就需要加密和签名，TLS/SSL采用非对称式加密public key和private key来加密数据。其中public key是公有的任何人都可以得到，因此你无法确定这个公有的key是不是真的。为了确定这个公有的key就是你想要的，因此采用数字证书。证书就相当于身份证，在public key和一个受信任的第三方（CA）建立一个连接即签名。得到了这个数字证书既证明了自己的身份，自己的public key就在证书里面。

### TLS/SSL机制

TLS/SSL机制分为两个方面：  
- 握手阶段  
   双方协商生成密匙
- 通信阶段
   双方通过密匙来加密通信   

主要机制即握手阶段，握手阶段分为四部分，本文将通过wireshark抓包来来展示具体请求。如图所示的整个握手的时序图：
![ssl](/img/posts/TLS-SSL.png)
1. Client Hello(客户端向server发出请求)，如下图所示：
![clienthello](/img/posts/clienthello.png)  

消息中包含了客户端所支持的TLS的版本号、加密算法、随机数(用于生成密匙)、压缩方法。
2. Server Hello
服务端收到客户端的消息后，向客户端发出回应，内容包含：  
  1）确认服务器TLS通信版本    
  2）服务器端的随机数  
  3）确认使用的加密算法  
  ![serverhello](/img/posts/serverhello.png)
  在确认客户端的TLS通信版本、加密算法等后，服务端就可以向客户端发送证书了，如下图：
  ![certificate](/img/posts/certificate.png)  
  如果客户端和服务端所采用的加密算法比较特殊如DH，server端可以在向客户端发送一个server-key-exchange请求，又或者server端需要验证客户端的身份又会在发送一个certificate-request消息。最后服务端会发送server-done消息表明server端请求结束，如下图：
  ![serverdone](/img/posts/serverdone.png)


 3. 客户端回应
