---
title:  “Nginx配置参数研究”
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
向代理服务器传递request header，请求字段的具体解释可以参考[https://zh.wikipedia.org/wiki/HTTP头字段][http-wiki]

- `proxy_set_header Upgrade $http_upgrade;`：[WebSocket proxying][websocket-nginx]
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

[http-wiki]:https://zh.wikipedia.org/wiki/HTTP%E5%A4%B4%E5%AD%97%E6%AE%B5
[websocket-nginx]:http://nginx.org/en/docs/http/websocket.html
