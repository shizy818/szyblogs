---
title:  “Nginx配置参数研究”
mathjax: true
layout: post
date:   2017-08-01 08:00:12 +0800
categories: nginx
---

之前看了一下nginx的timeout配置，最近又把项目中用到的几个配置项整理一下，希望对以后nginx的使用有所帮助。

# proxy_pass & proxy_redirect
- proxy_pass: 充当代理服务器，转发请求

需要注意一下，如果proxy_pass最后带'/'，表示绝对路径。举个例子：
{% highlight shell %}
location /proxy/ {
    proxy_pass http://9.30.57.48:9080;
}
{% endhighlight %}


此时如果访问`http://xxx.xxx.xxx.xxx/proxy/test.html`，会被代理到`http://9.30.57.48:9080/proxy/test.html`；
而如果将代理服务器改为`proxy_pass http://9.30.57.48:9080/`，则会被代理到`http://9.30.57.48:9080/test.html`。

- proxy_redirect: 修改301或者302转发过程中的Location

同样举个例子：  
在9.30.57.48节点运行Node Server，端口9080。
{% highlight javascript %}
var express = require('express');
var app = express();

app.use(express.static('public'));

app.get('/test', function (req, res) {
  res.location('http://9.30.57.48:9080/test.html');
  res.sendStatus(302);
});

var server = app.listen(9080, function () {
  var host = server.address().address;
  var port = server.address().port;

  console.log('Example app listening at http://%s:%s', host, port);
});
{% endhighlight %}

在9.30.123.96节点运行nginx
{% highlight shell %}
http {
        server {
                listen 8080;   ## for easier diag , ruling out ssl problems
                server_name 9.30.123.96;

                location / {
                    proxy_pass http://9.30.57.48:9080;
                    proxy_redirect off;
               }

        }
}
{% endhighlight %}

此时访问`http://9.30.123.96:8080/test`，浏览器显示跳转地址是`http://9.30.57.48:9080/test.html`，即Response Header里Location地址暴露给了浏览器。
解决的方法是设置`proxy_redirect default`(等同于`proxy_redirect http://9.30.57.48:9080/ /`)。再次输入`http://9.30.123.96:8080/test`，
浏览器跳转地址显示`http://9.30.123.96:8080/test.html`。

# proxy_set_header (Host & Upgrade & Connection)
向代理服务器传递request header，请求字段的具体解释可以参考[简书HTTP头字段](https://www.jianshu.com/p/6e86903d74f7)

- `proxy_set_header Upgrade $http_upgrade;`：[WebSocket proxying](http://nginx.org/en/docs/http/websocket.html)
- `proxy_set_header Connection "upgrade";`：WebSocket proxying
- `proxy_set_header Host $host;`：传递服务器的域名和端口号

# sub_filter & sub_filter_once & sub_filter_types
sub_filter模块用于替换Response中的字符串。

- `sub_filter "ibm-nginx-svc" $host;`：把返回中的ibm-nginx-svc替换为请求中的HOST地址
- `sub_filter_once off`：替换所以符合条件的地方
- `sub_filter_types`：指定除text/html之外的MIME type。

# proxy_http_version
代理Http协议版本，默认1.0。推荐使用1.1以便于keepalive Connection

# rewrite
改写请求地址，例如：
{% highlight shell %}
location /name/ {
    rewrite    /name/([^/]+) /users?name=$1 break;
    proxy_pass http://127.0.0.1;
}
{% endhighlight %}

# buffering

缓存主要是合理设置缓冲区大小，尽量避免缓冲到磁盘的情况：
- proxy_buffering

    默认开启，nginx会将后端响应内容先放到缓冲区，缓冲区大小由proxy_buffer_size和proxy_buffers指令决定。如果响应内容超出了缓冲区大小，则部分内容会被写到磁盘上的临时文件。临时文件由proxy_max_temp_file_size和proxy_temp_file_write_size这两个指令决定的。

    如果关闭proxy_buffering，nginx会立即把收到的响应传送给客户端，每次从后端读取的大小为proxy_buffer_size，这样效率比较低。

    当proxy_buffering设置为off，proxy_buffers和proxy_busy_buffers_size将会失效；但无论proxy_buffering是否开启，proxy_buffer_size都是生效的。

- proxy_buffer_size

    用来接受后端响应的第一部分，通常是响应头（Header）。默认大小是一个内存页，根据系统的不同可能是4K或者8K。

- proxy_buffers

    对于每个客户端连接，设置后端响应内容的缓冲区大小，总大小为number*size，size大小由系统内存页面大小决定；默认值是8*size。

- proxy_busy_buffers_size

    nginx会在没有完全读完后端响应的时候就开始向客户端传送数据，默认会划出2个内存页大小的缓冲区向客户端传送数据，然后继续从后端读取数据，缓冲区满了之后写到磁盘的临时文件。
