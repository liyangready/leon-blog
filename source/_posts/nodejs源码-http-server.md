title: "nodejs源码-http server"
date: 2015-09-16 14:20:46
tags:
---
<pre>
var http = require('http');
http.createServer(function(req, res) {
  res.write('ok');
  res.end();
}).listen(3000);
</pre>

本文主要想探索一下上面这段代码在nodejs内部到底发生了什么？

## http.createServer
nodejs源码在github上面，其中js部分源码存在了 node/lib 目录下。我们先看 http 模块的代码。
`node/lib/http.js`:   
<pre>
const server = require('_http_server');
...
const Server = exports.Server = server.Server;
...
exports.createServer = function(requestListener) {
  return new Server(requestListener);
};
</pre> 

可以清楚的看出，createServer返回了一个Server的实例，然后将用户的回调传递进去，Server来自`node/lib/_http_server_`：
<pre>
function Server(requestListener) {
  if (!(this instanceof Server)) return new Server(requestListener);
  net.Server.call(this, { allowHalfOpen: true });
  if (requestListener) {
    this.addListener('request', requestListener);
  }
  this.httpAllowHalfOpen = false;
  this.addListener('connection', connectionListener);
  this.addListener('clientError', function(err, conn) {...});
  this.timeout = 2 * 60 * 1000;
}
util.inherits(Server, net.Server);
</pre>
从上面代码可以看出，Server会继承net.Server，同时自己增加了 httpAllowHalfOpen(用到再说) 和默认的超时时间(2min)，绑定了三个事件(onrequest-即用户在createServer传递的事件、onconnection和onclientError);
先不管事件，继续看 net.Server,位于`node/lib/net.js`:

<pre>
function Server(options, connectionListener) {
  // options: { allowHalfOpen: true }, connectionListener: undefined
  ...
  events.EventEmitter.call(this); //继承事件，让server拥有事件能力
  ...
 this._connections = 0;
 //这一段定义connections了connections，标记其get set时候都不能被外部使用，否则打印一个warning
 Object.defineProperty(this, 'connections', {
    get: internalUtil.deprecate(function() {
      if (self._usingSlaves) {
        return null;
      }
      return self._connections;
    }, 'Server.connections property is deprecated. ' +
       'Use Server.getConnections method instead.'),
    set: internalUtil.deprecate(function(val) {
      return (self._connections = val);
    }, 'Server.connections property is deprecated.'),
    configurable: true, enumerable: false
  });

  this._handle = null;
  this._usingSlaves = false;
  this._slaves = [];
  this._unref = false;

  this.allowHalfOpen = options.allowHalfOpen || false;
  this.pauseOnConnect = !!options.pauseOnConnect;
  //---------调用结束---------
}
util.inherits(Server, events.EventEmitter);//继承事件，让server拥有事件能力
</pre>
由上，实际上net.Server不过是定义了一堆变量挂在自己身上，因此 http Server也挂了这些属性到this上面。到此，createServer其实已经结束了。

<!-- more -->

## Server.listen(3000)

由于http.Server是继承的 net.Server,listen方法也来自 net.Server.listen:
<pre>
Server.prototype.listen = function() {
  ...
  var backlog = toNumber(arguments[1]) || toNumber(arguments[2]);
  if (typeof lastArg === 'function') {
    self.once('listening', lastArg); //绑定listen的回调
  }
  if (arguments...) {
   //一堆关于arguments写法的判断，在这里我们只关注自己的情况，只有一个port:3000
   
  }
  else if (arguments[1] === undefined ||
             typeof arguments[1] === 'function' ||
             typeof arguments[1] === 'number') {
    // The first argument is the port, no IP given.
    listen(self, null, port, 4, backlog); //调用function listen
  }
}
function listen(self, address, port, addressType, backlog, fd, exclusive) {
   //self -> server, port-> 3000, addressType -> 4, backlog-> false
   exclusive = !!exclusive;
   if (!cluster) cluster = require('cluster');

   if (cluster.isMaster || exclusive) {
    self._listen2(address, port, addressType, backlog, fd);//调用function listen2
    return;
   }
}
Server.prototype._listen2 = function(address, port, addressType, backlog, fd) {
  //port-> 3000, addressType -> 4, backlog-> false
  if (self._handle) {
    debug('_listen2: have a handle already');
  } else { //我们的调用没有handle
  	var rval = null;

    if (!address && typeof fd !== 'number') {
      rval = createServerHandle('::', port, 6, fd); //这里调用createServerHandle
    }
    ...
    self._handle = rval; //rval是TCP实例，绑定了一个 ipv6
    self._handle.onconnection = onconnection;//TCP的onconnection事件绑定
    self._handle.owner = self;//通过owner指向server
    var err = _listen(self._handle, backlog);
    if (err) {...} 
    this._connectionKey = addressType + ':' + address + ':' + port;
    //设置一个 connection的key值 4：null：3000
     // unref the handle if the server was unref'ed prior to listening
    if (this._unref) //如果是_unref状态，调用unref方法，unref会在event loop中
                     //创建一个活跃的timer，保持程序运行  
      this.unref();
    }
    process.nextTick(emitListeningNT, self); //在下一个tick 触发 listening事件，
                                             //listening在Server.prototype.listen
                                             //中有定义，就是用户的回调
    //------------调用结束-----------------

}
...
var createServerHandle = exports._createServerHandle = function(address, port, addressType, fd) {
 // address-> '::' port -> 3000 addressType -> 4 fd -> null
 if (typeof fd === 'number' && fd >= 0) {...}
 else if (port === -1 && addressType === -1) {...}
 else {
    handle = new TCP(); //prcess.binding 来自原生模块 暂且不表
    isTCP = true;
 }
 if (address || port || isTCP) {
    if (!address) {
      err = handle.bind6('::', port); //先尝试绑定ipv6 为什么这样还不懂
      //https://github.com/nodejs/node-v0.x-archive/issues/9195 这里有关于此的讨论
      if (err) {
        handle.close();
        // Fallback to ipv4
        return createServerHandle('0.0.0.0', port);
     }
 }
   return handle;
}
...
function _listen(handle, backlog) {
  // Use a backlog of 512 entries. We pass 511 to the listen() call because
  // the kernel does: backlogsize = roundup_pow_of_two(backlogsize + 1);
  // which will thus give us a backlog of 512 entries.
  return handle.listen(backlog || 511);
  //设置connection pending的最大数目，默认是512个
}

function emitListeningNT(self) {
  // ensure handle hasn't closed
  if (self._handle)
    self.emit('listening');
}

</pre>

我们回过头来看这一部分，这一部分做的事情其实也很简单，先是处理了一大堆arguments，即listen可以传的方法，我们就传了一个port，就会调用 listen方法，listen方法判断是不是主进程，我们也是，就会调用_listen2方法，_listen2方法创建了一个c++模块的TCP实例（默认尝试ipv6），并和当前server关联起来，给这个TCP实例绑上了onconnect方法，设置了默认的连接数和端口号。
## onconnection

nodejs通过底层的libuv监听了我们的3000端口，当请求来临之时，TCP实例会触发onconnection事件，上面我们提到了`self._handle.onconnection = onconnection;`,所以请求到来会首先触发 `node/lib/net.js`中的onconnection 事件:
<pre>
function onconnection(err, clientHandle) {
  // err -> null, clientHandle -> 客户端的TCP实例
  var handle = this; //TCP
  var self = handle.owner; //前面提到的TCP的owner属性会指向Server
  if(err) {...}
  if (self.maxConnections && self._connections >= self.maxConnections) {
    clientHandle.close();
    return;
  }
  var socket = new Socket({
    handle: clientHandle, //TCP
    allowHalfOpen: self.allowHalfOpen, //是否支持半连接,false
    pauseOnCreate: self.pauseOnConnect //false
  });
  //此处会创建一个 socket实例，socket构造函数在下面
  //从构造函数里面可以看出来，暂时scoket只是加了很多熟悉，有三个方法 onfinish,on_socketEnd和onread
  
  socket.readable = socket.writable = true;
  self._connections++; //连接数+1
  socket.server = self; //socket的server指向上述的Server实例
  
  DTRACE_NET_SERVER_CONNECTION(socket);
  LTTNG_NET_SERVER_CONNECTION(socket);
  COUNTER_NET_SERVER_CONNECTION(socket);
  self.emit('connection', socket); //触发了Server的onconnection事件,在最上面提到的_http_server_中定义了 connectionListener
  
}
function Socket(options) {
  if (!(this instanceof Socket)) return new Socket(options);
  this._connecting = false;
  this._hadError = false;
  this._handle = null;
  this._parent = null;
  this._host = null;

  if (typeof options === 'number')
    options = { fd: options }; // Legacy interface.
  else if (options === undefined)
    options = {};
  if (options.handle) {
    this._handle = options.handle; // 从onconnection传递过来的客户端TCP
  }
  stream.Duplex.call(this, options);

  this.on('finish', onSocketFinish); //socket结束事件
  this.on('_socketEnd', onSocketEnd); //socket end事件
  initSocketHandle(this); //把socket和handle连接起来，同时加了一堆属性
}
util.inherits(Socket, stream.Duplex); // socket继承双工流，可读写

function initSocketHandle(self) {
  self.destroyed = false;
  self.bytesRead = 0;
  self._bytesDispatched = 0;
  self._sockname = null;

  // Handle creation may be deferred to bind() or connect() time.
  if (self._handle) {
    self._handle.owner = self; //客户端TCP和self绑定起来
    self._handle.onread = onread;

    // If handle doesn't support writev - neither do we
    if (!self._handle.writev)
      self._writev = null;
  }
}

</pre>

在上面我们看到了 `maxConnections` 和 `backlog` 两个参数，貌似都与连接数有关系，有什么区别别弄混了：

 `backlog`：是给底层TCP连接用的，当超过512(默认)的时候，底层TCP将不再接受请求，剩余请求排队等待。其有可能小于512，受操作系统内核影响。[tcp_max_syn_backlog](http://blog.duteba.com/technology/article/84.htm)      

！！！无语，就知道这个不靠谱，特意多搜了几下，这篇文章说：  
> net.ipv4.tcp_max_syn_backlog参数决定了SYN_RECV状态队列的数量，一般默认值为512或者1024，即超过这个数量，系统将不再接受新的TCP连接请求，一定程度上可以防止系统资源耗尽。可根据情况增加该值以接受更多的连接请求。

看起来就是内核只能同时处理512个请求，超过就不会在接受只能排队了，而在node中，bigpipe、websocket的使用都很常见，长连接难道node或者说操作系统只能处理512个？越想越不对啊，于是查了点其它资料，看linux手册：    
>  tcp_max_syn_backlog (integer; default: see below; since Linux 2.2)
              The maximum number of queued connection requests which have
              still not received an acknowledgement from the connecting
              client.  If this number is exceeded, the kernel will begin
              dropping requests.  The default value of 256 is increased to
              1024 when the memory present in the system is adequate or
              greater (>= 128Mb), and reduced to 128 for those systems with
              very low memory (<= 32Mb).
              Prior to Linux 2.6.20, it was recommended that if this needed
              to be increased above 1024, the size of the SYNACK hash table
              (TCP_SYNQ_HSIZE) in include/net/tcp.h should be modified to
              keep  TCP_SYNQ_HSIZE * 16 <= tcp_max_syn_backlog and the kernel should be recompiled.  In Linux 2.6.20, the
              fixed sized TCP_SYNQ_HSIZE was removed in favor of dynamic
              sizing.

可以看出实际上这个队列维护的是三次握手中 第二次握手之后服务器发给客户端等待的连接，和keep-alive没有半毛线关系，它指的是如果二次握手之后客户端没有响应，就会看超过512之后丢弃。并不影响已经建立好的连接。[三次握手](http://baike.baidu.com/link?url=XQceOnFrI6nhdPNK42Gc4UEK1g_ay-wU2oqFTg7r8i3HzSKKKvpsXsryiSX2jFXi5-Jthoxrl_cNMe7Pp2-3cq) [靠谱资料](http://man7.org/linux/man-pages/man7/tcp.7.html)

`maxConnections`: node控制的连接请求数，当超过这个值了之后，连接会直接关闭，以减少操作系统的负担。这个参数会影响请求，当超过这个值了之后请求直接挂了，而 backlog不过是排队而已。

## connectionListener
<pre>
//此方法由上面的  self.emit('connection', socket);  触发而来
function connectionListener() {
  var self = this;
  var outgoing = [];
  var incoming = [];
  httpSocketSetup(socket); //重新绑定drain事件
  if (self.timeout) //如果有设置超时
    socket.setTimeout(self.timeout);
  socket.on('timeout', function() {...});
  var parser = parsers.alloc(); //parsers定义在下面
  parser.reinitialize(HTTPParser.REQUEST); //native的方法 todo
  parser.socket = socket;
  socket.parser = parser;
  parser.incoming = null;
  // Propagate headers limit from server instance to parser
  if (util.isNumber(this.maxHeadersCount)) {
    parser.maxHeaderPairs = this.maxHeadersCount << 1;
  } else {
    // Set default value because parser may be reused from FreeList
    parser.maxHeaderPairs = 2000;
  }

  socket.addListener('error', socketOnError);
  socket.addListener('close', serverSocketCloseListener);
  parser.onIncoming = parserOnIncoming;
  socket.on('end', socketOnEnd);
  socket.on('data', socketOnData);
  socket._paused = false;
  socket.on('drain', socketOnDrain);

}

function httpSocketSetup(socket) { //位于 lib/_http_common
  // socket -> Scoket实例
  socket.removeListener('drain', ondrain);
  socket.on('drain', ondrain);
}
function ondrain() {
  if (this._httpMessage) this._httpMessage.emit('drain');
}

var parsers = new FreeList('parsers', 1000, function() {
  //parsers是一个 FreeList数据结构的实例，freelist定义如下
  var parser = new HTTPParser(HTTPParser.REQUEST);

  parser._headers = [];
  parser._url = '';

  // Only called in the slow case where slow means
  // that the request headers were either fragmented
  // across multiple TCP packets or too large to be
  // processed in a single run. This method is also
  // called to process trailing HTTP headers.
  parser[kOnHeaders] = parserOnHeaders;
  parser[kOnHeadersComplete] = parserOnHeadersComplete;
  parser[kOnBody] = parserOnBody;
  parser[kOnMessageComplete] = parserOnMessageComplete;

  return parser;
});
exports.FreeList = function(name, max, constructor) {
  this.name = name;
  this.constructor = constructor;
  this.max = max;
  this.list = [];
};
</pre>

## onread
native代码会在有数据的时候出发 onread事件
<pre>
function onread(nread, buffer) {
  var handle = this; //TCP
  var self = handle.owner; //Scoket
  self._unrefTimer(); //这个未知
  if (nread > 0) {
   //nread的状态待查 , >0代表成功
    self.bytesRead += nread;
    var ret = self.push(buffer); //buffer就是收到的数据
    //buffer 被push的时候,会触发 ondata事件
    if (handle.reading && !ret) {
      handle.reading = false;
      var err = handle.readStop();
      if (err)
        self._destroy(errnoException(err, 'read'));
    }
    return;
  }

}
function socketOnData(d) {
    var ret = parser.execute(d);
    //native代码，会触发 parserOnHeadersComplete -> parserOnIncoming
    onParserExecuteCommon(ret, d);
}

function parserOnHeadersComplete(info) {
 var parser = this;
  var headers = info.headers;
  var url = info.url;

  if (!headers) {
    headers = parser._headers;
    parser._headers = [];
  }

  if (!url) {
    url = parser._url;
    parser._url = '';
  }

  parser.incoming = new IncomingMessage(parser.socket);
  parser.incoming.httpVersionMajor = info.versionMajor;
  parser.incoming.httpVersionMinor = info.versionMinor;
  parser.incoming.httpVersion = info.versionMajor + '.' + info.versionMinor;
  parser.incoming.url = url;

  var n = headers.length;

  // If parser.maxHeaderPairs <= 0 - assume that there're no limit
  if (parser.maxHeaderPairs > 0) {
    n = Math.min(n, parser.maxHeaderPairs);
  }

  parser.incoming._addHeaderLines(headers, n);

  if (isNumber(info.method)) {
    // server only
    parser.incoming.method = HTTPParser.methods[info.method];
  } else {
    // client only
    parser.incoming.statusCode = info.statusCode;
    parser.incoming.statusMessage = info.statusMessage;
  }

  parser.incoming.upgrade = info.upgrade;

  var skipBody = false; // response to HEAD or CONNECT

  if (!info.upgrade) {
    // For upgraded connections and CONNECT method request,
    // we'll emit this after parser.execute
    // so that we can capture the first part of the new protocol
    skipBody = parser.onIncoming(parser.incoming, info.shouldKeepAlive);
  }

  return skipBody;
}

function parserOnIncoming(req, shouldKeepAlive) {
  incoming.push(req); //req 是一个 IncomingMessage的实例
  var res = new ServerResponse(req); //继承outGoingMessage 以后再讲
  res.on('finish', resOnFinish);
  self.emit('request', req, res); //简化版，触发request事件,这就是用户的回调！

}
</pre>

## req 和  res
req和res是http Server上回调参数，非常重要，单拿出来讲。
req是上面看到了是`parser.incoming = new IncomingMessage(parser.socket);` 创建的 IncommingMessage的实例：

<pre>
function IncomingMessage(socket) {
  Stream.Readable.call(this);

  // XXX This implementation is kind of all over the place
  // When the parser emits body chunks, they go in this list.
  // _read() pulls them out, and when it finds EOF, it ends.

  this.socket = socket;
  this.connection = socket;
  this.httpVersionMajor = null;
  this.httpVersionMinor = null;
  this.httpVersion = null;
  this.complete = false;
  this.headers = {};
  this.rawHeaders = [];
  this.trailers = {};
  this.rawTrailers = [];
  this.readable = true;
  this.upgrade = null;
  // request (server) only
  this.url = '';
  this.method = null;

  // response (client) only
  this.statusCode = null;
  this.statusMessage = null;
  this.client = socket;

  // flag for backwards compatibility grossness.
  this._consuming = false;

  // flag for when we decide that this message cannot possibly be
  // read by the user, so there's no point continuing to handle it.
  this._dumped = false;
}
util.inherits(IncomingMessage, Stream.Readable);
</pre>
其包含了很多和client TCP的信息，大部分方法分布在lib/_http_incomming里面，我们之前的方法中没有用到，先不讲，看看res.write和res.end是咋会事。res是ServerResponse的实例，大部分方法在lib/_http_outgoing中
<pre>
function ServerResponse(req) {
  OutgoingMessage.call(this);

  if (req.method === 'HEAD') this._hasBody = false;

  this.sendDate = true;

  if (req.httpVersionMajor < 1 || req.httpVersionMinor < 1) {
    this.useChunkedEncodingByDefault = chunkExpression.test(req.headers.te);
    this.shouldKeepAlive = false;
  }
}
util.inherits(ServerResponse, OutgoingMessage);

function OutgoingMessage() {
  Stream.call(this);

  this.output = [];
  this.outputEncodings = [];
  this.outputCallbacks = [];

  this.writable = true;

  this._last = false;
  this.chunkedEncoding = false;
  this.shouldKeepAlive = true;
  this.useChunkedEncodingByDefault = true;
  this.sendDate = false;
  this._removedHeader = {};

  this._contentLength = null;
  this._hasBody = true;
  this._trailer = '';

  this.finished = false;
  this._headerSent = false;

  this.socket = null;
  this.connection = null;
  this._header = null;
  this._headers = null;
  this._headerNames = {};
}
util.inherits(OutgoingMessage, Stream);
</pre>

### res.write
<pre>
OutgoingMessage.prototype.write = function(chunk, encoding, callback) {
  if (this.finished) {...} //如果请求已经关闭
  if (!this._header) {
    this._implicitHeader(); //如果没有头，输入一个默认的头
  }
  if (!this._hasBody) { //对304这种头，不能给写进去body
    debug('This type of response MUST NOT have a body. ' +
          'Ignoring write() calls.');
    return true;
  }

  if (typeof chunk !== 'string' && !(chunk instanceof Buffer)) { //chunk必须是字符串或者buffer
    throw new TypeError('first argument must be a string or Buffer');
  }
  if (chunk.length === 0) return true;
  var len, ret;
  if (this.chunkedEncoding) {
	if (typeof chunk === 'string' &&
        encoding !== 'hex' &&
        encoding !== 'base64' &&
        encoding !== 'binary') {
      len = Buffer.byteLength(chunk, encoding); //字符串或者buffer长度转换成16进制
      chunk = len.toString(16) + CRLF + chunk + CRLF;
      ret = this._send(chunk, encoding, callback);
   }
}

OutgoingMessage.prototype._send = function(data, encoding, callback) {
	// data -> chunk , encoding -> undefined, callback -> undefined
   if (!this._headerSent) { //false
    if (typeof data === 'string' &&
        encoding !== 'hex' &&
        encoding !== 'base64') {
      data = this._header + data; //将_header和要write的数据拼接起来
    } else {
      this.output.unshift(this._header);
      this.outputEncodings.unshift('binary');
      this.outputCallbacks.unshift(null);
    }
    this._headerSent = true;
  }
  return this._writeRaw(data, encoding, callback);

}

OutgoingMessage.prototype._writeRaw = function(data, encoding, callback) {
  if (typeof encoding === 'function') { //callback也可位于第二位
    callback = encoding;
    encoding = null;
  }
  if (data.length === 0) { //length为0，异步回调
    if (typeof callback === 'function')
      process.nextTick(callback);
    return true;
  }
  var connection = this.connection; //socket

  if (connection &&
      connection._httpMessage === this &&
      connection.writable &&
      !connection.destroyed) {
    // There might be pending data in the this.output buffer.
    var outputLength = this.length;
    if (outputLength > 0) { //false
      var output = this.output;
      var outputEncodings = this.outputEncodings;
      var outputCallbacks = this.outputCallbacks;
      for (var i = 0; i < outputLength; i++) {
        connection.write(output[i], outputEncodings[i],
                         outputCallbacks[i]);
      }

      this.output = [];
      this.outputEncodings = [];
      this.outputCallbacks = [];
    }

    // Directly write to socket.
    return connection.write(data, encoding, callback);
  } else if (connection && connection.destroyed) {
    // The socket was destroyed.  If we're still trying to write to it,
    // then we haven't gotten the 'close' event yet.
    return false;
  } else {
    // buffer, as long as we're not destroyed.
    return this._buffer(data, encoding, callback);
  }

}

ServerResponse.prototype._implicitHeader = function() {
  this.writeHead(this.statusCode);
};
</pre>

### res.end
<pre>
OutgoingMessage.prototype.end = function(data, encoding, callback) {
  //一堆参数校验  
  if (typeof data === 'function') {
    callback = data;
    data = null;
  } else if (typeof encoding === 'function') {
    callback = encoding;
    encoding = null;
  }
  if (data && typeof data !== 'string' && !(data instanceof Buffer)) {
    throw new TypeError('first argument must be a string or Buffer');
  }
  if (this.finished) {
    return false;
  }
  var self = this;
  function finish() {
    self.emit('finish');
  }
  if (typeof callback === 'function')
    this.once('finish', callback);
  }
  if (!this._header) {...}
  if (data && !this._hasBody) {
    debug('This type of response MUST NOT have a body. ' +
          'Ignoring data passed to end().');
    data = null;
  }
  if (this.connection && data)
    this.connection.cork();

  if (this._hasBody && this.chunkedEncoding) { //如果是trunk类型，输出一个长度为0的chunk表示结束
    ret = this._send('0\r\n' + this._trailer + '\r\n', 'binary', finish);
  } else {
    // Force a flush, HACK.
    ret = this._send('', 'binary', finish);//输出一个空行
  }
  this.finished = true;
 debug('outgoing message end.');
  if (this.output.length === 0 &&
      this.connection &&
      this.connection._httpMessage === this) {
    this._finish(); //触发结束
  }

  return ret;
</pre>


## 总结
node的http部分业务逻辑非常之多，而且这还是没有窥探到底层libuv代码的情况下，我们简单梳理一下上面出现过的实例：    
1 creatServer时： 创建一个http Server实例server，继承自net.Server。    
2 listen调用时： 创建一个TCP的实例tcp ，让 server._handle = tcp; tcp.owner = server; tcp监听指定端口。同时绑定一个 onread事件。    
3 连接到来时：先触发 tcp.onconnection()传递回来一个参数 clientTCP，在这里会创建一个 Scoket的实例reqSocket，reqSocket._hanlde = clientTCP; clientTCP.owner = reqSocket;    
4 步骤3完成时会触发 reqSocket 的connection方法，在这个方法里面会涉及到 HTTPParse的实例httpParser，这一步是为socket绑定事件和parser的定义。
5 请求数据来到，触发clientTCP的onread事件，onread事件会触发socket的socketOnData事件，而在这个事件里面，会触发 parserOnHeadersComplete 。    
6 执行parserOnHeadersComplete，会创建一个 IncomingMessage 的实例，将parser header的一些结构赋予它， 这个实例就是用户会接触到的 req，parserOnHeadersComplete处理完成之后会触发parserOnIncoming。    
7 parserOnIncoming会创建一个 ServerResponse 实例 res，也就是用户能够接触到的res，然后把req和res作为参数触发 server的request事件（**来自用户**）。   

接触到的实例有：

Http server -> server & net server -> server 用于和用户打交道，传递端口号回调等。   
TCP -> tcp 用于调用native代码监听端口以及接受回调。    
TCP -> clientTCP 用于native代码回调传递客户端相关信息。    
Socket -> reqSocket 用于js方接受clientTCP相关信息。   
HTTPParser -> httpParser 用于解析http相关信息。    
IncomingMessage > req 用户回调中的request对象。
ServerResponse -> res 用户回调中的response对象。
