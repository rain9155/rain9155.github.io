---
title: Http网络请求浅析
date: 2018-12-31 15:33:15
tags: http
categories: 计算机网络
---


## 前言
随着互联网的发展，网络已经越来越普及了，绝大多数的网络请求都是基于HTTP协议的，因此在开发中，了解HTTP的基本原理是必要的，在TCP/IP四层体系结构中，HTTP协议位于应用层，它是应用层主要使用的协议，应用层往下一层就是运输层，HTTP在运输层采用的是TCP协议来保证可靠传输，知道这些后，接下来详细介绍一下 Http。

<!--more-->

## HTTP协议版本

我们先来简单了解一下 HTTP 协议的历史演变：

* HTTP/1.0：1996年，HTTP/1.0 版本发布，可以传输文字，图像、视频、二进制文件，它的特点是每次请求都需要建立一个单独的TCP连接，发送数据完毕，连接就关闭，如果还要请求其他资源，就必须再新建一个连接，上一次请求和下一次请求完全分离，这种非持续连接过程又叫做短连接。它的特点也是它的缺点，客户端和服务端每次请求都要建立TCP连接，而建立TCP连接和关闭TCP连接都是相对比较费时的过程，严重影响客户端和服务端的性能。
* HTTP/1.1：1997年，HTTP/1.1 版本发布，1999年广泛应用于各大浏览器网络请求中，直到现在HTTP/1.1也是使用最为广泛的HTTP协议，它进一步完善了HTTP/1.0，HTTP/1.1支持在一个TCP连接上传送多个HTTP请求和响应，即一个TCP连接可以被多个请求复用，减少了建立和关闭连接的消耗延迟，一定程度上弥补了HTTP/1.0每次请求都要创建连接的缺点，这种持续连接过程又叫做长连接，HTTP/1.1默认使用长连接。
* HTTP/2.0：2015年，HTTP/2 .0版本发布，前面的两个版本都是基于超文本的协议，HTTP 2.0把基于超文本的协议改成了基于二进制的，把HTTP请求和响应变成数据帧，这样就实现了多路复用，在一个TCP连接上可以同时“混合”发送多个HTTP的请求和响应，效率大大提高。

### 1、长连接和多路复用的区别

上面讲到HTTP/1.1的长连接和HTTP 2.0的多路复用都是复用TCP连接，它们之间有什么区别呢？如图：

{% asset_img http1.png http1 %}

可以看到，HTTP/1.1的长连接的若干个请求只能排队发送，后面的请求等待前面请求的响应返回才能获得执行机会，一旦有某请求超时，后续请求只能被阻塞，这就是人们常说的线头阻塞；同时由于浏览器自身的Max-Connection最大连接限制，导致同一个域名 (host) 下的请求连接限制，比如谷歌浏览器的Max-Connection值为6，即同一个域名下一次最多连续发送6个请求，超过限制后续请求就会被阻塞。

而HTTP/2.0的多路复用才是做到了同一个域名下的真正的并发请求，多个请求可同时在一个TCP连接上并行发送，某个请求任务超时，不会影响到其它连接的正常执行。

### 2、HTTPS是什么

HTTPS就是在HTTP的应用层与运输层之间加了一层**SSL/TLS**，如下：

{% asset_img http2.png http2 %}

HTTP协议运行在TCP之上，所有传输的内容都是明文；HTTPS协议运行在SSL/TLS之上，SSL/TLS运行在TCP之上，所有传输的内容都通过SSL/TLS层进行了加密，保证了数据安全。目前大多数网站都运行在HTTPS协议之上，把HTTP报文经过加密后才发送出去，在接收到HTTP报文后经过解密才能使用。

由于HTTP/2.0还未大规模应用，所以下面的讨论都是围绕HTTP/1.x，只有熟悉HTTP/1.x的原理，才能更好的读懂[HTTP/2.0](https://juejin.im/post/5a4dfb2ef265da43305ee2d0)。

## HTTP工作过程
HTTP是基于TCP的应用层协议，从更高层次封装了TCP的使用细节，使得网络操作更为简单，一个HTTP请求就是一个典型的C/S模式，HTTP协议首先要和服务器建立TCP连接，当建立TCP连接的三报文握手的前两次报文握手完成后，在第三次握手，客户端就把HTTP请求报文作为第三个握手报文的数据发送给服务端，服务器收到请求报文后，就把所请求的文档作为响应报文返回给客户端。

{% asset_img http3.png http3 %}

HTTP的工作特点可以总结为以下两点：

* 1、面向无连接的：即通信双方在交换HTTP报文时不需要向建立HTTP连接，但HTTP使用了面向连接的TCP作为运输层协议。
* 2、无状态的：服务器不会记得每个客户访问的状态，同一个客户访问两次服务器上的页面时，服务器响应与第一次访问相同，所以出现了**Cookie/Session机制**维护连接的状态，可自行了解。

## HTTP的请求方式

HTTP协议提供了几种请求方式，大家熟知的请求方式有7种GET、POST、DELETE、PUT、HEAD、TRACE、OPTIONS，其中最常用的是PUT（增）、DELETE(删)、POST（改）、GET（查）。下面以一张表来看看它们各自的作用。

| 请求 | 作用 |
|:-----:|:-----|
| CET | 客户端获取服务器中的某个资源，然后服务端返回对应资源给客户端，请求参数放在URL中 |
| POST | POST请求通常会用来提交HTML表单，把数据填在表单中，传给服务器，然后服务器对这些数据进行处理 |
| PUT | 与GET相反，PUT向服务器写入数据 |
| DELETE | 顾名思义，请服务器删除请求URL所指定的资源，但是服务端可以在客户端不知情下撤销此请求，请求参数放在URL中 |
| HEAD | HEAD与GET类似，但服务器在响应中只返回首部不会返回主体部分，HEAD是用来在不获取资源的情况下对资源的首部进行检查，如查看响应的状态码，看看资源是否被修改，对象是否存在 |
| TRACE | 客户端发起一个请求时，可能要穿过防火墙，代理，网关等，每一个中间点都会修改HTTP原始请求，TRACE允许请求最终发送给服务端时，看看它最终变成什么样，服务端会返回一个TRACE响应 |
| OPTIONS | 客户端请求Web服务器告知其支持的各种功能，就是询问服务器支持什么方法或者对某些特殊资源支持什么方法，这就让客户端不用访问那些实际的资源就能判定访问各种资源的最优方法 |

## HTTP的报文格式

不同的请求方式，它们的请求格式（报文格式）可能是不一样的，但是通常来说一个HTTP的请求报文都是由请求行，请求头部，空行，请求数据4个部分组成，如图。

{% asset_img http4.png http4 %}

* **请求行**
又叫起始行，就是报文的第一行，在请求报文中说明要以什么方式做什么请求，而在响应报文中粗略的说明了简略的报文执行结果。
* **请求首部**
又叫首部字段，起始行后由零个或多个首部字段，每个字段包含一个key和value，用冒号分割（如Connection：keep-Alive）。首部以一个空行结束。
* **请求数据**
又叫主体，其中可以包含任意类型的数据（如图片，视频等），而首部和起始行只能是文本形式。在请求主体中包括了要发送给Web服务器的数据，在响应主体中装载了要返回给客户端的数据。
###  1、请求报文
下面以GET、DELETE、PULL、POST举例。

#### 1.1、GET、DELETE的请求报文

对于GET和DELETE来说，它们的所有参数都是拼接在URL最后，第一个参数前通过"?"连接，然后请求参数按照"key=value"格式进行追加，每个请求参数之间通过"&"连接，如http://www.myhost.com/text/?id=1&name=rain，如果是GET请求表示获取http://www.myhost.com/text/ 下用户id为1，名为rain的文本，如果是DELETE请求表示删除该文本。
`注意：GET和DELETE的URL最长度为1024字节（1KB）`

```
GET /?id=1&name=rain HTTP/1.1
Host: www.myhost.com
Cache-Control: no-cache
```
从上面的HTTP请求格式知，第一行为请求行，表明请求方式为GET，子路径为 /?id=1&name=rain，HTTP版本为1.1。后两行为请求头部，Host为主机地址，Cache-Control为no-cache。因为GET,DELETE的请求参数都在URL中，所以请求数据为空。

#### 1.2、PULL、POST的请求报文

而对于PULL和POST来说，它们的报文格式一般是表单格式，也就是说请求参数存储在请求数据（主体）位置上：

```
POST /local/ HTTP/1.1
Host: www.myhost.com
Accept-Encoding：gzip
Content-Length: 222222
Content-Type:multipart/from-data；boundary=dRGP2cPPTxE6WRTssnh4jC7HJLcSde
Connection：Keep-Alive

--dRGP2cPPTxE6WRTssnh4jC7HJLcSde
Content-Disposition：from-data；name=“username”
Content-Type：text/plain：charset=UTF-8
Content-Transfer-Encoding: 8bit

rain
--dRGP2cPPTxE6WRTssnh4jC7HJLcSde
Content-Diaposition:from-data:name="image"
filename="/storage/emulated/0/image/1234.png"
Content-Type:application/octet-stream
Content-Transfer-Encoding:binary

图片二进制数据，在此省略...
--dRGP2cPPTxE6WRTssnh4jC7HJLcSde--
```
上述的请求含义是向http://www.myhost.com/local/这个地址发送一个POST请求，请求的数据格式为multipart/from-data，报文的boundary值为dRGP2cPPTxE6WRTssnh4jC7HJLcSde，报文有两个请求参数：一个是名为username的文本，值为rain；一个是名为image的二进制参数，值为图片的二进制数据。
请求参数是以两个横杠（--）加上boundary开始的，然后是请求参数的一些首部属性，如参数名，格式等，然后加上一个空行，最后才是参数的值，如上述的username=name。

```
--dRGP2cPPTxE6WRTssnh4jC7HJLcSde		//两横杠加boundary
Content-Disposition：from-data；name=“username”//以下三个参数时首部属性
Content-Type：text/plain：charset=UTF-8
Content-Transfer-Encoding: 8bit
						//不可省略的空行
rain						//参数值
```
POST和PULL都要遵守这种格式，每个参数以两横杠加boundary分隔，参数首部与值之间有一个空行。

>这里要注意，请求数据的最后是以两个横杆+boundary+两个横杠结束作为整个报文的结束符。如这里的- -dRGP2cPPTxE6WRTssnh4jC7HJLcSde- -

### 2、响应报文
由状态行、消息报头、响应正文组成，如下：
```
<状态行>
<响应报文首部>
<空行>
<响应报文内容>
```
可以看到与请求报文的格式类似，不同的就是第一行用状态信息代替了请求信息，格式如下：
```
HTTP-Version Status-Code Reason-Phrase CRLF
```
其中HTTP-Version代表HTTP协议版本，Status-Code代表响应状态码，Reason-Phrase代表状态码的文本描述，其中状态码的5种取值范围如下：

|取值范围|含义|
|:---:|:---|
|100~199|表示请求已被接收，继续处理|
|200~299|表示请求已被成功接收，理解，接收|
|300~399|重定向，要完成请求必须要进一步的操作|
|400~499|客户端错误，表示请求有语法错误或请求无法实现|
|500~599|服务端错误，表示服务器未能实现合法的请求|

例如这是一个GET请求的返回的响应报文格式：
```
HTTP1.1 200 OK
Data:Sat, 30, Dec 2006 23:23:00 GMT
Content-Type:text/plain；charset=UTF-8
Content-Length:14

{data:这里是返回结果}
```
上面表示，响应状态码为200，表示请求成功，返回数据类型为text/plain，编码为UTF-8，内容长度为7，空行之后就为返回的数据。
#### 2.1、常见的状态码
* 200 OK：客户端请求成功
* 400 Bad Request：客户端请求有语法错误，不能被服务器理解
* 401 Unauthorized：请求未授权
* 403 Forbidden：服务器收到请求，但是拒绝提供服务
* 404 Not Found：请求资源不存在
* 500 Internal Server Error：服务器发生不可预估的错误
* 503 Server Unavailable：服务器当前不能处理客户端请求，一段时间后可能恢复正常

### 3、常见的头部

- Content-Type：请求数据的格式
- Content-Length：消息长度
- Host：请求的主机名
- User-Agent：发出请求的浏览器类型，可以自行设置
- Accept：客户端可识别的内容类型列表
- Accept-Endcoding：客户端可识别的数据编码
- Connection：允许客户端和服务端指定与请求/响应连接相关的选项，如设置为Connection：Keep-Alive，表示保持连接
- Transfer-Encoding：告知接收端为了保证报文的可靠传输性，对报文采用了什么的编码方式

#### 3.1、Connection ：Keep-Alive工作原理 (长连接)

从HTTP/1.1起，默认都开启了Keep-Alive保持连接特性，简单地说，当一个网页打开完成后，客户端和服务端之间用于传输HTTP数据的TCP连接不会关闭，如果客户端再次访问这个服务端上的网页，会继续使用这一条已经建立的TCP连接，Keep-Alive不会永久保持连接，它有一个保持时间，可以在不同服务器软件中设置这个时间。

那么，长连接是如何工作的呢？长短连接是运输层（TCP）的概念，HTTP是应用层协议，它只能说告诉运输层我打算一段时间内复用TCP通道，而没有自己去建立、释放TCP通道的能力。那么HTTP是如何告诉运输层复用TCP通道的呢？分为以下几个步骤：

- 客户端第一次发送请求报文时，顺带发送一个Connection: keep-alive的Header，表示需要保持连接，同时客户端可以顺带发送Keep-Alive: timeout=5,max=100这个Header给服务端
- 然后服务端识别Connection: keep-alive这个Header，并且通过响应报文Header带同样的Connection: keep-alive，告诉客户端我可以保持连接
- 客户端和服务端之间通过保持的通道收发数据
- 当客户端最后一次发送请求报文，顺带发送Connection：close这个Header，表示长连接关闭

Keep-Alive: timeout=5,max=100表示TCP连接最多保持5秒，长连接接受100次请求就断开，长连接虽好，但是长时间的TCP连接容易导致系统资源无效占用，浪费系统资源，所以需要有一些限制。

## 结语
网络请求已经成为了一个应用最基本的部分，所以熟悉HTTP对于我们开发很重要，我们不仅会用开发环境提供给我们的API，还要属性它的原理，这样开发才能做到胸有成竹。

参考资料：

[Web的工作方式](https://www.jianshu.com/p/84aa55a8a7eb)

[HTTP1.0、HTTP1.1 和 HTTP2.0 的区别](https://www.cnblogs.com/heluan/p/8620312.html)

