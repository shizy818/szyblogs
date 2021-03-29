---
title:  “InnoDB 内核阅读笔记（十三）- 数据字典”
mathjax: true
layout: post
date:   2018-06-20 08:00:12 +0800
categories: database
---

InnoDB存储引擎中，每个表都有其自己的元数据定义，被称为数据字典（data dictionary）。用户打开一个表，与表相关的定义信息如列，索引等信息就加入到数据字典。用户使用这些对象时无须从磁盘中通过IO加载到内存，对表定义相关修改会通过mini-transaction及时回刷到磁盘中。


# **dict文件夹**

文件名称 | 说明
:----|:----
dict0boot.cc | 启动数据字典模块
dict0crea.cc | 创建数据字典中的对象
dict0dict.cc | 数据字典操作集合
dict0load.cc | 加载数据字典对象
dict0mem.cc | 数据字典内存结构定义及相关操作
dict0stats.cc | 数据字典统计信息

# **数据字典概述**

InnoDB存储引擎的数据字典，与frm文件有何区别？frm文件是MySQL数据库上层产生的文件，可以理解为MySQL数据库上层的数据字典，适用于所有存储引擎。InnoDB的数据字典是引擎内部产生的，存储的都是引擎相关内容，通过mini-transaction来保证事务安全性。frm只是一个比较简单的二进制文件，InnoDB存储引擎的数据字典通过B+树方式进行组织，然后加载到缓冲池进行加速。

dict_sys_t结构

变量 | 类型 | 说明
:----|:----|:----
mutex | ib_mutex_t | 保护数据字典及其表空间对象的互斥量
row_id | row_id_t | 未被分配的最小ID
table_hash | hash_table_t* | 基于数据库表名的哈希表
table_id_hash | hash_table_t* | 基于数据库ID的哈希表
size | ulint | 被数据字典中表和索引对象占据的内存字节数
sys_tables| dict_table_t* | 用于链接SYS_TABLES系统表
sys_columns| dict_table_t* | 用于链接SYS_COLUMNS系统表
sys_indexes| dict_table_t* | 用于链接SYS_INDEXES系统表
sys_fields| dict_table_t* | 用于链接SYS_FILEDS系统表
table_LRU | UT_LIST_BASE_NODE_T(dict_table_t) | InnoDB实例中可以被换出缓冲区的表对象链表
table_non_LRU | UT_LIST_BASE_NODE_T(dict_table_t) | InnoDB实例中不能被换出缓冲区的表对象链表

InnoDB为每一个表新增一列DB_ROW_ID，row_id的取值在数据字典系统对象中，所以为每个行记录分配自增值是全局的，而不是基于某个表递增。当size在缓冲池中占据空间过大时，可以通过清理table_LRU链表来释放部分数据字典空间。

dict_table_t结构，大致分为表的基本信息，表中成员对象信息，表的统计信息和自增锁信息。

变量 | 类型 | 说明
:----|:----|:----
id | table_id_t | 表ID
heap | mem_heap_t* | 表的内存堆
name | char* | 表名
dir_path_of_temp_table | const char* | 临时表存放目录
path_dir_path | char* | 数据存放目录
space | unsigned | 聚集索引存放的表空间
cached | unsigned | 表对象是否已在数据字典缓存中
n_def | unsigned | 表中目前已经定义的列数
n_cols | unsigned | 表中的列数
cols | dict_col_t* | 保存表中各列定义的指针数组
name_hash | hash_node_t | 用于链接dict_sys_t->table_hash的下一个表
id_hash | hash_node_t | 用于链接dict_sys_t->table_id_hash的下一个表
indexes | UT_LIST_BASE_NODE_T(dict_index_t) | 该表所有索引对象的链表
foreign_set | dict_foreign_set | 外键对象集合 （该表为从表）
referenced_set | dict_foreign_set | 外键对象集合 （该表为主表）
table_LRU | UT_LIST_NODE_T(dict_table_t) | 链接到dict_sys_t->table_LRU
stat_n_rows | ib_uint64_t | 该表行的数目，周期性更新
stat_clustered_index_size | ulint | 该表聚集索引大小，以页为单位
stat_sum_of_other_index_sizes | ulint | 该表非聚集索引页数目
stat_initialized | unsigned（1 byte） | 统计字段是否已经初始化
stat_modified_counter | ulint | 该表被修改的次数
autoinc_lock | lock_t* | 该表的自增锁缓冲区
autoinc_mutex | ib_mutex_t* | 保护自增计数器的互斥量
autinc | ib_uint64_t | 自增计数器的值
locks | UT_LIST_BASE_NODE_T(lock_t) | 该表当前所有锁对象的链表

dict_col_t结构

变量 | 类型 | 说明
:----|:----|:----
prtype | unsigned（32 bytes）| 列类型，有效值DATA_ROW_ID, DATA_TRX_ID, DATA_ROLL_PTR, DATA_N_SYS_COLS等
mtype | unsigned（8 bytes）| 数据类型，有效值DATA_VARCHAR, DATA_CHAR, DATA_BINARY等等
len | unsigned（16 bytes）| 列长度
ind | unsigned（10 bytes）| 该列在表中的偏移位置
ord_part | unsigned（1 byte）| 若该列作为索引排序列，则非0

dict_index_t结构

变量 | 类型 | 说明
:----|:----|:----
id | index_id_t | 索引ID
heap | mem_heap_t* | 索引的内存堆
name | const char* | 索引名称
table_name | const char* | 对应表的名称
table | dict_table_t* | 指向对应表对象
space | unsigned（32 bytes）| 索引树存放的表空间ID
page | unsigned（32 bytes）| 索引树根节点所在表空间页号
type | unsigned | 索引类型
trx_id_offset | unsigned | DB_TRX_ID列在该索引中的字节偏移
n_user_defined_cols | unsigned（10 bytes）| 索引中由用户定义的列数
n_uniq | unsigned（10 bytes）| 能够唯一确定索引项所需的列数
n_def | unsigned（10 bytes）| 当前已定义的索引列个数
n_fields | unsigned（10 bytes）| 索引中列的总数
n_nullable | unsigned（10 bytes）| 索引中可以是NULL的列个数
cached | unsigned（1 bytes）| 该索引对象是否已经在数据字典缓存中
fields | dict_field_t* | 索引中列的描述数组
indexes | UT_LIST_NODE_T(dict_index_t) | 链接索引所在表中所有索引对象的链表
search_info | btr_search_t* | 用于乐观查找的信息
stat_n_diff_key_vals | ib_uint64_t | 该索引中不同键值的大概数目，周期性更新
stat_index_size | ulint | 该索引占据的数据库页数量
stat_n_leaf_pages | ulint | 该索引的索引树中叶子页的大概数目

type的类型有以下四种：
- DICT_CLUSTERED 聚集索引
- DICT_UNIQUE 唯一索引
- DICT_UNIVERSAL 普通索引
- DICT_IBUF Insert Buffer索引

表对象有且仅有一个聚集索引，聚集索引记录会添加DB_TRX_ID和DB_ROLL_PTR两个隐藏的系统列。非聚集索引（DICT_UNIVERSAL）会将聚集索引的唯一标识列加入到索引中；唯一索引（DICT_UNIQUE）无须包含DB_ROW_ID。索引类型可以组合，但是DICT_CLUSTERED和DICT_UNIVERSAL是互斥的。

dict_foreign_t结构

变量 | 类型 | 说明
:----|:----|:----
heap | mem_heap_t* | 外键约束对象的内存堆
id | char* | 外键约束ID的字符串
n_fields | unsigned（10 bytes）| 定义外键约束的列个数
type | unsigned（6 bytes）| 有效值DICT_FOREIGN_ON_DELETE_CASCADE, DICT_FOREIGN_ON_DELETE_SET_NULL
foreign_table_name | char* | 外键从表的表名
foreign_table | dict_table_t* | 外键约束的从表对象
foreign_col_names | const char** | 外键从表各列名称的指针
referenced_table_name | char* | 外键主表的表名
referenced_table | dict_table_t* | 外键约束的主表对象
referenced_col_names | const char** | 外键主表各列名称的指针
foreign_index | dict_index_t* | 外键中的从表列在从表中对应的索引
referenced_index | dict_index_t* | 外键中的引用列在主表中对应的索引

## InnoDB系统表对象

- SYS_TABLES
- SYS_COLUMNS
- SYS_INDEXES
- SYS_FIELDS
- SYS_FOREIGN
- SYS_FOREIGN_COLS
- SYS_TABLESPACES


# **数据字典创建**

函数innobase_start_or_create_for_mysql调用dict_hdr_create创建数据字典。由于需要进行磁盘新数据的写入，全程作为一个mini-transaction操作。其主要步骤：

1. 创建数据字典系统表空间段，段的根节点所在页为DICT_HDR_PAGE_NO(7)
2. 初始化段根节点页中的dict_header对象，在对应偏移位置DICT_HDR_ROW_ID，DICT_HDR_TABLE_ID和DICT_HDR_INDEX_ID上写入用户可使用的起始行ID，索引ID和表ID
3. 调用btr_create创建四个系统表SYS_TABLES, SYS_INDEXS, SYS_COLUMNS, SYS_FIELDS

![image01]({{site.baseurl}}/image/20180620/dict_space.png)

函数dict_boot用来加载数据字典到缓存，加速对数据字典的访问。主要步骤：
1. dict_init - 包括分配dict_sys_t对象，初始化基于表名和表ID的两个哈希表对象，创建mutex互斥量和rw_lock读写锁
2. row_id初始化
3. 系统表对象创建 - 分配系统表的缓存对象，调用dict_table_add_to_cache将表加入缓存; 分配索引CLUST_IND和ID_IND缓存对象，调用dict_index_add_to_cache将索引加入缓存。
4. Insert Buffer表对象创建 - 调用ibuf_init_at_db_start初始化插入缓存
5. 系统表非聚集索引创建 - 调用dict_load_sys_table加载四个系统表的非聚集索引

![image02]({{site.baseurl}}/image/20180620/dict_cache.png)

# **数据字典对象加载**

InnoDB实例运行期间需要动态加载用户表相关内容，包括表，索引已经外键约束。

用户表的加载方式分为基于表名（dict_load_table）和基于表ID（dict_load_table_on_id）。同时知道表名和表ID，基于表名加载能有更好的性能表现。函数dict_load_table流程如下：

1. 获取SYS_TABLE的聚集索引CLUST_IND对象sys_index，根据该对象生成dtuple结构并将表名以及数据类型写入其中。调用btr_pcur_open_on_user_rec开始记录查找，调用btr_pcur_get_rec读取满足要求的记录rec。
2. 检查表名是否一致。
3. 调用dict_load_table_low函数分配新的dict_table_t对象，用rec记录中的数据初始化
4. 调用dict_load_columns加载表中所有的列数据
5. 调用dict_table_add_to_cache将该表对象加入数据字典缓存
6. 调用dict_load_indexes加载表中所有的索引对象
7. 调用dict_load_foreigns加载表中所有的外键约束

用户索引和外键约束加载方法与上述用户表加载方法大同小异。