---
title:  “Mysql在Eclipse上调试”
date:   2018-04-05 08:00:12 +0800
categories: database
---

调试Mysql 5.7源码，参考了[DBA的罗浮宫](http://mdba.cn/2013/12/31/%E4%BD%BF%E7%94%A8eclipse%E8%B0%83%E8%AF%95mysql%E6%BA%90%E7%A0%81/)
的文章，不过有些地方设置不同。

# **Ubuntu**

## 安装工具

- 通过apt-get安装gcc/g++, make, cmake, bison。
- 下载[Boost C++ Library](www.boost.org)源码(1.59.0)


## 编译源码

下载[mysql-server](https://github.com/mysql/mysql-server)，切换至5.7版本。
默认安装地址是/usr/local/mysql，make install的时候没有权限。因此创建目录/home/shi/mysql5.7，
指定为安装目录。
```shell
cmake . -DCMAKE_INSTALL_PREFIX=/home/shi/mysql5.7 -DWITH_DEBUG=1 -DWITH_BOOST=/home/shi/boost_1_59_0
make && make install
```

```
cd /home/shi/mysql5.7
bin/mysqld --initialize-insecure --defaults-file=/home/shi/mysql5.7/my.cnf --datadir=/home/shi/mysql5.7/data --basedir=/home/shi/mysql5.7 --log_error=/home/shi/mysql5.7/error.log
```
