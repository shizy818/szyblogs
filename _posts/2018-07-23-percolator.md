---
title:  “谷歌Percolator”
mathjax: true
layout: post
date:   2018-07-23 08:00:12 +0800
categories: machine learning
---

LevelDB是一个LSM（Log Structured Merge）策略的Key-Value数据库。LevelDb性能非常突出，官方网站报道其随机写性能达到40万条记录每秒，而随机读性能达到6万条记录每秒。


LeveDB有放大问题：
- 写放大： 实际写入磁盘的数据大小/用户期望写入的数据量。
    假如，SSTable文件每一层的大小是上一层的10倍，当把i-1层中的一个文件合并到i层中时，LevelDB 需要读取i层中的文件的数量多达10个，排序后再将他们写回到i层中去。所以这个时候的写放大是10。对于一个很大的数据集，生成一个新的table文件可能会导致L0-L6中相邻层之间发生合并操作，这个时候的写放大就是50（L1-L6中每一层是10）
- 读放大： 一次指定查询所需的IO操作数目
    查找一个 key-value 对时，LevelDB 可能需要在多个层中去查找。在最坏的情况下，LevelDB 在 L0 中需要查找8个文件，在 L1-L6 每层中需要查找1个文件，累计就需要查找14个文件。

LevelDB没有锁和事务的概念，只是高效的读写。如果有很强的数据一致性要求，可以在上层实现Percolator事务模型。Percolator是一个二阶段提交，通过”快照隔离“实现符合ACID的跨行跨表事务。

论文中经典的银行转账案例如下：

1. 初始状态，Joe有2美金，Bob有10美金。5和6表示时间戳版本。

    ![image01]({{site.baseurl}}/image/20180723/percolator_init.png)

2. 开始事务，写入主锁信息锁住Bob的账户

    ![image02]({{site.baseurl}}/image/20180723/percolator_primary_lock.png)

3. 写入副锁，更新Joe账户

    ![image03]({{site.baseurl}}/image/20180723/percolator_sec_lock.png)

4. 提交点，清除主锁，写入一条新记录（提交时间戳：8），write列指向存储Bob账户数据的时间戳。只要primary的行锁去掉，就表示该事务已经成功提交。secondary的commit是可以异步进行的，在异步提交进行的过程中，如果此时有读请求，可能会需要做一下锁的清理工作。

    ![image04]({{site.baseurl}}/image/20180723/percolator_commit_1.png)

5. 删除副锁信息，写入一条新记录，write列指向存储Joe账户数据的时间戳。

    ![image05]({{site.baseurl}}/image/20180723/percolator_commit_2.png)

参考：  
[LevelDB实现原理](http://www.cnblogs.com/haippy/archive/2011/12/04/2276064.html)  
[Percolator和TiDB事务算法](https://pingcap.com/blog-cn/percolator-and-txn/)