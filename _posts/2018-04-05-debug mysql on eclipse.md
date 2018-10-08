---
title:  “Mysql在Eclipse上调试”
date:   2018-04-05 08:00:12 +0800
categories: database
---

调试Mysql 5.7源码，参考了网上的文章，有些地方设置略有不同。

# **Ubuntu**

## 安装工具

- 通过apt-get安装gcc/g++, make, cmake, bison, libncurses5-dev。
- 下载[Boost C++ Library](www.boost.org)源码(1.59.0)


## 编译源码

目录/home/shi/github目录下克隆[mysql-server](https://github.com/mysql/mysql-server)5.7版本。
```shell
cd /home/shi/github
git clone -b 5.7 https://github.com/mysql/mysql-server.git
```
默认安装地址是/usr/local/mysql，make install的时候没有权限。因此创建目录/home/shi/mysql5.7，指定为安装目录。
```shell
cmake . -DCMAKE_INSTALL_PREFIX=/home/shi/mysql5.7 -DWITH_DEBUG=1 -DWITH_BOOST=/home/shi/boost_1_59_0
make && make install
```
初始化数据库：
```shell
cd /home/shi/mysql5.7
mkdir debug
bin/mysqld --initialize-insecure --basedir=/home/shi/mysql5.7/debug --datadir=/home/shi/mysql5.7/debug/data --lc-messages-dir=/home/shi/mysql5.7/share/english
```

## Eclipse项目
1. 建立一个C++ Empty项目
2. 新建文件夹，引入源代码

    ![image01]({{site.baseurl}}/image/20180405/new_folder.png)

3. 设置编译源目录Build Directory

    ![image02]({{site.baseurl}}/image/20180405/makefile.png)

4. 启动mysqld参数设置

    ![image03]({{site.baseurl}}/image/20180405/debug_main.png)

    ![image04]({{site.baseurl}}/image/20180405/debug_arguments.png)

然后就可以用客户端连接：
```shell
mysql --socket=/tmp/mysql.sock -uroot
```

# **Windows**

- 安装[bison 2.4.1](http://gnuwin32.sourceforge.net/packages/bison.htm)
- 安装[cmake 3.12.0](https://cmake.org/download/)
- 安装[Visual Studio Community 2015](https://download.microsoft.com/download/e/4/c/e4c393a9-8fff-441b-ad3a-3f4040317a1f/vs_community.exe)
- 安装[boost_1_59_0-msvc-14.0-32.exe](https://sourceforge.net/projects/boost/files/boost-binaries/1.59.0/)

## 生成Makefile

打开CMake-gui程序，在参数WITH_BOOST中填写Boost路径`C:/boost_1_59_0`，同时勾选WITH_DEBUG参数；generator选择`Visual Studio 14 2015`。点击generate生成Visual Studio项目。

![image05]({{site.baseurl}}/image/20180405/cmake.png)

## 编译MySQL.sln工程

点击MySQL.sln打开VS 2015项目，Build -> Build Solution。

![image06]({{site.baseurl}}/image/20180405/vs_build.png)

## 初始化

```shell
cd C:\mysql5.7\sql\Debug
mysqld.exe --initialize-insecure --initialize-insecure --basedir=C:\mysql5.7\Debug --datadir=C:\mysql5.7\Debug\data
```

## 启动 & 调试

将mysqld设置为默认启动项目，启动本地windows调试器。

![image07]({{site.baseurl}}/image/20180405/start_project.png)

在另外一个命令行中运行MySQL客户端：
```shell
cd C:\mysql5.7\client\Debug
mysql -uroot
```

参考：  
[DBA的罗浮宫](http://mdba.cn/2013/12/31/%E4%BD%BF%E7%94%A8eclipse%E8%B0%83%E8%AF%95mysql%E6%BA%90%E7%A0%81/)  
[Windows下使用VS2105调试MySQL](http://yanhuqing666.github.io/debug-mysql-with-vs2015)  
[MySQL源码阅读调试](http://yanhuqing666.github.io/debug-mysql-with-vs2015)