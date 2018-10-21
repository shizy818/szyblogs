---
title:  “InnoDB 内核阅读笔记（十一）- 缓冲池”
date:   2018-06-05 08:00:12 +0800
categories: database
---

磁盘数据库系统（disk-based database）通常使用缓冲池（Buffer Pool）来弥补CPU速度和磁盘速度之间的鸿沟。简单说缓冲池就是一块内存区域，通过内存来缓存频繁访问的数据，从而减少传统机械磁盘速度较慢对于数据库性能的影响。缓冲池中缓存的数据页类型有：索引页，数据页，undo页，插入缓存（Insert Buffer），自适应哈希索引（Adaptive Hash Index），锁信息。

每个事务锁的信息是在trx_lock_t对象的lock_heap变量中进行分配的，当申请空间大于8KB时，就会从缓冲池中进行分配，所以有部分内存空间保存的锁信息。这种情况发生在一个事务对大量记录进行了上锁操作。


通常来说，数据库中的缓冲池都是通过LRU（Latest Recent Used最近最少使用）算法进行管理，即最频繁使用的页在LRU链表的前端，最少使用的页在LRU链表的尾端。当缓冲池不能存放新读取到的页时，从LRU链表中释放尾端的页。稍有不同的是，InnoDB存储引擎中，新页并不是放在链表首部，而是默认放在链表3/8处（LRU_old_ratio）。这样是为了防止某些热点数据页从LRU链表中被刷出。另外需要注意的是，有些页（如自适应哈希，锁信息）虽然通过free链表申请空间，但并不放入LRU链表中。

# **buf文件夹**

文件名称 |  说明
:----|:----
buf0buf.cc | 缓冲池对象
buf0flu.cc | 缓冲池对于脏页刷新管理
buf0rea.cc | 从外存读取页到缓冲池
buf0lru.cc | 缓冲池中页的管理

## 相关数据结构

缓冲池主要的数据结构：buf_pool_t用于内存空间的申请和状态管理，buf_block_t用于记录缓冲池中每个页的信息以及状态，buf_page_t用于记录每个页的信息以及状态。Block(块)通常用于表示文件系统中一次I/O的大小，page用于缓冲池，frame表示在缓冲池中页对应的内存空间。在数据库中block和page是基本等同的概念。

buf_page_t结构

变量 | 类型 |  说明
:----|:----|:----
space | ib_uint32_t | 页的表空间ID
offset | ib_uint32_t | 页的表空间偏移量
state | unsigned | 有效值如BUF_BLOCK_READY_FOR_USE，BUF_BLOCK_FILE_PAGE，BUF_BLOCK_MEMORY等
flush_type | unsigned | 当前页如果正在被刷新，是从哪个链表刷出的，有效值为BUF_FLUSH_LRU，BUF_FLUSH_LIST或BUF_FLUSH_SINGLE_PAGE
hash | buf_page_t* | hash表上用于连接具有相同哈希值的页
newest_modification | lsn_t | 页最新修改的lsn
oldest_modification | lsn_t | 页首次修改的lsn
list | UT_LIST_NODE_T(buf_page_t) | 页在free链表或者flush链表中的位置
LRU |  UT_LIST_NODE_T(buf_page_t) | 页在lru链表中的位置
freed_page_clock | unsigned | 

buf_block_t结构

变量 | 类型 |  说明
:----|:----|:----
pace | buf_page_t | 对应的缓冲页
frame | byte* | 页对应在缓冲池中的内存地址，实际保存页的数据
mutex | ib_mutex_t | 并发保护buf_block_t
lock | rw_lock_t | 并发保护frame变量
modify_clock | ib_uint64_t |
index | dict_index_t* |

buf_pool_t结构

变量 | 类型 |  说明
:----|:----|:----
mutex | ib_mutex_t | 并发保护结构成员的latch
n_chunks | ulint | 缓冲块数量
chunks | buf_chunk_t* | 缓冲块指针
curr_size | ulint | 缓冲池当前容量，以页为单位
page_hash | hash_table_t* | 哈希表
flush_list | UT_LIST_BASE_NODE_T(buf_page_t) | flush链表
init_flush[BUF_FLUSH_N_TYPES] | ibool | 相应类型页的刷新操作是否已经开始
n_flush[BUF_FLUSH_N_TYPES] | nlint | 相应类型正在等待刷新页的数量
no_flush[BUF_FLUSH_N_TYPES] | os_event_t | 相应刷新类型的事件，用于唤醒等待脏页刷新的线程
freed_page_clock | ulint | 
free | UT_LIST_BASE_NODE_T(buf_page_t) | free链表
LRU | UT_LIST_BASE_NODE_T(buf_page_t) | LRU链表
LRU_old | buf_page_t* | 指向LRU链表old端起始buf_page_t对象，如果LRU的长度小于BUF_LRU_OLD_MIN_LEN则为NULL
LRU_old_len | ulint | LRU链表old端中页的数量

# **缓冲池管理**

## 链表维护

当LRU链表长度小于BUF_LRU_OLD_MIN_LEN（默认80）时，采用朴素的LRU算法，新读入的页放到LRU链表的第一个位置。若大于BUF_LRU_OLD_MIN_LEN，则将新读入的页放到LRU_old的后面，这部分操作由函数buf_LRU_add_block实现。

根据LRU算法原则，数据库总算希望将最热点的页放入到LRU链表的活跃处。每次页从缓冲池被替换出去，buf_pool->freed_page_clock就加1；page->freed_page_clock表示上次将页移动到LRU链表头部的buf_pool->freed_page_clock的值。若有大量的页被替换出去，那么有可能当前再次读取的页也有被替换出去的风险。因为再次被读取的页是热点，所以需要通过函数buf_page_make_young将其移动到LRU链表的首部，保证其不被轻易的替换出去。

当缓冲池的页被刷新到磁盘后，会将页移动到LRU链表的尾端，表示其可以从LRU链表中被替换，由函数buf_LRU_make_block_old实现。

## 页的分配

页首先尝试从缓冲池的free链表中申请空间，若free链表中已无可用页，则需要向LRU链表中申请。LRU链表尾端的页可以被替换，然而如果页是脏的，那么在被替换之前要进行刷新操作。然后对可以替换页进行free操作，这涉及到将bpage->state状态置为BUF_BLOCK_MEMORY，从缓冲池的哈希表中删除页，若页已经创建了自适应哈希，还需要删除对应记录的哈希信息。

# **页的读取**

InnoDB启动后，所有页都在free链表中。将外存中的页读取到缓冲池中，一般称做页的物理读取。读取一个页最底层函数为buf_read_page_low，首先调用函数buf_page_init_for_read从缓冲池中通过LRU算法分配一个buf_page_t对象，并对该对象加上x-latch用以保护其指向的内存(block->frame)。读取操作完成时，调用函数buf_page_io_complete释放x-latch。

可能存在多个线程调用buf_page_init_for_read，但其中会通过buf_pool->mutex来进行并发控制的保护。buf_page_t通过函数buf_page_init_for_read初始化完成后，调用函数fil_io将外存上的页读取到block->frame中。读取操作分为同步和异步（变量sync控制）。对于同步读取，操作完成后直接调用函数buf_page_io_complete释放持有的x-latch；对于异步读取，最终通过io-thread释放x-latch（fil_aio_wait）。

- 随机预读

随机预读指判断某个区域内的页是否大多数已经被访问，且被访问的页是热点数据，满足条件则InnoDB认为该区域的页都可能需要被访问，提前进行读取操作。随机预读区域大小通过buf_read_ahead_random_area定义；其次宏BUF_READ_AHEAD_RANDOM_THRESHOLD定义默认阀值，即区域中有多少页被访问过（且活跃）才触发随机预读操作。此外，通过buf_pool->n_pend_reads判断存储引擎的读取压力，当压力过大（缓冲池一半的页都在等待读取操作完成）则不会触发随机预读。

- 线性预读

读取一个页，若该页是某个区域的边界，并且该区域内的部分页已经被顺序的访问，则触发线性预读操作，顺序地读取之后或者之前的buf_read_ahead_linear_area个页。线性预读要求页在物理上也是连续的。

- 逻辑读取

逻辑读取指从缓冲池中访问指定页，若读取页不在缓冲池中，则首先通过物理读取将磁盘上的页加载到缓冲池中；若读取页已经在缓冲池中，则根据缓冲池的哈希表进行搜索。

逻辑读取函数是buf_page_get_gen。变量mode表示访问模式，BUF_GET表示一定要从缓冲池中读取到指定的页；BUF_GET_IF_IN_POOL表示仅从缓冲池中读取，若没有则返回NULL（Insert Buffer需要这样的访问模式）；BUF_GET_NO_LATCH表示读取到的页不需要加latch（非叶子节点页）；BUF_GET_NOWAIT表示对访问的页加latch失败则返回NULL，而不等待页上latch释放。

# **页的刷新**

检查点（checkpoint）是脏页刷新到磁盘的时间点，其目的有：
- 缩短数据库恢复时间
- 缓冲池不够用时，将脏页写回磁盘释放空间
- 重做日志不够用时，将脏页写回磁盘

InnoDB存储引擎存在两种检查点，分别是Sharp Checkpoint和Fuzzy Checkpoint。Sharp Checkpoint发生在数据库关闭时将所有脏页都刷新回磁盘，但运行时一般使用Fuzzy Checkpoint只刷新一部分脏页。Fuzzy checkpoint发生的情况如下:
- Master Thread中以每秒或者每10秒的速度从缓冲池的脏页链表中异步刷新一定比例的页
- 当checkpoint_age超过max_checkpoint_age_async，触发异步写checkpoint过程
- flush_lru_list检查点，因为需要保证lru链表和free链表中有可以立即使用的页。

InnoDB会刷新LRU链表中的脏页（BUF_FLUSH_LRU）以及flush链表中的脏页（BUF_FLUSH_LIST）。开始页的刷新操作时，其他线程不得使用页，刷新操作持有页对象的s-latch，刷新完成后通过IO thread释放s-latch。

为了提高刷新性能，InnoDB还支持邻接页的刷新，通过函数buf_flush_try_neighbors实现。当刷新一个脏页时，如果该页所在某个范围的所有页都是脏页且不在LRU链表的热端，则一起进行刷新。这样做可以将多个I/O写入操作合并为一个I/O操作。