---
title:  “DB2 Data Sharing vs Oracle RAC”
date:   2018-01-07 08:00:12 +0800
categories: database
---

首先声明一下，Data Sharing是DB2在大机上的说法，在LUW(Linux/Unix/Windows)叫做pureScale。
pureScale的特性之一是DPF（Data Partitioning Feature），这个比较类似其他数据库上的Data Sharding，
是“存储”上的分布式。Data Sharing没有DPF，大机上的存储是集中式的，可以主从备份。Data Sharing
可以理解为“计算”上的分布式，客户端通过负载均衡发送请求到不同的节点（DB2 Member）上执行SQL语句。
DB2 Member可以横向扩展，因此提高了TPS（Transaction Per Second）。


DB2/Oracle很重要的客户之一是银行保险OLTP业务，对于数据一致性的要求非常高。数据不一致可能出现
如下情况：
* 脏读： 事务A访问修改数据，修改没有提交；但另一事务B可以看到事务A的修改结果。
* 不可重复读：在事务A提交查询结果之前，允许其他事务修改和删除事务A涉及的数据；事务A执行同样操作的结果集变小。
* 幻读：在事务A提交查询结果之前，允许其他事务插入数据；事务A执行同样的操作的结果集增大。

”数据一致“和系统”并发能力“是对立的。在某些情况下，为了获得更大的“并发能力”，需要牺牲部分的“数据一致性”。
SQL标准中定义了4种”数据一致级别“也叫做”隔离级别“。

隔离级别 | 脏读 | 不可重复读 | 幻读
:-:|:-:|:-:|:-:
Read Uncommitted | Y | Y | Y
Read Committed | N | Y | Y
Repeatable | N | N | Y
Serializable | N | N | N

DB2中命名稍稍有点区别：

隔离级别 | 脏读 | 不可重复读 | 幻读
:-:|:-:|:-:|:-:
Uncommitted Read | Y | Y | Y
Cursor Stability | N | Y | Y
Read Stability | N | N | Y
Repeatable Read | N | N | N

Read Uncommitted不需要对数据上锁，读到脏数据也无所谓，但在OLTP业务中很少使用。其他情况下，
在多个节点环境中，当一个节点对数据修改，需要通知其他节点该数据已经修改并及时更新。下面我们分别
看看DB2 Data Sharing和Oracle RAC的做法。

# DB2 Data Sharing

其本质上是通过一个共享空间叫GBP（Global Buffer Pool）来维护一个数据页列表，所有超过一个
节点有访问需要的数据页都需要上传到GBP，并由GBP负责定时刷新到磁盘里。一个典型的流程如下图：  
![image01]({{site.baseurl}}/image/db2-datasharing.png)

1. DB2A节点的Transaction 1对Page上锁，然后从磁盘将Page读取到本地Buffer Pool，最后释放Page锁。
2. DB2B节点的Transaction 2对Page上锁，然后从磁盘将Page读取到本地Buffer Pool，
3. DB2B节点的Transaction 2修改Buffer Pool里的Page
4. 事务提交阶段，DB2B节点将Page上传至GBP，然后释放页锁
5. GBP通知DB2A节点，其本地Buffer Pool的Page已经失效
6. DB2A节点的Transaction 3对Page上锁，然后发现本地Buffer Pool已经失效，因此从GBP请求更新Page

# Oracle RAC

Oracle引入了一个机制叫分布式锁管理器(Distributed Lock Management)。可以把DLM想象成一个“仲裁”，
它记录着哪个节点用哪种方式操作哪个数据，并负责协调解决节点间的竞争。DLM提供两种锁：PCM Lock和Non-PCM
Lock，统称Global Lock，其中PCM Lock用于保护数据块。

数据块拷贝在集群节点间的状态分布图，是通过GRD（Global Resource Directory）实现的。每个节点的
实例中都有部分GRD，所有实例的GRD汇总在一起才是一个完整的GRD。RAC会根据每个资源的名称从集群中选择
一个节点作为它的Master Node，而其他节点叫做Shadow Node。Master Node的GRD记录了该资源在所有
节点上的使用信息，而Shadow Node的GRD中只需要记录资源在该节点上的使用信息，这些信息实际就是PCM Lock
信息。

下面来看看Oracle RAC的读写流程：  
![image02]({{site.baseurl}}/image/oracle-read.png)

1. 实例1计算数据库的Master Node是实例2，于是向实例2发出请求，请求数据块，以及PCM锁。
2. 实例2查找自己的GRD，发现数据块没有被任何实例使用，同意实例1请求。
3. 实例1从本地读取数据块到Buffer Pool，获得数据块PCM Lock（share-mode，local role）。
4. 实例2中更新GRD。

![image03]({{site.baseurl}}/image/oracle-write.png)

1. 实例3要修改这个数据块，向实例2请求exclusive-mode PCM锁。
2. 实例2检查GRD，发现实例1以share-mode持有数据块，于是向实例1发出请求，要求其Mode转换
成null-mode，意味着实例1可以释放这块数据所占有的内存空间。
3. 实例1向实例3发送数据块，并释放自己的PCM锁。
4. 实例3获得exclusive-mode PCM锁；通知实例2，更新GRD
5. 实例2更新GRD，删除实例1持有数据块的记录。

无论是DB2 Data Sharing的“共享内容”方式或者Oracle RAC的“消息传递”方式，都实现了数据库的
横向扩展。但是似乎故障切换(failover)都是痛点，DB2 DS中的GBP或者Oracle RAC中的某个节点
发生了故障，都会造成整个系统的不可用。
