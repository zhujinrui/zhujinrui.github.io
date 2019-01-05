---
title: '[WebSocket]服务器长连接（一）'
catalog: true
date: 2019-01-05 21:05:27
subtitle:
header-img:
tags:
- WebSocket
catagories:
- WebSocket
---


#### 1.为什么需要WebSocket?
因为HTTP协议有一个缺陷：通信只能由客户端发起。HTTP协议做不到服务器主动向客户端推送信息。

#### 2.之前的技术
##### 2.1 轮询(polling)

> 轮询是指JavaScript启动一个定时器，然后以固定的间隔给服务器发请求，询问服务器是否有新的数据改动。
> 最典型的场景就是聊天室。

**缺点**：这种方法会导致过多不必要的请求，浪费流量和服务器资源。



##### 2.2 Commet
> 又可以分为长轮询和流技术。
> 
`长轮询`改进了上述的轮询技术，减小了无用的请求。它会为某些数据设定过期时间，当数据过期后才会向服务端发送请求；这种机制适合数据的改动不是特别频繁的情况。

`流技术`通常是指客户端使用一个隐藏的窗口与服务端建立一个HTTP长连接，服务端会不断更新连接状态以保持HTTP长连接存活；这样的话，服务端就可以通过这条长连接主动将数据发送给客户端；流技术在大并发环境下，可能会考验到服务端的性能。

**缺点**：以多线程模式运行的服务器会让线程大部分时间处于挂起状态，`极大地浪费服务器资源`。另外，`网关的不可控性`，一个HTTP连接在长时间没有数据传输的情况下，链路上的任何一个网关都可能关闭这个连接，这就要求commet连接必须定期发一些ping数据表示连接“正常工作”。

##### 2.3 总结
这两种技术都是基于请求-应答模式，都不算是真正意义上的实时技术；它们的每一次请求、应答，都浪费了一定流量在相同的头部信息上，并且开发复杂度也较大。


#### 3.WebSocket简介
它是一种协议，诞生在2008年，2011年成为国际标准，所有的浏览器都支持。它是HTML5推出的协议标准。让服务器和浏览器之间可以建立无限制的`全双工通信`。

>WebSocket的工作流程是这样的：浏览器通过JavaScript向服务端发出建立WebSocket连接的请求，在WebSocket连接建立成功后，客户端和服务端就可以通过 TCP连接传输数据。因为WebSocket连接本质上是TCP连接，不需要每次传输都带上重复的头部数据，所以它的数据传输量比轮询和Comet技术小 了很多。


**最大特点**：服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息，是真正的双向平等对话，属于`服务器推送技术`的一种。
![img](https://upload-images.jianshu.io/upload_images/7888316-a7ed4bf431fffcf8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


WebSocket连接必须由浏览器发起，因为请求是一个标准的HTTP请求：
```yml
GET ws://localhost:3000/ws/chat HTTP/1.1
Host: localhost
Upgrade: websocket
Connection: Upgrade
Origin: http://localhost:3000
Sec-WebSocket-Key: client-random-string
Sec-WebSocket-Version: 13
```

该请求和普通的http请求有几点不同：
1.GET请求的地址不是类似`/path/`，而是以`ws://`开头的地址；
2.请求头`Upgrade: websocket`和`Connection: Upgrade`表示这个连接将要被转换为WebSocket连接；
3.`Sec-WebSocket-Key`是用于标识这个连接，并非用于加密数据；
4.`Sec-WebSocket-Version`指定了WebSocket的协议版本。

响应头：
```yml
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: server-random-string
```

###### 3.1为什么Websocket可以实现全双工通信而HTTP协议不行？
实际上HTTP协议是建立在TCP协议之上的，TCP协议本身就实现了全双工通信，但是HTTP的`请求-应答`机制限制了全双工通信。

websocket连接建立以后，其实只是简单规定了一下：接下来，咱们的通信就不使用HTTP协议了，直接互发数据吧。

###### 3.2服务器支持
由于WebSocket是一个协议，服务器具体怎么实现，取决于所用的编程语言和框架本身。

Node.js本身支持的协议有TCP协议和HTTP协议。要支持WebSocket协议，需要对Bode.js提高的HTTPServer做额外的开发。

已经有若干基于Node.js的稳定可靠的WebSocket实现，我们直接用npm安装使用即可。

###### 3.3ws模块

> 在Node.js中，使用最广泛的WebSocket模块是`ws`。

创建一个WebSocket的服务器实例很容易：
```js
var WebSocket = require('ws'); //导入WebSocket模块

var wss = new WebSocket.Server({ port: 8080 }); //引用Server类，并实例化
```
这样我们就在8080端口打开了一个webSocket Server。该实例由变量`wss`引用。

接下来如果有WebSocket请求接入，`wss`对象可以相应`connection`事件来处理这个`WebServer`。
```js
wss.on('connection', function connection(ws) { //connection事件
    console.log('server: receive connection.');
    
    ws.on('message', function incoming(message) {  
        console.log('server: received: %s', message);
    });

    ws.send('world');
});
```

**客户端创建WebSocket连接**

客户端的写法：
```js
 //打开一个webSocket连接
  var ws = new WebSocket('ws://localhost:8080');
  //打开websocket连接后立刻发送一条消息
  ws.onopen = function () {
    console.log('ws onopen');
    ws.send('from client: hello');
  };
  //响应收到的消息
  ws.onmessage = function (e) {
    console.log('ws onmessage');
    console.log('from server: ' + e.data);
  };
  //关闭连接
  ws.onclose = function (evt) {
    console.log("connection closed")
  }
```

在Node环境下，`ws`模块的客户端可以用于测试服务器端的代码，否则每次都必须在浏览器中执行JavaScript代码。



**同源策略**
WebSocket协议本身不支持同源策略，也就是某个地址为`http://a.com`的网页可以通过websocket连接到`ws://b.com`。但是，浏览器会发送`Origin`的HTTP头给服务器，服务器可以根据`Origin`拒绝这个WebSocket请求。所以，是否要求同源要看服务器端如何检查。
![img](https://upload-images.jianshu.io/upload_images/7888316-9457cdd15a361c1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 4.WebSocket API
###### 4.1事件
支持的事件有`open`、`message`、`close`和`error`。注意`message`事件在接受到所有数据时出发。

支持的原生的两个写法：
- onEvent=handler
- addEventListener(event,handler)