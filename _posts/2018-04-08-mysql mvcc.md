---
title:  “Mysql MVCC”
mathjax: true
layout: post
date:   2018-04-08 08:00:12 +0800
categories: database
---

之前看的一些mysql的东西，也搬到Github Page上来。

MVCC（Mutiple Version Concurrency Control)，最大的好处是实现非锁定读，从而提高数据库系统的并发性能。与之对应的一个术语是一致性非锁定读（consistent nonblocking read），即InnoDB存储引擎通过MVCC的方式来读取当前时刻数据库中行的数据，如果读取的行正在执行DELETE或UPDATE操作，读取操作不会等待行上锁的释放，而是去读取行的一个快照数据。


![image01]({{site.baseurl}}/image/mysql-nonblocking-read.jpg)

> 有个相似的术语叫半一致读（semi-consistent read)，说的是在READ COMMITTED隔离级别下，一个UPDATE语句，
> 如果读到一条已经加锁的记录，InnoDB返回记录最近提交的版本，由Mysql上层判断是否满足更新条件；如果条件满足，
> Mysql重新发起一次读操作，加锁读取该行的最新版本。

在不同的事务隔离级别下，对于快照的定义并不相同。
- READ COMMITTED  
    READ COMMITTED事务隔离级别下，对于快照数据，非一致性读总是读取被锁定行的最新一份快照数据。这也符合RC隔离的定义，只要提交了的数据都可见。

    Transaction A |  Transaction B
    :----|:----
    begin; |
    select * from user where id = 1; |
    `1 jiangcl F` |
    | begin; |
    | update user set gender = 'M' where id = 1; |
    | commit;
    select * from user where id = 1; |
    `1 jiangcl M` |

- REPEATABLE READ (默认)  
    REPEATABLE READ事务隔离级别下，对于快照数据，非一致性读总是读取事务开始时的行数据版本。根据RR隔离的定义，这样才能保证在一个事务中，所有的相同查询结果一致。

    Transaction A |  Transaction B
    :----|:----
    begin; |
    select * from user where id = 1; |
    `1 jiangcl F` |
    | begin; |
    | update user set gender = 'M' where id = 1; |
    | commit;
    select * from user where id = 1; |
    `1 jiangcl F` |

参考:  
[MySQL技术内幕-InnoDB存储引擎](product.dangdang.com/23255589.html)
