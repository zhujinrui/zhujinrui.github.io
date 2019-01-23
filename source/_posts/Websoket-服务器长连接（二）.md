---
title: '[Websoket]服务器长连接（二)'
catalog: true
date: 2019-01-23 16:56:27
subtitle:
header-img:
tags: 
- WebSocket
catagories:
- WebSocket
---

#### 一、背景
##### 1. 目标：浏览器与服务器全双工通信的机制
需要与服务器全双工通信且不需要依赖打开多个 HTTP 连接（例如，使用 XMLHttpRequest 或iframe和长轮询）的基于浏览器应用的提供一种机制。
##### 2.协议：包括一个打开阶段握手、接着是基本消息帧、TCP 之上的分层（layered over TCP）

##### 3. 解决了以下的问题：
> 实现客户端和服务之间双向通信web应用，需要一个滥用的HTTP来轮询服务器进行更新但以不同的 HTTP 调用发生上行通知。

- 服务器被迫为每个客户端使用一些不同的底层 TCP 连接： 一个用于发送 信息到客户端和一个新的用于每个传入消息。
- 线路层协议有较高的开销，因为每个客户端-服务器消息都有一个 HTTP 头信息。
- 客户端脚本被迫维护一个传出的连接到传入的连接的映射来跟踪回复。


##### 4. 应用场景:

游戏、股票行情、同时编辑的多用户应用、直播、即使通讯聊天、服务器端服务以实时暴露的用户接口等

##### 5. 与TCP和HTTP的关系
> WebSocket 协议是一个独立的基于 TCP 的协议。它与 HTTP 唯一的关系是它的握手是由 HTTP 服务器解释为一个Upgrade请求。默认情况下WebSocket使用80或者443（TLS）端口进行连接。
#### 二、再论WebSocket实时技术
##### 1.HTTP的历史
![image.png](https://upload-images.jianshu.io/upload_images/7888316-841d20bd363eacb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1996年 发布了HTTP1.0版本，1999年HTTP1.1,到现在普遍使用的是1.1。 2015年HTTP2.0。HTTP协议经历了17年的发展。
影响一个 HTTP 网络请求的因素主要有两个：带宽和延迟。(浏览器阻塞，同一域名只有4个连接，DNS查询，TCP三次握手)
- 1.1的改进：长连接，在一个TCP连接上可以传送多个HTTP请求和响应，减少了建立和关闭连接的消耗和延迟。
- 2.0：多路复用，二进制，消息头压缩，服务器可以主动推送响应到客户端。


websocket主要通过==http/1.1协议的101状态码==进行握手

##### 2.  Ajax短轮询
最早的一种实时Web应用的方案。客户端以一定的时间间隔服务器发送请求，以频繁请求的方式保持客户端和服务器端的同步。
![image.png](https://upload-images.jianshu.io/upload_images/7888316-9263164b2195db55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

场景再现：
```
客户端：啦啦啦，有没有新信息(Request)
服务端：没有(Response)
客户端：啦啦啦，有没有新信息(Request)
服务端：没有。。(Response)
客户端：啦啦啦，有没有新信息(Request)
服务端：有啦给你（Response)
客户端：啦啦啦，有没有新信息(Request)
服务端：。。。没。。。。没有(Response)
```
代码实现：

```js
<script type="text/javascript">
  var getting = {
  url:'server.php',
  dataType:'json',
  success:function(res) { 
    console.log(res);
  } 
};
//关键在这里，Ajax定时访问服务端，不断获取数据 ，这里是1秒请求一次。
window.setInterval(function(){$.ajax(getting)},1000);
</script>
```
##### 3.  长轮询
对定时轮询的改进和提高，为某些数据设置过期时间。当数据过期后才会向服务端发送请求；这种机制适合数据的改动不是特别频繁的情况。
![image.png](https://upload-images.jianshu.io/upload_images/7888316-242eb979031279c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

场景再现：
```
客户端：啦啦啦，有没有新信息,没有的话就等有了才返回给我吧(Request)
（一段时间之后。。。）
服务端：有消息啦。。来给你(Response)

客户端：啦啦啦，有没有新信息,没有的话就等有了才返回给我吧(Request)
（一段时间之后。。。）
服务端：有消息啦。。来给你(Response)
````
代码实现：

```js
<script type="text/javascript">
  //前端Ajax持续调用服务端，称为Ajax轮询技术
  var getting = {
    url:'server.php',
    dataType:'json',
    success:function(res) {
      console.log(res);
      $.ajax(getting); //关键在这里，回调函数内再次请求Ajax
    }        
    //当请求时间过长（默认为60秒），就再次调用ajax长轮询
    error:function(res){
      $.ajax($getting);
    }
  };
  $.ajax(getting);
</script>  
```

##### 4.  流技术
设置隐藏窗口向服务器发出长连接请求。服务器接收到这个请求后作出回应并不断更新连接状态以保证客户端和服务器端的连接不过期。

![image.png](https://upload-images.jianshu.io/upload_images/7888316-783bacd31712a357.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

iframe流方式是在页面中插入一个隐藏的iframe，利用其src属性在服务器和客户端之间创建一条长链接，服务器向iframe传输数据（通常是HTML，内有负责插入信息的javascript），来实时更新页面。
iframe流方式的优点是浏览器兼容好，Google公司在一些产品中使用了iframe流，如Google Talk。
在第一种方式中，浏览器在收到数据后会直接调用JS回调函数，但是这种方式该如何响应数据呢？可以通过在返回数据中嵌入JS脚本的方式，如“”，服务器端将返回的数据作为回调函数的参数，浏览器在收到数据后就会执行这段JS脚本。

##### 5.存在问题
- 非真正的实时技术
- 客户端和服务器端交互，都是一次HTTP的请求和应答过程，且每次的HTTP请求和应答都带有完整HTTP头信息，增加传输的数据量。
  
`ajax轮询`缺点：
- 导致过多不必要的请求，浪费流量和服务器资源。

`Comet`(包括长轮询和流技术，也称为服务器推送技术)缺点：
- 服务器会让线程大部分时间处于挂起状态，极大地浪费服务器资源，
网关的不可控性，一个HTTP连接在长时间没有数据传输的情况下，链路上的任何一个网关都可能关闭这个连接



#### 三、HTTP和WebSocket
##### 1. WebSocket和HTTP的区别
 http协议是用在应用层的协议，他是基于tcp协议的，http协议建立连接也必须要有三次握手才能发送信息。

http链接分为短连接，长连接，短连接是每次请求都要三次握手才能发送自己的信息。即每一个request对应一个response。长连接是在一定的期限内保持连接。保持TCP连接不断开。客户端与服务器通信，必须要有客户端发起然后服务器返回结果。客户端是主动的，服务器是被动的。 

WebSocket他是为了解决客户端发起多个http请求到服务器资源浏览器必须要经过长时间的轮训问题而生的，他实现了多路复用，他是全双工通信。在webSocket协议下客户端和浏览器可以同时发送信息。

建立了WenSocket之后服务器不必在浏览器发送request请求之后才能发送信息到浏览器。这时的服务器已有主动权想什么时候发就可以发送信息到服务器。而且信息当中不必再带有head的部分信息了与http的长连接通信来说，这种方式，不仅能降低服务器的压力。而且信息当中也减少了部分多余信息。

##### 2.  HTTP的长连接与websocket的持久连接

 HTTP1.1的连接`默认使用长连接`
 
 `HTTP长连接`即在一定的期限内保持连接，客户端会需要在短时间内向服务端请求大量的资源，**保持TCP连接不断开**。客户端与服务器通信，必须要有客户端发起然后服务器返回结果。客户端是主动的，服务器是被动的。
 
 在一个TCP连接上可以传输多个Request/Response消息对，所以本质上还是Request/Response消息对，仍然会造成资源的浪费、实时性不强等问题。
 
 如果不是持续连接，即`短连接`，那么每个资源都要建立一个新的连接，HTTP底层使用的是TCP，那么**每次都要使用三次握手建立TCP连**接，即每一个request对应一个response，将造成极大的资源浪费。

`长轮询`，即客户端发送一个超时时间很长的Request，**服务器hold住这个连接**，在有新数据到达时返回Response。

`websocket的持久连接`
只需建立一次Request/Response消息对，之后都是TCP连接，避免了需要多次建立Request/Response消息对而产生的冗余头部信息。lean

#### 四、WebSocket

HTML5新协议，诞生在2008年，2011年成为国际标准，现代浏览器（IE10+）基本上都已经支持WebSocket。让服务器和浏览器之间可以建立无限制的全双工通信。

是一种双向通信协议，建立连接后，websocket服务器和浏览器都能主动向对方发送和接收数据，数据以帧形式传输。
通过TCP握手连接，连接成功后才能互相通信。
该协议包括两个方面：==握手连接==（handshake)和==数据传输==（date transfer）

##### 1.握手过程
##### 客户端发起的请求头
![image.png](https://upload-images.jianshu.io/upload_images/7888316-85394c54400467c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- GET /chat HTTP/1.1：打开阶段握手，使用http协议。
- Upgrade: websocket，申请协议升级，Upgrade表示需要从HTTP切换到WebSocket协议。
- Sec-WebSocket-Key:：base64随机码，提供基本的防护，比如恶意连接或者无意的连接。
- Sec-WebSocket-Protocol：用户定义的字符串，用来区分相同URL下不同服务所需要的协议。

##### 服务器响应的头字段
```js
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```
- HTTP/1.1  101 Switching Protocols ,响应协议升级，状态码101表示到此完成协议升级，后续的数据按照新的协议来传输。
- Sec-WebSocket-Accept：经过服务器计算，并且加密过后的Sec-WebSocket-Key。

生成算法如下： 
```js
mask = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11 ";
base(sha1 (Sec-WebSocket-Key + mask));
```
分解动作如下：
```js
1. t = "GhlIHNhbXBsZSBub25jZQ==" + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
   -> "GhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
2. s = sha1(t) 
   -> 0xb3 0x7a 0x4f 0x2c 0xc0 0x62 0x4f 0x16 0x90 0xf6 
      0x46 0x06 0xcf 0x38 0x59 0x45 0xb2 0xbe 0xc4 0xea
3. base64(s) 
   -> "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="
```

##### 2.数据通信过程
client数据包的格式如下：
![image.png](https://upload-images.jianshu.io/upload_images/7888316-96bb7d0d5b29c21b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

数据包的长度是固定的。根据payload len 的长度是为126或127是不一样的。

传送的帧类型可以分为两类：`数据帧`和`控制帧`。数据帧可以携带文本数据或二进制数据；控制帧包含关闭帧（Close frame)和Ping/Pong帧。

FIN ：表示该数据帧是否是信息最后一帧的标记符。为1是为最后一帧，为0不是。

RSV1,RSV2,RSV3：为保留帧(每个占1位)，必须是0，除非一个扩展协商为非零值定义的。保留帧

MASK值：从客户端发送的帧必须置为1，从服务端发送的帧置为0。也就是从客户端发往服务器的数据必须掩码处理，从服务器返回的数据不用做处理。

opcode的值：表示数据类型。总共有五种类型。
> 0x01 : 文本数据帧
> 
>0x02：二进制帧
>
>0x08：关闭帧
>
>0x09：Ping帧
>
>0xA：Pong帧

Payload len ：当着7bit数据的长度为126时，增加额外的的==2字节==也表示数据长度，当数据长度为127时，后面的==8个字节==表示数据长度，也就是Extended payload len（扩展数据）的长度

掩码算法：按位做循环异或运算，先对该位的索引取模来获得 Masking-key 中对应的值 x，然后对该位与 x 做异或，从而得到真实的 byte 数据。

**注意**：
掩码的作用并不是
为了防止数据泄密，而是为了防止早期版本的协议中存在的代理缓存污染攻击（proxy cache poisoning attacks）等问题。

##### 3、WebSocket API 的实践

**客户端**
```js
  //打开一个websocket连接
  var ws = new WebSocket('ws//localhost:8080');
  //打开websocket连接后发送一条消息
  ws.onopen = function () {
    console.log('ws onopen');
    ws.send('from client: hello');
  }

  //响应收到的消息
  ws.onmessage = function (e) {
    console.log('ws onmessage');
    console.log('from server:' + e.data);
  } 
  ```
  利用WebSocket原生构造函数打开了一个WebSocket连接。
  
  共有**四个监听函数**:`onopen`、`onmessage`、`onclose`、`onerror`。分别表示：‘连接建立时’，‘收到服务端消息时’，‘连接已关闭时’，‘连接出错时’的监听函数。
  
 WebSocket方法有两个，send()和close()方法。表示`发动消息函数`和`连接主动关闭`。
 
 **属性有：**
 - readyState
>  0：connecting，连接正在进行,但还没建立Websocket。
>
>  1：open，连接已经建立，可以发送消息。
>
>  2：closing，连接正在进行关闭握手。
>
>  3：closed，连接已经关闭或不能打开。
- bufferedAmount
表示属性检查已经进入队列但还未被传输的数据大小

- Protocol 表示可以构建自己服务的协议，用来区分相同URL下不同服务所需要的协议。。
 
**服务端**

用基于Node.js的模块`ws`来搭建websocket服务。

```js
 var WebSocket = require('ws'); //导入websocket模块
 var wss = new WebSocket.Server({
  port: 8080
 }); //引用Sever类，并实例化
 wss.on('connection', function connection(ws) { //connec事件
  console.log(('server: receive connection.'));
  ws.on('message', function icoming(message) {
    console.log('server:receive: %s', message)
  })
 })
```
事件：`connection`、`open`、`message`、 `close`、 `error`、 `ping`、 `pong`。
#### 五、心跳机制
在使用websocket的过程中，有时候会遇到客户端网络断开的情况，而这时候在服务端并没有触发onclose事件。这样会：
- 多余的连接
- 服务端会继续给客户端发送数据，这些数据会丢失。


所以就需要一种机制来检测客户端和服务端是否处于正常连接的状态。心跳检测就是这样的一种机制，一般来说客户端每过一定时间向服务器发送一个数据包，告诉服务器自己还活着。

![image.png](https://upload-images.jianshu.io/upload_images/7888316-1d565ffb2278adb5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

心跳检测，用我自己的话来说，**就是为了检测客户端与服务端的连接是否能存活**

第一，心跳检测包由客户端（浏览器）定时向服务端（后端）发送约定好的消息格式，告诉服务端客户端在线，服务端收到消息后立即返回一个消息，告诉客户端长连接没问题，可以正常使用。

第二，心跳检测也可以用来检测后端是否正常，如果在连接正常的情况下，服务端并未能在设定的时间内返回特定消息，说明可能当前后端异常，当前连接不可用，客户端可以尝试重新建立websocket连接来重试。

实现心跳检测的方法思路算是比较简单，主要是通过定时向服务器send()相关消息，并且定下心跳包的超时时间，当收到服务器返回的消息时，清掉当前心跳计时器以及重连超时的定时器。若服务端未能够及时返回特定消息，超过设定的超时时间时，主动关闭当前websocket连接，并且尝试重新建立新的websocket连接。

以下是基于React项目心跳检测的部分代码。

**客户端**
```react
import React from 'react';
import ReactDOM from 'react-dom';
import { Table, Card } from 'antd';
import './index.css';
import moment from 'moment';
// import moment from 'moment';
// let socket = new WebSocket("ws://localhost:8092/guest");
// let socket;
let wsUrl = "ws://localhost:8092/guest";
let lockReconnect;
let socket;
    //心跳检测
let heartCheck = {
  timeout: 3000, //心跳包超时时间
  timeoutObj: null,
  serverTimeoutObj: null,
  reset: function (){
    clearTimeout(this.timeoutObj);
    clearTimeout(this.serverTimeoutObj);
    this.start();
  },
  start: function(){
    console.log('start');
    let self = this;
    this.timeoutObj = setTimeout(function(){
      //这里发送一个心跳，后端收到后，返回一个心跳消息，
      socket.send("HeartBeat");
      //服务器响应超时，关闭连接
      self.serverTimeoutObj = setTimeout(function() {
        socket.close();
      }, self.timeout);

    }, this.timeout)
  }
}

class App extends React.Component {
  constructor(props){
    super(props);
    this.state={
      name:'',
      time:''
    }
  }
  componentDidMount(){
    this.createWebSocket();
  }
  createWebSocket=()=>{
    try {
      socket = new WebSocket(wsUrl);
      
      this.init();
    } catch(e) {
      console.log('catch');
      this.reconnect(wsUrl);
    }
  }
  init =()=>{
    socket.onclose = closeEvent => {
      console.log("WebSocket closed.");
      this.reconnect(wsUrl);
    }; 
    socket.onerror = errorEvent => {
      console.log("WebSocket error: ", errorEvent);
      this.reconnect(wsUrl);
    };
    socket.onopen = function(openEvent) {
      //心跳检测重置
      heartCheck.start();
      console.log("WebSocket conntected.");
    };
    socket.onmessage = res =>{
      //当服务器返回消息时计时器清零。
      heartCheck.reset();
      let data= JSON.parse(res.data);
      this.setState({
        data
      })
   }
  }

  reconnect=(url)=>{
    if(lockReconnect) {
      return;
    };
    lockReconnect = true;
    //没连接上会一直重连，设置延迟避免请求过多
    let tt;
    clearTimeout(tt);
    tt = setTimeout(()=> {
      this.createWebSocket(url);
      lockReconnect = false;
    }, 4000);
  }
  render(){
    let { data} = this.state;
    const columns = [{
      title: '姓名',
      dataIndex: 'name',
      render: (text, record)=> <span>{`${text}${record.surname}`}</span>,
      key: 'name',
    }, {
      title: '性别',
      dataIndex: 'gender',
      key: 'gender',
      render: (text, record) => <span>{text == 'female'? '女': '男'}</span>
    },{
      title: '国家',
      dataIndex: 'region',
      key: 'region',
    }];
    console.log(data)
    return(
      <div style={{ background: '#ECECEC', padding: '30px'}}>
        <Card  title="WebSocket实践(1)" style={{padding:'24px', width: 700, margin: '0 auto'  }}>
          <Table dataSource={data} columns={columns}  />
        </Card>
      </div>
    )
  }
}

ReactDOM.render(<App />, document.getElementById('root'));
```

**以下是服务端的代码**
```js
var request = require("request");
var WebSocket = require("ws"),
    WebSocketServer = WebSocket.Server,
    wss = new WebSocketServer({
        port: 8092,
        path: "/guest"
    });
 
// 收到来自客户端的连接请求后，开始给客户端推消息
wss.on("connection", function(ws) {
    ws.isAlive = true;
    ws.on('pong', heartbeat); //心跳检测
    ws.on("message", function(message) {
        console.log("received: %s", message);
    });
    sendGuestInfo(ws);
});
 
function sendGuestInfo(ws) {
    request("https://uinames.com/api/?ext && amount=25 &&region=china",
        function(error, response, body) {
            if (!error && response.statusCode === 200) {
                var jsonObject = JSON.parse(body);

                if (ws.readyState === WebSocket.OPEN) {
 
                    // 发，送
                    ws.send(JSON.stringify(jsonObject));
 
                    //用随机来“装”得更像不定时推送一些
                    setTimeout(function() {
                        sendGuestInfo(ws);
                    }, (Math.random() * 5 + 2) * 1000);
                }
            }
        });
}
//心跳检测
heartbeat = ()=>{
    this.isAlive = true;
}

//心跳复活
const interval = setInterval(function ping() {
    wss.clients.forEach(function each(ws) {
      if (ws.isAlive === false) return ws.terminate();
   
      ws.isAlive = false;
      ws.ping('', false, true);
    });
  }, 30000);
```
[完整代码地址](https://github.com/zhujinrui/heartbeat)

#### 六、Socket.io
在NPM网站的WebSockets包排行榜上看出排在前三的是：
- websocket
- ws
- socket.io

![image.png](https://upload-images.jianshu.io/upload_images/7888316-677de63562bf5405.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

WebSocket是HTML5最新提出的规范，虽然主流浏览器都已经支持，但仍然可能有不兼容的情况，为了兼容所有浏览器，给程序员提供一致的编程体验，**SocketIO将WebSocket、AJAX和其它的通信方式全部封装成了统一的通信接口**，也就是说，我们在使用SocketIO时，不用担心兼容问题，底层会自动选用最佳的通信方式。因此说，WebSocket是SocketIO的一个子集。

- 跨终端，可以在任何平台，浏览器、设备上工作。
- 兼容性好，对于不兼容的环境采用降级策略，支持的浏览器最近达IE5.5。

##### 1. socket.io与ws对比
**区别**
- ws不支持浏览器，需要自行封装websocket
- socket.io客户端对浏览器有良好的支持。socket.io客户端封装的websocket请求自带id，服务器可以根据id区分客户端，进行精准推送。

**联系**
- ws和socket.io都可以单独作为websocket服务器，也可以挂到express和koa框架，同时提供web和websocket服务。

以下就是一个socket.io的请求。

![image.png](https://upload-images.jianshu.io/upload_images/7888316-bf191aeba18a5318.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 2. socket.io实践
Socket.io由两部分组成：
- 一个服务端用于集成 (或挂载) 到 Node.JS HTTP 服务器：==socket.io==
- 一个加载到浏览器中的客户端：==socket.io-clien==

**支持的事件有:**
connect，message，disconnect以及==自定义的事件==，还支持==广播消息（broadcast）==，广播消息给除当前客户端之外的所有在线客户端

**支持的方法**
- io.emmit()
- io.close()

**客户端**

```react
import React, { Component } from 'react';
import Card from 'antd/lib/card';
import Table from 'antd/lib/table';
import io from 'socket.io-client';

import './App.css';

const socket = io('http://localhost:4001');

class App extends Component {
  constructor(props){
    super(props);
    this.state={

    }
  }
  componentDidMount(){
    socket.on('news',res=>{ //自定义事件news
      let data= JSON.parse(res);
      console.log(data)
      //连接成功
      // socket.emit('my other event', { my: 'I am user1' });
      this.setState({data: data })
    });
  }
  render() {
    let { data} = this.state;
    const columns = [{
      title: '姓名',
      dataIndex: 'name',
      render: (text, record)=> <span>{`${text}${record.surname}`}</span>,
      key: 'name',
    }, {
      title: '性别',
      dataIndex: 'gender',
      key: 'gender',
      render: (text, record) => <span>{text === 'female'? '女': '男'}</span>
    },{
      title: '国家',
      dataIndex: 'region',
      key: 'region',
    }];
    console.log(data);
    return (
      <div className="App">
        <div style={{ background: '#ECECEC', padding: '30px'}}>
          <Card  title="WebSocket实践(2)" style={{padding:'24px', width: 700, margin: '0 auto'  }}>
            <Table dataSource={data} columns={columns}  />
          </Card>
        </div>
      </div>
    );
  }
}

export default App;

```

**服务端**
```js
const request = require("request");
const express = require('express');
const app = express();
const path = require('path');
const http = require('http').Server(app);
const io = require('socket.io')(http);
const port = 4001;

app.use(express.static(path.join(__dirname, 'public')));

app.get('/', function(req, res) {
    res.sendFile(__dirname + '/public/index.html');
});

app.get('/api', function(req, res) {
    res.send('.');
});

http.listen(port, function() {
    console.log(`listening on port:${port}`);
});

function sendGuestInfo(socket) {
  request("https://uinames.com/api/?ext && amount=25 &&region=china",
    function(error, response, body) {
      if (!error && response.statusCode === 200) {
        var jsonObject = JSON.parse(body);

        if (socket.readyState === io.OPEN) {

          // 发，送
          socket.emit('news',JSON.stringify(jsonObject));

          //用随机来“装”得更像不定时推送一些
          setTimeout(function() {
              sendGuestInfo(socket);
          }, (Math.random() * 5 + 2) * 1000);
        }
      }
    });
}
io.on('connection', function (socket) {
   sendGuestInfo(socket);
    //发送消息给客户端
    // socket.emit('news', { hello: 'world' });
    // socket.on('my other event', function (data) {
    //     console.log(data);
    // });
    //广播信息给除当前用户之外的用户
    socket.broadcast.emit('user connected');
    //广播给全体客户端
    io.sockets.emit('all users');
});
```

[代码地址](https://github.com/zhujinrui/socket.io)

参考：
- [WebSocket 协议介绍及 WebSocket API 应用](https://www.css88.com/archives/9293)
- [黑客是如何攻击 WebSockets 和 Socket.io的](https://xz.aliyun.com/t/2572)
- [Websocket协议](https://cnodejs.org/topic/59b8cc173c896622428ec6b1)
- [socket.io官网](https://socket.io/docs/#Using-with-Express)
