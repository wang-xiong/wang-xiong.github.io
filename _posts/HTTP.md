# 一、网络结构

网络结构有两种主流的分层方法：OSI七层模型和TCP/IP四层模型

OSI是指Open System InterConnect，意为开放式系统互联

TCP/IP是指传输控制协议/网间协议，是目前世界上应用最广的协议

![image-20190126112410948](/Users/wangxiong/Library/Application Support/typora-user-images/image-20190126112410948.png)



HTTP是基于TCP/IP协议的应用层协议，不包括数据包的传输，主要规定了客户端和服务端的通信格式，默认使用80端口。

# 二、HTTP的发展史

1991年发布Http/0.9版本，只有Get命令，且服务端直返HTML格式字符串，服务器响应完毕就关闭TCP连接。

1996年发布Http/1.0版本，优点：可以发送任何格式内容，包括文字、图像、视频、二进制。也丰富了命令Get，Post，Head。请求和响应的格式加入头信息。缺点：每个TCP连接只能发送一个请求，而新建TCP连接的成本很高，导致Http/1.0新能很差。

1997发布Http/1.1版本，完善了Http协议，直至20年后的今天仍是最流行的版本。
优点：

a. 引入持久连接，TCP默认不关闭，可被多个请求复用，对于一个域名，多数浏览器允许同时建立6个持久连接。

b. 引入管道机制，即在同一个TCP连接中，可以同时发送多个请求，不过服务器还是按顺序响应。

c. 在头部加入Content-Length字段，一个TCP可以同时传送多个响应，所以就需要该字段来区分哪些内容属于哪个响应。

d. 分块传输编码，对于耗时的动态操作，用流模式取代缓存模式，即产生一块数据，就发送一块数据。

e. 增加了许多命令，头信息增加Host来指定服务器域名，可以访问一台服务器上的不同网站。
缺点：TCP连接中的响应有顺序，服务器处理完一个回应才能处理下一个回应，如果某个回应特别慢，后面的请求就会排队等着（对头堵塞）

2015年发布Http2.0版本，它有几个特性：二进制协议、多工、数据流、头信息压缩、服务器推送。

**Http的基本优化**

影响http网络请求的因素主要有两个：宽度和延迟

宽度：目前已经满足了。

延迟：

请求阻塞：同一个域名同时连接的连接数，超过就会阻塞。

DNS查询：通过DNS系统解析域名为IP，通常利用DNS缓存来达到减少这个时间的目的。

建立连接：http是基于tcp协议的，需要经过三次握手建立连接，通过连接复用来减少这块的时间。

## 1.Http1.0和Http1.1的区别：

1. **缓存处理**，在HTTP1.0中主要使用header里的If-Modified-Since,Expires来做为缓存判断的标准，HTTP1.1则引入了更多的缓存控制策略例如Entity tag，If-Unmodified-Since, If-Match, If-None-Match等更多可供选择的缓存头来控制缓存策略。
2. **带宽优化及网络连接的使用**，HTTP1.0中，存在一些浪费带宽的现象，例如客户端只是需要某个对象的一部分，而服务器却将整个对象送过来了，并且不支持断点续传功能，HTTP1.1则在请求头引入了range头域，它允许只请求资源的某个部分，即返回码是206（Partial Content），这样就方便了开发者自由的选择以便于充分利用带宽和连接。
3. **错误通知的管理**，在HTTP1.1中新增了24个错误状态响应码，如409（Conflict）表示请求的资源与资源的当前状态发生冲突；410（Gone）表示服务器上的某个资源被永久性的删除。
4. **Host头处理**，在HTTP1.0中认为每台服务器都绑定一个唯一的IP地址，因此，请求消息中的URL并没有传递主机名（hostname）。但随着虚拟主机技术的发展，在一台物理服务器上可以存在多个虚拟主机（Multi-homed Web Servers），并且它们共享一个IP地址。HTTP1.1的请求消息和响应消息都应支持Host头域，且请求消息中如果没有Host头域会报告一个错误（400 Bad Request）。
5. **长连接**，HTTP 1.1支持长连接（PersistentConnection）和请求的流水线（Pipelining）处理，在一个TCP连接上可以传送多个HTTP请求和响应，减少了建立和关闭连接的消耗和延迟，在HTTP1.1中默认开启Connection： keep-alive，一定程度上弥补了HTTP1.0每次请求都要创建连接的缺点。

## 2.Https与http的区别

1. ## https协议需要到CA申请证书，大多数情况下需要一定费用

2. 断开变化，http默认端口是80，https默认端口是443

3. https可以有效的防止运行商劫持，解决防劫持的一个大问题.

4. Http是超文本传输协议，信息采用明文传输，Https则是具有安全性SSL加密传输协议

5. Http和Https端口号不一样，Http是80端口，Https是443端口

6. Http连接是无状态的，而Https采用Http+SSL构建可进行加密传输、身份认证的网络协议，更安全。

7. Http协议建立连接的过程比Https协议快。因为Https除了Tcp三次握手，还要经过SSL握手。连接建立之后数据传输速度，二者无明显区别。

## 3.SPDY:Http1.x的优化

2012年google提出了SPDY的方案，优化了Http1.x的请求延迟，解决了Http1.x的安全性。

1. **降低延迟**，针对HTTP高延迟的问题，SPDY优雅的采取了多路复用（multiplexing）。多路复用通过多个请求stream共享一个tcp连接的方式，解决了HOL blocking的问题，降低了延迟同时提高了带宽的利用率。
2. **请求优先级**（request prioritization）。多路复用带来一个新的问题是，在连接共享的基础之上有可能会导致关键请求被阻塞。SPDY允许给每个request设置优先级，这样重要的请求就会优先得到响应。比如浏览器加载首页，首页的html内容应该优先展示，之后才是各种静态资源文件，脚本文件等加载，这样可以保证用户能第一时间看到网页内容。
3. **header压缩。**前面提到HTTP1.x的header很多时候都是重复多余的。选择合适的压缩算法可以减小包的大小和数量。
4. **基于HTTPS的加密协议传输**，大大提高了传输数据的可靠性。
5. **服务端推送**（server push），采用了SPDY的网页，例如我的网页有一个sytle.css的请求，在客户端收到sytle.css数据的同时，服务端会将sytle.js的文件推送给客户端，当客户端再次尝试获取sytle.js时就可以直接从缓存中获取到，不用再发请求了。SPDY构成图：

## 4.HTTP2.0

HTTP2.0可以说是SPDY的升级版（其实原本也是基于SPDY设计的），但是，HTTP2.0 跟 SPDY 仍有不同的地方，如下：

1. HTTP2.0 支持明文 HTTP 传输，而 SPDY 强制使用 HTTPS
2. HTTP2.0 消息头的压缩算法采用 **HPACK** http://http2.github.io/http2-spec/compression.html，而非 SPDY 采用的 **DEFLATE** http://zh.wikipedia.org/wiki/DEFLATE

### HTTP2.0和Http1.x相比的新特性

- **新的二进制格式**（Binary Format），HTTP1.x的解析是基于文本。基于文本协议的格式解析存在天然缺陷，文本的表现形式有多样性，要做到健壮性考虑的场景必然很多，二进制则不同，只认0和1的组合。基于这种考虑HTTP2.0的协议解析决定采用二进制格式，实现方便且健壮。
- **多路复用**（MultiPlexing），即连接共享，即每一个request都是是用作连接共享机制的。一个request对应一个id，这样一个连接上可以有多个request，每个连接的request可以随机的混杂在一起，接收方可以根据request的 id将request再归属到各自不同的服务端请求里面。
- **header压缩**，如上文中所言，对前面提到过HTTP1.x的header带有大量信息，而且每次都要重复发送，HTTP2.0使用encoder来减少需要传输的header大小，通讯双方各自cache一份header fields表，既避免了重复header的传输，又减小了需要传输的大小。
- **服务端推送**（server push），同SPDY一样，HTTP2.0也具有server push功能。

## 5.Socket和Http的区别

HTTP协议：对应于应用层，基于TCP连接的

tcp协议：对应于传输层

ip协议：对应于网络层

TCP/IP是传输层协议，主要解决数据如何在网络中传输，而Http是应用层协议，主要解决如何包装数据。

Socket是对TCP/IP协议的封装，Socket本身并不是协议，而是一个调用接口（API），通过Socket，我们才能使用TCP/IP协议，输入传输层。

## 6.HTTP请求格式

Request格式

GET /barite/account/stock/groups HTTP/1.1
QUARTZ-SESSION: MC4xMDQ0NjA3NTI0Mzc0MjAyNg.VPXuA8rxTghcZlRCfiAwZlAIdCA
DEVICE-TYPE: ANDROID
API-VERSION: 15
Host: shitouji.bluestonehk.com
Connection: Keep-Alive
Accept-Encoding: gzip
User-Agent: okhttp/3.10.0

Response格式：

```
HTTP/1.1 200 OK
Server: nginx/1.6.3
Date: Mon, 15 Oct 2018 03:30:28 GMT
Content-Type: application/json;charset=UTF-8
Pragma: no-cache
Cache-Control: no-cache
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Content-Encoding: gzip
Transfer-Encoding: chunked
Proxy-Connection: Keep-alive

{"errno":0,"dialogInfo":null,"body":{"list":[{"flag":2,"group_id":1557,"group_name":"港股","count":1},{"flag":3,"group_id":1558,"group_name":"美股","count":7},{"flag":1,"group_id":1556,"group_name":"全部","count":8}]},"message":"success"}
```

- Host：指定服务器域名，可用来区分访问一个服务器上的不同服务
- Connection：keep-alive表示要求服务器不要关闭TCP连接，close表示明确要求关闭连接，默认值是keep-alive
- Accept-Encoding：说明自己可以接收的压缩方式
- User-Agent：用户代理，是服务器能识别客户端的操作系统（Android、IOS、WEB）及相关的信息。作用是帮助服务器区分客户端，并且针对不同客户端让用户看到不同数据，做不同操作。
- Content-Type：服务器告诉客户端数据的格式，常见的值有text/plain，image/jpeg，image/png，video/mp4，application/json，application/zip。这些数据类型总称为MIME TYPE。
- Content-Encoding：服务器数据压缩方式
- Transfer-Encoding：chunked表示采用分块传输编码，有该字段则无需使用Content-Length字段。
- Content-Length：声明数据的长度，请求和回应头部都可以使用该字段。



## 7.TCP三次握手

![image-20190126113113606](/Users/wangxiong/Library/Application Support/typora-user-images/image-20190126113113606.png)

- ACK：响应标识，1表示响应，连接建立成功之后，所有报文段ACK的值都为1
- SYN：连接标识，1表示建立连接，连接请求和连接接受报文段SYN=1，其他情况都是0
- FIN：关闭连接标识，1标识关闭连接，关闭请求和关闭接受报文段FIN=1，其他情况都是0，跟SYN类似
- seq number：序号，一个随机数X，请求报文段中会有该字段，响应报文段没有
- ack number：应答号，值为请求seq+1，即X+1，除了连接请求和连接接受响应报文段没有该字段，其他的报文段都有该字段

三次握手的具体流程：

1. 第一次握手：建立连接请求。客户端发送连接请求报文段，将SYN置为1，seq为随机数x。然后，客户端进入SYN_SEND状态，等待服务器确认。
2. 第二次握手：确认连接请求。服务器收到客户端的SYN报文段，需要对该请求进行确认，设置ack=x+1（即客户端seq+1）。同时自己也要发送SYN请求信息，即SYN置为1，seq=y。服务器将SYN和ACK信息放在一个报文段中，一并发送给客户端，服务器进入SYN_RECV状态。
3. 第三次握手：客户端收到SYN+ACK报文段，将ack设置为y+1，向服务器发送ACK报文段，这个报文段发送完毕，客户端和服务券进入ESTABLISHED状态，完成Tcp三次握手。

断开连接经历了四次挥手：

1. 第一次挥手：主机1（可以是客户端或服务器），设置seq和ack向主机2发送一个FIN报文段，此时主机1进入FIN_WAIT_1状态，表示没有数据要发送给主机2了
2. 第二次挥手：主机2收到主机1的FIN报文段，向主机1回应一个ACK报文段，表示同意关闭请求，主机1进入FIN_WAIT_2状态。
3. 第三次挥手：主机2向主机1发送FIN报文段，请求关闭连接，主机2进入LAST_ACK状态。
4. 第四次挥手：主机1收到主机2的FIN报文段，想主机2回应ACK报文段，然后主机1进入TIME_WAIT状态；主机2收到主机1的ACK报文段后，关闭连接。此时主机1等待主机2一段时间后，没有收到回复，证明主机2已经正常关闭，主机1页关闭连接。

## 8.HTTPS协议/SSL协议

Https协议是以安全为目标的Http通道，简单来说就是Http的安全版。主要是在Http下加入SSL层（现在主流的是SLL/TLS），SSL是Https协议的安全基础。Https默认端口号为443。

**Http存在的风险吗**

1. 窃听风险：Http采用明文传输数据，第三方可以获知通信内容
2. 篡改风险：第三方可以修改通信内容
3. 冒充风险：第三方可以冒充他人身份进行通信

SSL/TLS协议就是为了解决这些风险而设计，希望达到：

1. 所有信息加密传输，三方窃听通信内容
2. 具有校验机制，内容一旦被篡改，通信双发立刻会发现
3. 配备身份证书，防止身份被冒充

下面主要介绍SSL/TLS协议。

## 9.SSL发展史（互联网加密通信）

1. 1994年NetSpace公司设计SSL协议（Secure Sockets Layout）1.0版本，但未发布。
2. 1995年NetSpace发布SSL/2.0版本，很快发现有严重漏洞
3. 1996年发布SSL/3.0版本，得到大规模应用
4. 1999年，发布了SSL升级版TLS/1.0版本，目前应用最广泛的版本
5. 2006年和2008年，发布了TLS/1.1版本和TLS/1.2版本

### **SSL原理及运行过程**

SSL/TLS协议基本思路是采用公钥加密法（最有名的是RSA加密算法）。大概流程是，客户端向服务器索要公钥，然后用公钥加密信息，服务器收到密文，用自己的私钥解密。

为了防止公钥被篡改，把公钥放在数字证书中，证书可信则公钥可信。公钥加密计算量很大，为了提高效率，服务端和客户端都生成对话秘钥，用它加密信息，而对话秘钥是对称加密，速度非常快。而公钥用来机密对话秘钥。

过程

1. 客户端给出协议版本号、一个客户端随机数A（Client random）以及客户端支持的加密方式
2. 服务端确认双方使用的加密方式，并给出数字证书、一个服务器生成的随机数B（Server random）
3. 客户端确认数字证书有效，生成一个新的随机数C（Pre-master-secret），使用证书中的公钥对C加密，发送给服务端
4. 服务端使用自己的私钥解密出C
5. 客户端和服务器根据约定的加密方法，使用三个随机数ABC，生成对话秘钥，之后的通信都用这个对话秘钥进行加密

### **SSL证书**

Https协议中需要使用到SSL证书。


SSL证书是一个二进制文件，里面包含经过认证的网站公钥和一些元数据，需要从经销商购买。
证书有很多类型，按认证级别分类：



- 域名认证（DV=Domain Validation）：最低级别的认证，可以确认申请人拥有这个域名
- 公司认证（OV=Organization Validation）：确认域名所有人是哪家公司，证书里面包含公司的信息
- 扩展认证（EV=Extended Validation）：最高级别认证，浏览器地址栏会显示公司名称。

EV证书浏览器地址栏样式：

![image-20190126113539173](/Users/wangxiong/Library/Application Support/typora-user-images/image-20190126113539173.png)

OV证书浏览器地址栏样式：

![image-20190126113600356](/Users/wangxiong/Library/Application Support/typora-user-images/image-20190126113600356.png)

DV证书浏览器样式：

![image-20190126113616643](/Users/wangxiong/Library/Application Support/typora-user-images/image-20190126113616643.png)

### **RSA加密和DH加密**

#### **加密算法分类**

加密算法分为对称加密、非对称加密和Hash加密算法。

- 对称加密：甲方和乙方使用同一种加密规则对信息加解密
- 非对称加密：乙方生成两把秘钥（公钥和私钥）。公钥是公开的，任何人都可以获取，私钥是保密的，只存在于乙方手中。甲方获取公钥，然后用公钥加密信息，乙方得到密文后，用私钥解密。
- Hash加密：Hash算法是一种单向密码体制，即只有加密过程，没有解密过程

对称加密算法加解密效率高，速度快，适合大数据量加解密。常见的堆成加密算法有DES、AES、RC5、Blowfish、IDEA


非对称加密算法复杂，加解密速度慢，但安全性高，一般与对称加密结合使用（对称加密通信内容，非对称加密对称秘钥）。

常见的非对称加密算法有RSA、DH、DSA、ECC


Hash算法特性是：输入值一样，经过哈希函数得到相同的散列值，但并非散列值相同则输入值也相同。常见的Hash加密算法有MD5、SHA-1、SHA-X系列

**下面着重介绍一下RSA算法和DH算法。**

#### **RSA加密算法**

Https协议就是使用RSA加密算法，可以说RSA加密算法是宇宙中最重要的加密算法。
RSA算法用到一些数论知识，包括互质关系，欧拉函数，欧拉定理。此处不具体介绍加密的过程，如果有兴趣，可以参照RSA算法加密过程。

*http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html*
RSA算法的安全保障基于大数分解问题，目前破解过的最大秘钥是700+位，也就代表1024位秘钥和2048位秘钥可以认为绝对安全。

大数分解主要难点在于计算能力，如果未来计算能力有了质的提升，那么这些秘钥也是有可能被破解的。

#### **DH加密算法**

DH也是一种非对称加密算法，DH加密算法过程。

*https://zh.wikipedia.org/wiki/%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E7%88%BE%E6%9B%BC%E5%AF%86%E9%91%B0%E4%BA%A4%E6%8F%9B*DH算法的安全保障是基于离散对数问题

## 10.SSLSocket

SSLSocket扩展Socket并提供使用SSL或TLS协议的安全套接字，简单来说SSLSocket通信就是需要服务端和客户端进行证书验证的socket通信

SSL:Secure Sockets Layer 安全套接层

密钥对：公钥和私钥

密钥对是一起产生的 ，可以互相加密解密。对证书而言私钥由自己保存，用于签名信息。公钥发给想要通信的一端，用于验证信息是你发过来的。一般只用私钥进行签名。

## 11.证书验证（Certificate）

支持hppts的网站基本上都是CA机构颁发的证书，默认情况下是可以信任的。但是有一些网站的证书是自签名的，使用浏览器访问的时候会报风险警告，比如12306网站。正常访问有证书的服务端，需要进行证书的校验，对于权威机构颁发的证书当然我们会直接认为合法。对于自签名的证书，那么我们就需要去校验合法性了，也就是说我们只需要让OkhttpClient去信任这个证书就可以畅通的进行通信了。当然，对于自签名的网站的访问，网上的部分的做法是直接设置信任所有的证书，对于这种做法肯定是有风险的。

配置服务端自签名的证书

我们可以制作自签名的证书，通过keytool生成证书。

1.先生成一个证书请求文件wx_server.jks。2.再利用wx_server.jks生成包含公钥的证书wx_server.cer。3.将生成的jks文件配置到服务端。客户端就可以利用wx_server.cer文件进行证书校验。

双向证书验证

即客户端也有一对wx_client.jks文件和wx_client.cer文件。然后进行配置

1.配置服务端信任wx_client.cer文件。再用浏览器访问就访问不到服务端了。会显示证书身份验证失败。

2.配置客户端，利用wx_client.jks文件。进行设置

Java平台默认识别jks格式的证书文件，但是android平台只识别bks格式的证书文件。我们需要将我们的jks文件转化为bks文件。通过bcprov.jar文件将wx_client.jks文件转换为wx_client.bks文件

## 12.DNS(Domain Name System，域名系统)

DNS服务用于网络请求时，将域名转为IP地址。

传统的基于UDP协议的公共DNS服务极容易发生DNS劫持，从而造成安全问题。

**HTTPDns**

HTTPDNS利用HTTP协议与DNS服务器交互，代替了传统的基于UDP协议的DNS交互，绕开运营商的LocalDNS，有效防止了域名劫持。

## 13.Cookie

cookie的结构
key:cookie的名称
value:cookie的值
expires:cookie的失效日期（UTC时间字符串）
max-age:cookie的失效间隔（秒），优先级高于expires
path:根据目录限制cookie的分享，如不主动设置，默认为当前页面的路径

domain:根据域名限制cookie的分享

**具体使用**：服务端先设置cookie信息，并在客户端请求时把这个cookie信息发送给客户端，客户端会自动保存cookie的key/value值，下次向服务端发送请求时，客户端会自动带上cookie信息，服务端会根据cookie信息来识别状态。（之前是否访问过）




