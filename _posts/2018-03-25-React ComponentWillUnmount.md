---
title:  “React ComponentWillMount & ComponentWillUnmount”
mathjax: true
layout: post
date:   2018-03-25 08:00:12 +0800
categories: frontend
---

# React生命周期

React Component的生命周期包括
- Mounting：已插入真实DOM (ComponentWillMount -> render -> ComponentDidMount)
- Updating：正在被重新渲染 (ComponentWillReceiveProps -> shouldComponentUpdate -> ComponentWillUpdate -> render -> ComponentDidUpdate)
- Unmounting：已移出真实DOM (ComponentWillUnmount)


# ComponentWillUnmount + browserHistory.goback()

编程细节上遇到一个坑。我们将一些清理工作放在了函数ComponentWillUnmount里。比如点击`/detail`页面里的Back按键，调用
browserHistory.goback()，浏览器回到上一个页面`/list`，于是触发ComponentWillUnmount。正常情况下是OK的，
然而如果用户并不是从`/list`页面的链接进入`/detail`页面，而是直接在浏览器输入`/detail`, browserHistory.goback()
回到的并不是`/list`页面，而是浏览器之前停留的页面比如百度首页。我的理解是此时React App直接被销毁了，而没有机会去
调用ComponentWillUnmount；浏览器现在打开的一个新窗口（window）。

一个解决方法是，在窗口上侦听unload事件，触发Cleanup函数（即使关闭浏览器也有效，但是别使用回调函数)。同时
CompoentWillUnmount也调用Cleanup函数。
```javascript
CompopnentWillMount() {
  window.addEventListener('beforeunload', this.handleCleanup);
}

ComponentWillUnmount() {
  this.handleCleanup();
}

handleCleanup() {
  // your work here
  window.removeEventListener('beforeunload', this.handleCleanup);
}
```

然而或许这样还是无法保证清理工作一定能够触发，比如浏览器奔溃的情况；此时可以通过服务器端定期检查来做清理工作。

# ComponentWillMount + Server Rendering

有一个问题是在哪个函数里向后端抓取数据：ComponentWillMount还是ComponentDidMount? 结论是一般在ComponentDidMount中负责抓取数据，因为如果采用Server Rendering方式，ComponentWillMount会在后端和前端执行两次，一般都不需要这样。

参考：  
[Where to Fetch Data: componentWillMount vs componentDidMount](https://daveceddia.com/where-fetch-data-componentwillmount-vs-componentdidmount/)
