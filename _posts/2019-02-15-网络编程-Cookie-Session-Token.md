---
layout: post
title: "「网络编程」 六、Cookie、Session、Token"
subtitle: 'Android'
date:       2019-02-15
author: "Wangxiong"
header-style: text
tags:
  - Cookie
  - 网络编程
---
## 1. Cookie

Http协议是无状态的协议，意味着服务端无法从连接上跟踪回话状态。而Cookie技术是针对客户端的解决方案，Cookie是由服务端发送给客户端的特殊信息，这些信息以文件的方式存放在客户端，然后客户端每次访问服务器发送请求的时候都会带上这些特殊信息，

### 1.1 Cookie的结构

- key:cookie的名称
- value:cookie的值
- expires:cookie的失效日期（UTC时间字符串）
- max-age:cookie的失效间隔（秒），优先级高于expires
- path:根据目录限制cookie的分享，如不主动设置，默认为当前页面的路径
- domain:根据域名限制cookie的分享

### 1.2 Cookie的具体使用

服务端先设置cookie信息，并在客户端请求时把这个cookie信息发送给客户端，客户端会自动保存cookie的key/value值，下次向服务端发送请求时，客户端会自动带上cookie信息，服务端会根据cookie信息来识别状态。（之前是否访问过）

- 服务端Response响应头携带的Cookie信息

- ```http
  Set-Cookie	WXUID=WXE11:FG=1; expires=Thu, 02-Apr-20 02:43:01 GMT; max-age=31536000; path=/; domain=.baidu.com; version=1
  ```

- 客户端Request请求头携带的Cookie信息

- ```http
  Cookie	WMST=1554206453; WMID=ab69b1a2218fdef88ef4574e997662fc
  ```

代码上可以直接使用OkHttp添加Interceptor的方式，拦截Response保存Cookie，拦截Request添加Cookie。

## 2. Session

Session是针对服务端的技术方案，客户端没有Session。Session是服务端在和客户端建立连接时添加的客户端连接标志，最终在服务端转化为一个临时的Cookie发送给客户端，当客户端第一次请求时服务端检查客户端是否携带了这个Session(Cookie)，如果没有则会添加Session，如果有就好拿出这个Session做相关操作。

## 3. Token

token是用户身份的验证方式，通常称为令牌，主要用于身份验证，比如你授权（登录）一个程序时，他就是个依据，判断你是否已经授权该软件；cookie就是写在客户端的一个txt文件，里面包括你登录信息之类的，这样你下次在登录某个网站，就会自动调用cookie自动登录用户名；session和cookie差不多，只是session是写在服务器端的文件，也需要在客户端写入cookie文件，但是文件里是你的浏览器编号.Session的状态是存储在服务器端，客户端只有session id；而Token的状态是存储在客户端。

## 4. Cookie和Session的区别

- 1、cookie数据存放在客户的浏览器上，session数据放在服务器上。
- 2、cookie不是很安全，别人可以分析存放在本地的cookie并进行cookie欺骗,考虑到安全应当使用session。
- 3、session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能,考虑到减轻服务器性能方面，应当使用cookie。
- 4、单个cookie保存的数据不能超过4K，很多浏览器都限制一个站点最多保存20个cookie。