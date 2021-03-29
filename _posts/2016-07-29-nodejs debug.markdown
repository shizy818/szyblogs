---
title:  "nodejs debug"
mathjax: true
layout: post
date:   2016-07-29 12:00:12 +0800
categories: javascript
---

大部分时间直接用`console.log`或者`logger.info`调试代码，但是还是有时候需要跟步调试。公司没有钱，
申请不了Webstorm这种高大上的IDE，只好研究一下有啥免费的调试工具。

一搜网上一大堆文章，选择尝试了一下Chrome浏览器的调试器和Atom node-debugger插件，都还不错。


# Chrome浏览器调试

1) 首先我们需要通过npm安装node-inspector
`cd /usr/local`
`sudo npm install -g node-inspector  // -g 导入安装路径到环境变量`

2) node-inspector是通过websocket方式转向debug输入输出的。因此我们调试前启动node-inspector监听nodejs的debug调试端口  
![image01]({{site.baseurl}}/image/node-inspector.png)

3) 启动node-inspector之后，通过--debug来启动nodejs程序  
![image02]({{site.baseurl}}/image/node-debug.png)

4) 在Chrome浏览器中打开调试地址，会自动加载nodejs源码；找到需要调试js文件，添加断点，然后在另一窗口或Terminal终端发送Http请求，进入调试模式  
![image03]({{site.baseurl}}/image/chrome-debug.png)

# Atom插件调试

Atom IDE的使用还在探索阶段，当然整体界面还是比较清爽的。之前在用的插件有2个：`atom-beautify`用于代码自动格式化，`git-plus`用于git版本控制。

1) 在Packages -> Settings View -> Manage Packages中找到`node-debugger`,设置Node Path  
![image04]({{site.baseurl}}/image/atom-setting.png)

2) 通过--debug启动nodejs程序  
![image05]({{site.baseurl}}/image/node-debug.png)

3) 加入断点，F5启动node-debugger插件  
![image06]({{site.baseurl}}/image/atom-debug.png)

4) 通过浏览器或Terminal终端发送Http请求，进入调试模式

参考：  
[http://www.cnblogs.com/moonz-wu/archive/2012/01/15/2322120.html][chrome-debug-link]  
[https://atom.io/packages/node-debugger][node-debugger-link]  

[chrome-debug-link]:http://www.cnblogs.com/moonz-wu/archive/2012/01/15/2322120.html
[node-debugger-link]:https://atom.io/packages/node-debugger
