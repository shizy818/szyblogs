---
title:  “WebDriver”
date:   2018-01-29 08:00:12 +0800
categories: testing
---

这个博客的名字叫“那些年跳过的坑”，但是好像很久没说坑了。今天来说说最近在用WebDriver做测试的时候遇到的一个。

# 框架

项目的前端集成测试用到了Behave + WebDriver（不去细分WebDriver和Selenium了，具体差别可以参考[Selenium VS WebDriver](https://www.ibm.com/developerworks/cn/web/1309_fengyq_seleniumvswebdriver/index.html))。
Behave是一个BDD（Behavior Drive Devlopment)框架。行为驱动开发号称用诗一样的语言来写测试行为，我们姑且认为是首
现代打油诗，就像下面这样定义Feature：
```
Feature: DataSource_Actions
  We use this feature to validate datasource.

  Background: Login with specified user
    Then we can login dsx local with configured user

  Scenario: Create "DB2z" datasource for sanity test
    Given we can create and open the project "IML_UI_Sanity666"
    And we can create the datasource "db2z" for project "IML_UI_Sanity666"
```


Feature对应的行为在一个Python文件中。其中Utils封装了配置类和UI操作类；DataSourceUI是数据源的封装类。
```python
from behave import given, when, then, step
from DataSourceUI import UIDataSource
from Utils import iml_ui, iml_config

@given('we can create the datasource "{datasource_name}" for project "{project_name}"')
def step_impl(context, datasource_name, project_name):
    datasource = UIDataSource(context.browser)
    # open datasources page
    datasource.getDataSourcesPage(project_name)
    # create datasource
    assert datasource.createDataSource(datasource_name) is True
```

# Selenium实例

在本地环境下，开启一个Selenium浏览器实例：
```python
from selenium import webdriver
driver = webdriver.Chrome()
```

如果是Remote模式（希望操作的浏览器不一定在本地），需要先用Selenium-Server-Standalone-xxx的jar包启动
一个Selenium Server端http://127.0.0.1:4444/wd/hub，然后开始浏览器实例：
```python
from selenium import webdriver
driver = webdriver.remote.webdriver.WebDriver(command_executor="http://127.0.0.1:4444/wd/hub",desired_capabilities=DesiredCapabilities.CHROME)  
```

我们来看一下selenium/webdriver/chrome/webdriver.py里的初始化代码。executable_path指定了chromedriver的路径，
如果已经通过环境变量加入PATH里，就可以省略executable_path。
```python
class WebDriver(RemoteWebDriver):
    """
    Controls the ChromeDriver and allows you to drive the browser.

    You will need to download the ChromeDriver executable from
    http://chromedriver.storage.googleapis.com/index.html
    """

    def __init__(self, executable_path="chromedriver", port=0,
                 chrome_options=None, service_args=None,
                 desired_capabilities=None, service_log_path=None):
        """
        Creates a new instance of the chrome driver.

        Starts the service and then creates new instance of chrome driver.

        :Args:
         - executable_path - path to the executable. If the default is used it assumes the executable is in the $PATH
         - port - port you would like the service to run, if left as 0, a free port will be found.
         - desired_capabilities: Dictionary object with non-browser specific
           capabilities only, such as "proxy" or "loggingPref".
         - chrome_options: this takes an instance of ChromeOptions
        """
        ......
        self.service = Service(
            executable_path,
            port=port,
            service_args=service_args,
            log_path=service_log_path)
```

举例一个简单的动作，点击`Add Data Source`按键：
```python
addDataSource = driver.find_element_by_xpath("//*[@id=\"addDataSourceButton\"]")
addDataSOurce.click()
```
此处有两个命令，先找到id是addDataSourceButton的DOM元素，然后点击。命令是通过HTTP请求发送到Server端，然后由浏览器驱动程序控制
浏览器上的动作（chromedriver和geckodriver分别对应Chrome和Firefox）。
```
command: findElement
commands[command]: ('POST', '/session/$sessionId/element')
url: http://127.0.0.1:64757/session/b900e0c42d26e868fc065cd5099c6a5b/element
data: {"using": "xpath", "sessionId": "b900e0c42d26e868fc065cd5099c6a5b", "value": "//*[@id=\"addDataSourceButton\"]"}

command: clickElement
commands[command]: ('POST', '/session/$sessionId/element/$id/click')
url: http://127.0.0.1:64757/session/b900e0c42d26e868fc065cd5099c6a5b/element/0.7943016811410035-1/click
data: {"sessionId": "b900e0c42d26e868fc065cd5099c6a5b", "id": "0.7943016811410035-1"}
```

# ElementNotVisibleException

当我的测试程序找到这个addDataSourceButton按键，然后点击的时候，报错ElementNotVisibleException。
```
Traceback (most recent call last):
        File "/Users/shizy/anaconda/lib/python2.7/site-packages/behave/model.py", line 1456, in run
          match.run(runner.context)
        File "/Users/shizy/anaconda/lib/python2.7/site-packages/behave/model.py", line 1903, in run
          self.func(context, *args, **kwargs)
        File "features/steps/DataSource_steps.py", line 17, in step_impl
          assert datasource.createDataSource(datasource_name, "jdbc:db2://9.125.72.72:430/LOCDB11", "tuser01", "c7deshop") is True
        File "./library/DataSourceUI.py", line 83, in createDataSource
          addDataSourceLink.click()
        File "/Users/shizy/anaconda/lib/python2.7/site-packages/selenium/webdriver/remote/webelement.py", line 77, in click
          self._execute(Command.CLICK_ELEMENT)
        File "/Users/shizy/anaconda/lib/python2.7/site-packages/selenium/webdriver/remote/webelement.py", line 493, in _execute
          return self._parent.execute(command, params)
        File "/Users/shizy/anaconda/lib/python2.7/site-packages/selenium/webdriver/remote/webdriver.py", line 256, in execute
          self.error_handler.check_response(response)
        File "/Users/shizy/anaconda/lib/python2.7/site-packages/selenium/webdriver/remote/errorhandler.py", line 194, in check_response
          raise exception_class(message, screen, stacktrace)
      ElementNotVisibleException: Message: element not visible
        (Session info: chrome=64.0.3282.119)
        (Driver info: chromedriver=2.35.528157 (4429ca2590d6988c0745c24c8858745aaaec01ef),platform=Mac OS X 10.12.6 x86_64)
```

可是明明在页面能看见这个按键啊。很愚蠢的查了一个下午，看这个元素有没有hidden属性。最后原因竟然是在其他页面有一个同Id的元素。这个项目是一个
SPA（Single Page Application），WebDriver找到了另外一个addDataSourceButton按键，此时当然是不可见的。
