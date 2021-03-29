---
title:  “Pool设计”
mathjax: true
layout: post
date:   2018-09-10 08:00:12 +0800
categories: java
---

本来在考虑为Jupyter Kernel的管理实现一个缓冲池（Pool），所以参考了一下网上缓冲池的设计和实现。某些对象的初始化工作（比如数据库连接）包含费时的操作，因此大量生成这样的对象时，会对性能造成不可忽略的影响。一个有效的减少对象创建次数的技巧就是缓冲池技术。

一个简单的实现可以参考[A Generic and Concurrent Object Pool](https://dzone.com/articles/generic-and-concurrent-object)。设计如下：


![image01]({{site.baseurl}}/image/20180910/simple_pool.png)

- 接口Pool定义了get/release/shutdown方法。get方法取出缓冲池中的一个对象；release方法释放对象并放回缓冲池中。
- 接口Pool包含一个静态Validator接口，JDBCConnectionValidator是一个实现类。
- 抽象类AbstractPool实现标准release流程。
- BoundedBlockingPool和BoundedPool分别继承AbstractPool类和BlockingPool接口。前者使用BlockingQueue作为缓冲池中的队列；后者使用Queue，所以是非阻塞的。
- ObjectFactory是一个工厂模式，JDBCConnectionFactory作为实现类可以生成JDBC Connection对象。

产品级别的对象池化技术（Object Pooling）参考[Apache Common Pool](http://commons.apache.org/proper/commons-pool/)。数据库连接池[Apache Common DBCP](https://commons.apache.org/proper/commons-dbcp/)中即依赖于Apache的common-pool2。

common-pool2中的重要接口:
- PoolObject（池对象）

    封装连接对象（如数据库连接，TCP连接），将其包裹成缓冲池可管理对象。方法包括
    * getCreatedTime: 获取PooledObject创建时间
    * getActiveTimeMillis: 获取PooledObject处于激活状态的时间
    * getIdleTimeMillis: 获取PooledObject处于空闲状态的时间
    * getLastBorrowTime: 获取PooledObject最近借出时间
    * getLastReturnTime: 获取PooledObject最近归还时间
    * getLastUsedTime: 获取PooledObject最近使用时间

    两个默认的池对象实现
    * DefaultPoolObject
    * PooledSoftReference

    ![image02]({{site.baseurl}}/image/20180910/pooledobject.png)

- PooledObjectFactory/KeyedPooledObjectFactory（池对象工厂）

    产生PooledObject的工厂，方法包括
    * makeObject: 生成一个新的PooledObject实例
    * destroyObject: 销毁不需要的PooledObject
    * validateObject: PooledObject归还时验证是否有效
    * activateObject: 钝化的PooledObject从池中再次借出前重新初始化（reinitialize）
    * passivateObject: PooledObject归还时钝化（uninitialize)

    两个默认的抽象工厂类：
    * BasePooledObjectFactory
    * BaseKeyedPooledObjectFactory

    ![image03]({{site.baseurl}}/image/20180910/pooledobjectfactory.png)

- ObjectPool/KeyedObjectPool（对象池）

    负责管理缓冲池中的PooledObject，方法包括
    * borrowObject: 从中借出一个池对象；可能调用PooledObjectFactory.makeObject方法创建，或者对一个空闲对象调用PooledObjectFactory.activeObject激活，然后调用PooledObjectFactory.validateObject方法验证后返回
    * returnObject: 将一个池对象返回给缓冲池
    * invalidateObject: 废弃一个池对象
    * addObject: 使用工厂创建一个对象
    * getNumIdle: 返回缓冲池中空闲池对象数量
    * getNumActive: 返回借出池对象数量
    * clear: 清除池中所有空闲池对象
    * close: 关闭池并释放相关资源

    5个默认实现：
    * GenericObjectPool： 默认Last In First Out队列方式
    * SoftReferenceObjectPool：对象被包装到SoftReference中，允许垃圾回收机制在需要释放内存时回收对象池中的对象
    * ProxiedObjectPool ： 使用动态代理技术，代理由GenericObjectPool或SoftReferenceObjectPool生成的PooledObject对象
    * GenericKeyedObjectPool
    * ProxiedKeyedObjectPool

    ![image04]({{site.baseurl}}/image/20180910/objectpool.png)

PooledObjectState枚举了PooledObject可能的状态：

状态| 描述
:----|:----
IDEL | 位于队列中，未使用
ALLOCATED | 在使用
EVICTION | 位于队列中，当前正在测试，可能会被回收
EVICTION_RETURN_TO_HEAD | 不在队列中，当前正在测试，可能会被回收。待测试完毕后，如果没被回收，该对象放回队列头
INVALID | 回收或者验证失败，将销毁
ABANDONED | 即将无效
RETURNING | 返还到池中

PooledObject状态变化如下图所示：

![image05]({{site.baseurl}}/image/20180910/object_state.png)

GenericObjectPool中借出对象的逻辑实现，有阻塞式和非阻塞式两种获取对象的模式：

![image06]({{site.baseurl}}/image/20180910/borrow.png)

GenericObjectPool中归还对象的逻辑实现，有先进先出（FIFO）和后进先出（LIFO）两种模式：

![image07]({{site.baseurl}}/image/20180910/return.png)

参考：  
[CNBLOG: Apache Commons Pool2源码分析](https://www.cnblogs.com/softidea/p/5759466.html)  
[CSDN: Apache common-pool2 2.4.2源码学习笔记](https://blog.csdn.net/zilong_zilong/article/details/78556281)  
[RUNOOB.COM:抽象工厂模式](http://www.runoob.com/design-pattern/abstract-factory-pattern.html)

