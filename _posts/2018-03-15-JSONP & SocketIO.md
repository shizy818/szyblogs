---
title:  “JSONP & SocketIO”
date:   2018-03-15 08:00:12 +0800
categories: frontend
---

# JSONP

JSONP的引入主要是解决浏览器跨域问题。同源要求域名，协议和端口都相同。同源策略下，某个服务器下的
页面是无法通过REST API直接获取该服务器以外的数据。但是img src(图片), link href(css)和
script src(javascript)不受此限制。

因此JSONP就是利用了script来实现跨域获取JSON数据。简单的说就是：1）动态创建script标签，
利用src来跨域获取数据；2）创建一个回调函数，在远程服务上调用这个函数，将JSON数据作为参数传递。
```html
<script>
  function showData (result) {
    var data = JSON.stringify(result); //json对象转成字符串
    $("#text").val(data);
  }

  $(document).ready(function () {
    $("#btn").click(function () {
      //向头部输入一个脚本，该脚本发起一个跨域请求
      $("head").append("<script src='http://localhost:9090/student?callback=showData'><\/script>");
    });
  });
</script>
```


服务器端对/student?callback=showData的返回如下。前端收到showData(result)形式，因为返回是script，
所以自动调用showData函数。
```java
//前端传过来的回调函数名称
String callback = request.getParameter("callback");
//用回调函数名称包裹返回数据，这样，返回数据就作为回调函数的参数传回去了
result = callback + "(" + result + ")";
```

在Express框架里, 封装了res.jsonp方法:
```javascript
res.jsonp = function jsonp(obj) {
  var val = obj;
  ......

  // jsonp
  if (typeof callback === 'string' && callback.length !== 0) {
    this.set('X-Content-Type-Options', 'nosniff');
    this.set('Content-Type', 'text/javascript');

    // restrict callback charset
    callback = callback.replace(/[^\[\]\w$.]/g, '');

    // replace chars not allowed in JavaScript that are in JSON
    body = body
      .replace(/\u2028/g, '\\u2028')
      .replace(/\u2029/g, '\\u2029');

    // the /**/ is a specific security mitigation for "Rosetta Flash JSONP abuse"
    // the typeof check is just to reduce client error noise
    body = '/**/ typeof ' + callback + ' === \'function\' && ' + callback + '(' + body + ');';
  }

  return this.send(body);
}
```


最后说一下jquery的JSONP方式：
```javascript
// 第一种方式指定dataType: 'jsonp'。请求自动带了callback=xxx,
// xxx是jquery随机生成的回调函数名。
$.ajax({
  url: "http://localhost:9090/student",
  type: "GET",
  dataType: "jsonp", //指定服务器返回的数据类型
  success: function (data) {
    var result = JSON.stringify(data); //json对象转成字符串
    $("#text").val(result);
  }
});

//或者第二种方式
$.getJSON("http://localhost:9090/student?callback=?",
  function(data) {
    var result = JSON.stringify(data); //json对象转成字符串
    $("#text").val(result);
  });
```

# socket.io

Socket.io用于在浏览器和服务器端建立双向通信。

在服务器端，初始化连接，连接成功后发送一个news事件，监听my other event事件:
```javascript
io.on('connection', function (socket) {
  socket.emit('news', { hello: 'world' });
  socket.on('my other event', function (data) {
    console.log(data);
  });
});
```

浏览器端，连接服务器后监听news事件，然后发送my other event事件:
```html
<script src="/socket.io/socket.io.js"></script>
<script>
  var socket = io.connect('http://localhost');
  socket.on('news', function (data) {
    console.log(data);
    socket.emit('my other event', { my: 'data' });
  });
</script>
```

请求过程如下：
1. GET http://localhost/socket.io/?EIO=3&transport=polling&t=M8lj44N

    浏览器通过一个XHR，告诉服务器开始长轮询。
    返回`0{"sid":"yNUc2XFUWvLpuP1EAAAI","upgrades":["websocket"],"pingInterval":25000,"pingTimeout":60000}`。
    sid是本次Socket会话ID；upgrades字段表示可以把连接方式从长轮询升级到WebSocket；数字0表示open。

2. GET http://localhost/socket.io/?EIO=3&transport=polling&t=M8lj44n&sid=yNUc2XFUWvLpuP1EAAAI

    第一条数据收发，是通过长轮询实现的。Polling指前端发送一个request，服务器端等到有数据时返回response；前端
    收到response后马上发送下一条request，从而实现双向通信。请求收到返回`42["news",{"hello":"world"}]`。
    42表示消息类型。

3. POST http://localhost/socket.io/?EIO=3&transport=polling&t=M8lj450&sid=yNUc2XFUWvLpuP1EAAAI

    前端收到news后，发送`42["my other event",{"my":"data"}]`；返回ok。

4. GET http://localhost/socket.io/?EIO=3&transport=polling&t=M8lj450.0&sid=yNUc2XFUWvLpuP1EAAAI

    返回6，服务器端没有新的数据。

5. GET ws://localhost/socket.io/?EIO=3&transport=websocket&sid=yNUc2XFUWvLpuP1EAAAI

    升级到WebSocket连接，然后通过WebSocket往服务器发一条内容为2probe, 数字2表示ping数据。如果这时服务器返回了
    内容为3probe, 3表示pong数据，前端就会把前面建立的HTTP长轮询停掉，后面只使用WebSocket通道进行收发数据。
    数字5表示upgrade，然后websocket会收到一个“40”消息，40代表连接功能了，可以进行通信了。

    ![image01]({{site.baseurl}}/image/ping-pong.png)

# socket.io-redis

对于多个Nodejs instance的情况，需要借助[socket.io-redis](https://github.com/socketio/socket.io-redis)适配器，来保证消息广播和发送能够相互到达。
