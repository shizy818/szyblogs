---
title:  “Reactjs e2e testing with protractor”
date:   2017-08-19 08:00:12 +0800
categories: Reactjs testing
---

公司Reactjs项目没有做成普通的SPA(Single Page Application)，而是Sever Rendering方式。每次浏览器请求或刷新url，
从后台生成新的html页面返回。因为页面包含React Component，似乎也有人称这种方式为Isomorphic app。
可以参考国外的一个blog [http://jeffhandley.github.io/QuickReactions/][Isomorphic-intro]

另一方面，在微服务大行其道的环境下，架构也对整个UI项目拆分如下。  
>        project-common  <-   UI portal-A   ->   api-service-A  
>              ^  
>              |-----------   UI portal-B   ->   api-service-B

project-common生成导航栏部分和登录用户信息；api-service-xx提供portal-A所需的数据以及各种Restful请求；
portal-xx通过dust模板和Reactjs生成HTML页面。看起来一切都好，然而在e2e测试时遇到一些麻烦；理想的情况下，
portal-xx的测试应该不依赖于api-service或者其他的项目。对于SPA程序，HTMT/js/css在第一次请求时返回，
然后浏览器端发送API请求获得数据并更新页面，因此可以通过一些模块例如[protractor-http-mock][protractor-http-mock]
或者[nock][nock]模拟API请求返回。而sever rendering是在后端发出的Restful请求，数据用于dust模板，
实验发现protractor-http-mock或者nock都无法模拟API请求返回。

个人的解法是在gulp文件里，启动一个Express App来满足protal-xx项目的API请求。


# Gulpfile.js
selenium-standalone用于安装chromeDriver(手动命令node_modules/protractor/bin/webdriver-manager update)
和启动selenium server http://127.0.0.1:4444/wd/hub (手动命令node_modules/protractor/bin/webdriver-manager start)。
定义e2etests:server在e2etests:api启动之后；test脚本在e2etests:selenium和e2etests:server启动完成后执行。


# e2e test scripts
用protractor测试非Angular应用，需要设置browser.ignoreSynchronization = true，关闭protractor等待Angular动作。
{% highlight javascript %}
// test/e2e/protractor.conf.js
exports.config = {
    framework: 'jasmine',
    // The file path to the selenium server jar ()
    seleniumAddress: 'http://localhost:4444/wd/hub',
    specs: ['.js'],
    capabilities: {
        browserName: 'chrome'
    },
    baseUrl: 'http://localhost:4850',
    onPrepare: function() {
      browser.ignoreSynchronization = true;
    },
    mochaOpts: {
      enableTimeouts: true,
    },
    allScriptsTimeout: 15000,
};
{% endhighlight %}

model-builder.js是一个很简单的例子，仅仅是测试页面的Title是否符合预期。
{% highlight javascript %}
// test/e2e/model-builder.js
describe('e2e test', function() {

  beforeAll(function(){
  });

  it('test title', function() {
      // Can be considered to be beforeAll, which Protractor lacks.
      browser.get('/ml/model-builders/new-model-builder?projectid=c341d83f-c302-4846-a99d-0636d5af975a&context=analytics');
      // browser.sleep(5000);
      expect(browser.getTitle()).toEqual('xxx Title');
  });
});
{% endhighlight %}

[Isomorphic-intro]:http://jeffhandley.github.io/QuickReactions/
[protractor-http-mock]:[https://github.com/atecarlos/protractor-http-mock]
[nock]:[https://github.com/node-nock/nock]
