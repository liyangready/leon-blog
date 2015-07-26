title: "构建一个支持https的透明代理"
date: 2015-07-26 16:26:27
tags: nodejs
---

##需求

在做 [多host工具](https://github.com/liyangready/multiple-host) 项目时，我需要一个透明代理，让浏览器的所有的请求通过一个nodejs server转发，从而可以实现修改请求的目的。问题就是，如何构建一个能够同时支持http && https的web proxy。

##http

支持http请求有非常简单，通过 http.createSever起一个服务，然后让浏览器设置代理到这个server， 在 server 的 req 中能够拿到原请求头的所有信息，我们解析出host之后请求远端服务器，回数了之后再传给浏览器。

两个方法： `http.createServer` 和 `http.request` 能够轻松搞定，要注意的就是由于http 1.1的存在，最好是用流来解决。

##https

https不能同http一样的思路，为什么？ 我们先看看https的基本知识。

[SSL/TLS协议运行机制的概述](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)  
[HTTPS连接的前几毫秒发生了什么](http://blog.jobbole.com/48369/)

大致的流程就是：  

+ client 给服务器端发送一个hello信息(包含支持什么协议/解密方法/随机数等)
+ 服务器端返回一个hello信息(包含使用了什么加密算法)和证书(由CA通过SHA1签名，包括服务器端的公钥)
+ client信任CA的证书，所有信任server端的公钥，通过公钥加密一个随机数发给服务器端
+ 服务器端通过私钥解密，协商出一个新的对称加密秘钥(因为对称加密快，非对称太消耗性能，只能用于刚开始的协商阶段)和方法，返给客户端。
+ 客户端和服务器端使用对称加密传输信息。

我们看到了其中关键的就是前期的协商阶段需要CA签名的证书，我们如果还用 https.createSever之类的，就需要自己搞个证书，而我们的代理其实只是做一个中间人，并不需要修改传输内容，有没有更简单的办法？

## 方案一
方案一返璞归真，在http/https协议这一层做代理需要区分是http还是https，很头疼，而且https还搞不定。那么我们往深了想，http和https其实都是基于TCP/IP的，或者说是socket协议的，由于我们并不需要修改传输内容，用socket协议代理server只需要和client以及远端的server保持两个socket连接，传递内容就行了，并不需要关心内容是什么。

<pre>
var net = require("net");

net.createServer(function(connect) {
	console.log("client connected");
	connect.on("data", function(data) {
		debugger
		console.log(data.toString());
		//解析头，然后做net.connet...
	});
	connect.on("end", function() {
		console.log('client disconnected');
	})
}).listen(9393);

</pre>
打开系统代理设置，把代理设置成 127.0.0.1:9393，然后通过浏览器访问http和https的网站，发现都能被记录下来。

**http console**:

<pre>
$ node test.js
client connected
GET http://www.hao123.com/ HTTP/1.1
Accept: text/html, application/xhtml+xml, */*
Accept-Language: zh-CN
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko
Accept-Encoding: gzip, deflate
Host: www.hao123.com
DNT: 1
Proxy-Connection: Keep-Alive
Cookie: BAIDUID=079434675F842CCB0BC016833D11806F:FG=1; Hm_lvt_22661fc940aadd927d385f4a67892bc3=1426167798,1426212566; HUM=; HUN=; m
 hz=0
</pre>

**https console**

<pre>
client connected
CONNECT www.baidu.com:443 HTTP/1.0
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko
Host: www.baidu.com
Content-Length: 0
DNT: 1
Proxy-Connection: Keep-Alive
Pragma: no-cache
</pre>
这个方案很可行，通过 nodejs的 `net` 模块可以建立socket连接。

写了一半放弃的原因是，这个方案的确可以完美实现需求，但是需要自己去处理 http和https协议头，比如说通过socket协议来了一个请求，我想知道http头里面的host port等等信息，都只能从数据流中再做字符串解析，特别麻烦，而用http.createServer这种方法，实际上是可以使用已经封装好的方法来处理头信息。于是我在想有没有更简单的办法实现？

##方案二
我们在上面的 net server中看到一个奇怪的东西，就是 https的console竟然会以明文的形式出现：
<pre>
client connected
CONNECT www.baidu.com:443 HTTP/1.0
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko
Host: www.baidu.com
Content-Length: 0
</pre>

没有content-length的一条消息，协议名是 **CONNECT** , 看来浏览器在给代理server传递消息的时候如果是https就会走一个 connect的协议，我们看看这个connect协议是啥？

原来http1.1引入了一个叫做 http tunnel的代理协议，也就是上面那个 connect协议，专门用来处理代理事宜的。

[HTTP tunnel](https://en.wikipedia.org/wiki/HTTP_tunnel)    
[When should one use CONNECT and GET HTTP methods at HTTP Proxy Server](http://stackoverflow.com/questions/11697943/when-should-one-use-connect-and-get-http-methods-at-http-proxy-server)

> A CONNECT request urges your proxy to establish an HTTP tunnel to the remote end-point. Usually is it used for SSL connections, though it can be used with HTTP as well (used for the purposes of proxy-chaining and tunneling)

> CONNECT www.google.com:443 
The above line opens a connection from your proxy to www.google.com on port 443. After this, content that is sent by the client is forwarded by the proxy to www.google.com:443.

> If a user tries to retrieve a page http://www.google.com, the proxy can send the exact same request and retrieve response for him, on his behalf.

> With SSL(HTTPS), only the two remote end-points understand the requests, and the proxy cannot decipher them. Hence, all it does is open that tunnel using CONNECT, and lets the two end-points (webserver and client) talk to each other directly.

意思就是说可以通过 connect 协议在 客户端和服务器中间按照一个代理人，一般用于ssl(https的一种实现)，在发送https请求之前通过connect和代理人建立连接，然后代理人负责转发，但是代理人也不能知道传输的内容，因为是加密的。

**这正好是我要的滑板鞋啊！**

浏览器发现设置了系统代理时，如果是https请求就会尝试通过 connect 协议和proxy server发生连接，而这个connect协议里面就有 一些基本的原请求信息，而且它是http协议的一部分,完美！

我们看看nodejs有没有这个协议支持：

> Event: 'connect'#
function (request, socket, head) { }

> Emitted each time a client requests a http CONNECT method. If this event isn't listened for, then clients requesting a CONNECT method will have their connections closed.

> request is the arguments for the http request, as it is in the request event.
socket is the network socket between the server and client.
head is an instance of Buffer, the first packet of the tunneling stream, this may be empty.
After this event is emitted, the request's socket will not have a data event listener, meaning you will need to bind to it in order to handle data sent to the server on that socket.

所以方案二诞生了：

<pre>
 var server = http.createServer();
 //requestHandler处理http请求，拿到origin host,然后通过http.request和远端连接
 server.on("request", this.requestHandler);
 //connectHandler处理https请求，通过connect格式拿到origin host，然后通过socket和远端连接
 server.on("connect", this.connectHandler);
</pre>

具体实现可看 最终版：

[基于nodejs的迷你易用的proxy](https://github.com/liyangready/mini-proxy)