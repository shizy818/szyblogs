---
title:  “InnoDB 内核阅读笔记（一）- 基本数据结构”
date:   2018-04-22 08:00:12 +0800
categories: database
---

之前看过[姜承尧](https://weibo.com/insidemysql)《MySQL技术内幕》一书，后来在2014年出了后续《MySQL内核》，
但是他自己说只卖出了几千本，心灰意冷。《MySQL内核》对于阅读MySQL代码，还是很有帮助的，所以一边读一边做一个笔记
整理。

书中使用MySQL 3.23版本，直到5.6版本都大体相近。但是5.7/8.0版本中似乎有不少改变。


# **ut文件夹**

文件名称 |  说明
:----|:----
ut0bh.cc | 最小堆实现
ut0byte.cc | 字节操作实现
ut0crc32.cc | Facebook CRC校验实现
ut0dbg.cc | debug工具
ut0list.cc | 双向线性链表实现
ut0mem.cc | 内存管理基元，内存块实现
ut0rbt.cc | 红黑树实现
ut0rnd.cc | 寻找素数
ut0ut.cc | 其他常用操作，包括时间，打印等等
ut0vec.cc | 指针向量
ut0wqueue.cc | 工作队列

内存池的实现被单独移到/mem/mem0mem.cc和/mem/mem0pool.cc文件里面去了。

# **内存管理**

InnoDB存储引擎并不直接使用系统原生的malloc/free或者new/delete方法管理内存，因为频繁的系统函数调用会带来潜在性能开销。解决方案就是一次申请分配大块的内存，之后的内存请求由InnoDB引擎负责控制。 上层调用者可以通过mem_area_alloc从`通用内存池`建立内存堆(memory heap)或为内存堆增加一个内存块；或者通过buf_block_alloc从`缓冲池`(buffer pool)分配一整页大小的内存空间。

```cpp
/** The info structure stored at the beginning of a heap block */
struct mem_block_info_t {
	ulint	magic_n;/* magic number for debugging */
#ifdef UNIV_DEBUG
	char	file_name[8];/* file name where the mem heap was created */
	ulint	line;	/*!< line number where the mem heap was created */
#endif /* UNIV_DEBUG */
	UT_LIST_BASE_NODE_T(mem_block_t) base; /* In the first block in the
			the list this is the base node of the list of blocks;
			in subsequent blocks this is undefined */
	UT_LIST_NODE_T(mem_block_t) list; /* This contains pointers to next
			and prev in the list. The first block allocated
			to the heap is also the first block in this list,
			though it also contains the base node of the list. */
	ulint	len;	/*!< physical length of this block in bytes */
	ulint	total_size; /*!< physical length in bytes of all blocks
			in the heap. This is defined only in the base
			node and is set to ULINT_UNDEFINED in others. */
	ulint	type;	/*!< type of heap: MEM_HEAP_DYNAMIC, or
			MEM_HEAP_BUF possibly ORed to MEM_HEAP_BTR_SEARCH */
	ulint	free;	/*!< offset in bytes of the first free position for
			user data in the block */
	ulint	start;	/*!< the value of the struct field 'free' at the
			creation of the block */
#ifndef UNIV_HOTBACKUP
	void*	free_block;
			/* if the MEM_HEAP_BTR_SEARCH bit is set in type,
			and this is the heap root, this can contain an
			allocated buffer frame, which can be appended as a
			free block to the heap, if we need more space;
			otherwise, this is NULL */
	void*	buf_block;
			/* if this block has been allocated from the buffer
			pool, this contains the buf_block_t handle;
			otherwise, this is NULL */
#endif /* !UNIV_HOTBACKUP */
};
```

变量 | 类型 |  说明
:----|:----|:----
base | UT_LIST_BASE_NODE_T | 内存堆中链表基节点，仅在堆中第一个内存块中定义
list | UT_LIST_NODE_T | 用于链接内存堆链表的前后内存块节点
len | ulint | 内存块大小，以字节为单位
total_size | ulint | 所有内存块总大小，仅在堆中第一个内存块中定义
type | ulint | 内存堆的类型
free | ulint | 该内存块的空闲位置相对于内存块起始地址的偏移
start | ulint | 内存块创建时的free字段偏移位置
free_block | void * | 在MEM_HEAP_BTR_SEARCH类型堆中用来包含一个可作为堆空闲块的缓冲页框。改字段仅在该堆需要更多空间时使用
buf_block | void * | 如果该内存块是从缓冲池分配，包含buf_block_t句柄

三种内存堆类型：
- MEM_HEAP_DYNAMIC
- MEM_HEAP_BUFFER
- MEM_HEAP_BTR_SEARCH: MEM_HEAP_BUFFER子类，仅在自适应哈希索引中使用

mem_block_t:  
![image01]({{site.baseurl}}/image/20180422/mem_block.jpeg)

mem_heap_t:  
![image02]({{site.baseurl}}/image/20180422/mem_heap.jpeg)

## 通用内存池

InnoDB启动后，内存实例中会有一个mem_comm_pool对象，称为通用内存池，主要用于小块内存的分配，比如InnoDB引擎内存数据结构对象。大小通过参数innodb_additional_mem_pool_size定义，默认8MB。

mem_pool_t结构：
```cpp
struct mem_pool_t{
	byte*		buf;		/*!< memory pool */
	ulint		size;		/*!< memory common pool size */
	ulint		reserved;	/*!< amount of currently allocated
					memory */
	ib_mutex_t		mutex;		/*!< mutex protecting this struct */
	UT_LIST_BASE_NODE_T(mem_area_t)
			free_list[64];	/*!< lists of free memory areas: an
					area is put to the list whose number
					is the 2-logarithm of the area size */
};
```

频繁的请求和释放不同大小的内存，会导致内存池中存在很大碎片化的小内存区。与Linux内核的内存管理类似，InnoDB的内存池也使用伙伴系统（Buddy System）。内存池中所有内存分组为64个内存链表，每个链表分别包含(2^i)字节的内存区。但实际上前几个列表为空，因为内存池仅将内存区拆分到MEM_AREA_MIN_SIZE大小，MEM_AREA_MIN_SIZE为mem_area_t所占内存空间对齐后的2倍。

![image03]({{site.baseurl}}/image/20180422/buddy.png)

**问题：Linux伙伴系统和InnoDB伙伴系统有何不同？**
- InnoDB有64个链表；Linux kernel是11个
- InnoDB链表单位是字节；Linux Kernel链表对应的是页框，1024个页框的最大请求对应着4MB大小的连续RAM块


# **哈希表**

哈希部分代码在ha目录下。InnoDB存储引擎中，可以通过两种方式来实现哈希链，保存在哈希对象中或者自己创建哈希链。ha0ha.cc对应外部哈希链的情况, 当哈希表中存放的是缓冲池的页（页维护链表信息）或者自适应哈希索引（没有链表信息）时候，通过调用ib_create创建；
hash0hash.cc是普通哈希表，通过调用hash_create创建。

# **双向链表**

双链表也分为两种类型，分别为组织内存对象的内存双链表和组织磁盘文件中数据的磁盘双链表，后者在文件fut/fut0lst.cc中。

内存双链表链接示意图：  
![image04]({{site.baseurl}}/image/20180422/list.png)

磁盘双链表主要使用在表空间管理模块，事务处理模块和B树模块中，用于建立保存在磁盘中的数据结构间关系。磁盘数据块设备无法像内存一样通过指针进行随机地址访问，要访问某个磁盘数据，需要先将其对应磁盘快读取到内存中，然后通过块内偏移位置最终定位所需的数据。

参考：  
[MySQL系列：innodb源码分析之内存管理](https://blog.csdn.net/yuanrxdu/article/details/40985363)