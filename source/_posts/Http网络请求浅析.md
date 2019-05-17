---
title: Http网络请求浅析
date: 2018-12-31 15:33:15
tags: http
categories: android
---


## 前言
随着互联网的发展，网络已经越来越普及了，对于Android开发来说，绝大多数的应用都需要网络的请求，而HTTP在网络开发是使用频率最高，也是最为重要的手段，因此了解HTTP的基本原理是必要的。
<!--more-->
## HTTP网络请求原理和过程
HTTP网络请求原理和过程我推荐在我简书上看过的一篇文章[Web的工作方式](https://www.jianshu.com/p/84aa55a8a7eb)，里面写的很详细。
从上面的文章我们可以知道，HTTP是基于TCP的应用层协议，从更高层次封装了TCP的使用细节，使得网络操作更为简单，一个HTTP请求就是一个典型的C/S模式，服务端监听某个端口，客户端向服务端发起请求，服务端解析请求并向客户端返回结果。而TCP是通过IP分组（数据报）来发送数据。
HTTP要传送一条报文时，会以流的形式将报文数据通过一条打开的TCP连接按序传输，TCP收到数据流后，会将数据流分割成称为段的小数据块，并将段封装在IP分组中，通过因特网进行传输。
<br> {% asset_img HTTP层次.jpg HTTP层次%} <br>
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
<br>{% asset_img HTTP报文格式.jpg HTTP报文格式%}<br>
* **请求行**
又叫起始行，就是报文的第一行，在请求报文中说明要以什么方式做什么请求，而在响应报文中粗略的说明了简略的报文执行结果。
* **请求首部**
又叫首部字段，起始行后由零个或多个首部字段，每个字段包含一个key和value，用冒号分割（如Connection：keep-Alive）。首部以一个空行结束。
* **请求数据**
又叫主体，其中可以包含任意类型的数据（如图片，视频等），而首部和起始行只能是文本形式。在请求主体中包括了要发送给Web服务器的数据，在响应主体中装载了要返回给客户端的数据。
###  #请求报文
下面以GET、DELETE、PULL、POST举例。对于GET和DELETE来说，它们的所有参数都是拼接在URL最后，第一个参数前通过"?"连接，然后请求参数按照"key=value"格式进行追加，每个请求参数之间通过"&"连接，如http://www.myhost.com/text/?id=1&name=rain，如果是GET请求表示获取http://www.myhost.com/text/ 下用户id为1，名为rain的文本，如果是DELETE请求表示删除该文本。
`注意：GET和DELETE的URL最长度为1024字节（1KB）`

```
GET /?id=1&name=rain HTTP/1.1
Host: www.myhost.com
Cache-Control: no-cache
```
从上面的HTTP请求格式知，第一行为请求行，表明请求方式为GET，子路径为 /?id=1&name=rain，HTTP版本为1.1。后两行为请求头部，Host为主机地址，Cache-Control为no-cache。因为GET,DELETE的请求参数都在URL中，所以请求数据为空。
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
上述的请求含义是向http://www.myhost.com/local/这个地址发送一个POST请求，请求的数据格式为multipart/from-data，报文的boundary值为dRGP2cPPTxE6WRTssnh4jC7HJLcSde，报文有两个参数，一个是文本类型的username，值为rain，一个是名为image的二进制参数，值为图片的二进制数据。
一个参数是以两个横杠（--）加上boundary开始的，然后是请求参数的一些首部属性，如参数名，格式等，然后加上一个空行，最后才是参数的值，如上述的username=name。
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

### #响应报文
由状态行、消息报头、响应正文组成，如下：
```
<状态行>
<响应报文首部>
<空行>
<响应报文内容>
```
可以看到与请求的格式类似，不同的就是第一行用状态信息代替了请求信息，格式如下：
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
#### ##常见的状态码
* 200 OK：客户端请求成功
* 400 Bad Request：客户端请求有语法错误，不能被服务器理解
* 401 Unauthorized：请求未授权
* 403 Forbidden：服务器收到请求，但是拒绝提供服务
* 404 Not Found：请求资源不存在
* 500 Internal Server Error：服务器发生不可预估的错误
* 503 Server Unavailable：服务器当前不能处理客户端请求，一段时间后可能恢复正常

#### ##常见的请求头部
* Content-Type：请求数据的格式
* Content-Length：消息长度
* Host：请求的主机名
* User-Agent：发出请求的浏览器类型，可以自行设置
* Accept：客户端可识别的内容类型列表
* Accept-Endcoding：客户端可识别的数据编码
* Connection：允许客户端和服务端指定与请求/响应连接相关的选项，如设置为Connection：Keep-Alive，表示保持连接
* Transfer-Encoding：告知接收端为了保证报文的可靠传输性，对报文采用了什么的编码方式

## 结语
网络请求已经成为了一个应用最基本的部分，所以熟悉HTTP请求对于我们开发很重要，我们不仅会用开发环境提供给我们的API，还要属性它的原理，这样开发才能做到胸有成竹，而且也是我的一篇学习笔记，供以后参考。

