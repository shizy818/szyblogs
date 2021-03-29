---
title:  “项目中遇到的各种Timeout问题总结”
mathjax: true
layout: post
date:   2017-03-12 08:00:12 +0800
categories: nodejs express
---

严格意思上说，之前的设计是一个不太好的设计。后端提供的Train API是一个同步的过程，当数据库表比较大
或者Spark环境执行比较慢的时候，训练模型等待时间很长，由此发生各种Timeout问题。整体调用流程如下：

>    Browser   ->   Nginx   ->   Nodejs Server   ->   Service


# Browser Timeout

前端通过XMLHttpRequest异步调用API接口，XMLHttpRequest提供了timeout属性来允许设置请求的超时时间，默认值是0，即不设置超时。
注：部分浏览器不支持设置请求超时。

# Nginx Timeout

在nginx.conf里配置：
{% highlight shell %}
location /registration/initaccount {
        access_by_lua_file /nginx_data/checkjwt.lua;
        proxy_pass https://portal-main-svc.ibm-private-cloud.svc.cluster.local;
        proxy_connect_timeout 120s;
        proxy_read_timeout 120s;
        proxy_send_timeout 120s;
}
{% endhighlight %}

- proxy_connect_timeout: 后端服务器连接的超时时间，发起握手等候响应超时时间
- proxy_read_timeout: 连接成功后等候后端服务器响应时间（也可以说是后端服务器处理请求的时间)，默认60s
- proxy_send_timeout: 后端服务器数据回传时间，就是在规定时间之内后端服务器必须传完所有的数据，默认60s

如果此处发生超时，浏览器会看见504 Gateway Timeout错误。

# Nodejs Timeout

真正的后端服务是在service里，但是Nodejs Server做了一层代理，通过node去调用Restful API。
这样做主要有2个原因，一方面可以进一步做权限控制，错误检查等等；另一方面避免了跨域请求问题。但是也存在2个潜在的坑:
- Nodejs Http Server默认120s超时
- Request (Simplified Http Client)默认有一个不确定时长的超时，官方解释如下:  
> timeout - Integer containing the number of milliseconds to wait for a server to send response headers
> (and start the response body) before aborting the request. Note that if the underlying TCP connection
> cannot be established, the OS-wide TCP connection timeout will overrule the timeout
> option (the default in Linux can be anywhere from 20-120 seconds).

总结：以后应该会将此调用做成一个异步的方式，通过消息返回训练结果。但是在目前同步调用的情况下，需要注意有四个地方都存在超时行为。
