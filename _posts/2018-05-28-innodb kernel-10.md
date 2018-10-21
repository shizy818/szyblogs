---
title:  “InnoDB 内核阅读笔记（十）- 插入缓存”
date:   2018-05-28 08:00:12 +0800
categories: database
---

Insert Buffer可以大幅提高数据库中**非唯一约束的辅助索引**插入性能。自增主键值的插入是顺序的，因此插入有较高的插入性能；而非时间字段的辅助索引，更可能的情况是随机插入。Insert Buffer的设计思想是：在插入时首先判断插入的辅助索引是否在缓冲池中。若在则直接插入，否则将插入的记录放到Insert Buffer中，然后通过后台线程慢慢的合并（merge）回辅助索引中。


# **ibuf文件夹**

文件名称 |  说明
:----|:----
ibuf0ibuf.cc | Insert Buffer实现

# **架构实现**

Insert Buffer其实也是一棵B+树，键值为待插入辅助索引页的page_no。Insert Buffer会将插入的辅助索引记录缓存起来，必须保证一个辅助索引页中的缓冲记录不会引起页的分裂。当缓存的记录过多时，会主动读取辅助索引页，然后进行合并操作。

InnoDB存储引擎存在一个Insert Buffer bitmap页，用来追踪每个索引辅助页的剩余空间，用IBUF_BITMAP_FREE(2 bit)表示：
- 0： 辅助索引页剩余空间小于512B
- 1： 辅助索引页剩余空间为512B ~ 1KB
- 2： 辅助索引页剩余空间为1KB ~ 2KB
- 3： 辅助索引页剩余空间大于2KB

当Insert Buffer bitmap页判断辅助索引页的可用空间小于512字节时，会主动读取辅助索引页，并将Insert Buffer缓存的对应记录插入到辅助索引页中。

Insert Buffer的B+树索引仅有一个数据段，存放所有的非叶子节点和页节点。该段的段头信息不像普通B+树索引保存在root页，而是放在一个独立的页中。另外，Insert Buffer的B+树索引页空间的分配不是直接通过段，而是自己的实现避免死锁。

Insert Buffer插入的记录格式是page_no + field_info + secondary index record。保存这部分信息的意义在于，从Insert Buffer合并到辅助索引页时，需要通过函数ibuf_build_entry_from_ibuf_rec将Insert Buffer中的记录转化为待插入到实际辅助索引页的逻辑记录。

## 合并操作

合并的情况分为主动与被动两类。

主动合并指后台master thread定期将Insert Buffer的记录回刷为辅助索引页，辅助索引页的选择是随机的。在master thread中，若每秒有足够空闲的I/O处理能力，则回刷5个辅助索引页的记录；而每10秒不管I/O压力如何，都回刷5个辅助索引页的记录。

被动合并指有用户线程主动发起辅助索引页的读取操作，这时会被动得将记录合并到原辅助索引中。由于辅助索引页已经读取到缓冲池，之后的操作就不需要再使用Insert Buffer。

关闭MySQL时，InnoDB存储引擎并不需要等待Insert Buffer被合并，而是下次启动时再进行合并。

## 相关数据结构

数据结构ibuf_t表示Insert Buffer内存对象。

变量 | 类型 |  说明
:----|:----|:----
size | ulint | 已经使用页的数量
max_size | ulint | 最大占用缓冲页的数量
seg_size | ulint | 已经从段中分配页的数量
empty | ibool | 当有记录插入到Insert Buffer的B+树时，该值为FALSE
free_list_len | ulint | root页中Free列表中页的数量
height | ulint | Insert Buffer树的高度
index | dict_index_t* | Insert Buffer索引对象
n_merges | ulint | 合并到辅助索引页的次数

# **死锁**

实现Insert Buffer最困难的是对于死锁的处理。例如一个辅助索引页读取到缓冲池，会引发Insert Buffer的合并操作，合并困难会引起Insert Buffer的B+树的收缩。除了Insert Buffer索引相关x-latch，还需要fsp x-latch，因为合并时可能需要对文件存储空间进行扩展或收缩。获得latch顺序是：
> secondary leaf page x-latch  -->  insert buffer tree x-latch  -->  insert buffer page x-latch  -->  fsp x-latch

而普通B+树索引实现中，获取latch的顺序是：
> -->  tree latch  -->  tree non-leaf page latch  --> leaf page latch  -->  fsp x-latch

为了避免上述死锁（？？没搞懂怎么死锁的？？），InnoDB存储引擎将页分为三层：
- 非Insert Buffer页
- Insert Buffer索引内存对象，Insert Buffer页
- Insert Buffer bitmap页
当持有某层latch时不得持有其上层的latch，这意味着，在处理Insert Buffer对象时，fsp模块相关的latch的优先级要高于Insert Buffer索引树的latch。fsp模型相关的latch比索引树latch的优先级高，处理Insert Buffer的B+树索引的合并与扩展就与普通索引树不同。Insert Buffer索引拥有自己的存储管理，root页中存在free list链表，这样的设计使得Insert Buffer在合并和扩展时不需要通过fsp相关模块的latch保护，只需要向free list中添加或删除页即可。

另外一种死锁情况是，所有I/O线程都在异步读取辅助索引页，且都需要进行Insert Buffer合并，这时候没有空闲I/O线程处理Insert Buffer读取从而导致死锁。解决这个问题，将Insert Buffer的异步I/O操作放在一个单独的I/O线程中，且不需要持有B+树索引的各种latch。对于Insert Buffer bitmap页的读取，总是转换为同步读取。另外，辅助索引页在进行读取操作时已经被x-latch，由发起读取的线程持有，同时在异步I/O线程中进行Insert Buffer合并操作时也需要持有该页的x-latch。因此latch是可以变更latch拥有者的，这由函数rw_lock_x_lock_move_ownership完成。