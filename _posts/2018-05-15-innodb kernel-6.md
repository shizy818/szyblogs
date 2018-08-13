---
title:  “InnoDB 内核阅读笔记（六）- 记录”
date:   2018-05-15 08:00:12 +0800
categories: database
---

InnoDB存储引擎，同其他大多数关系数据库一样，是一种面向行（row-oriented)的存储引擎。而MySQL Infobright存储引擎则是面向列（column-oriented）的存储引擎。

# **data & rem文件夹**

文件名称 |  说明
:----|:----
data0data.cc | 逻辑记录实现
data0type.cc | 逻辑记录常用操作
rem0cmp.cc | 物理记录的比较
rem0rec.cc | 物理记录实现

源码中，文件data0*.*用来实现逻辑记录（内存），文件rem0*.*用来实现物理记录（磁盘）。


# **物理记录**

除了实际存储行数据，物理记录包含一些额外信息，包括
- 变长列长度列表： 记录中每个列所在位置偏移量列表，根据列的顺序逆序存放
- NULL标志(1 byte): 表示第几列有NULL值
- 记录头（5 bytes）
- RowID (6 bytes)
- 事务ID (6 bytes)
- 回滚指针 (7 bytes)

记录头内容：

名称 | 大小 |  说明
:----|:----|:----
| 1 | 未使用
| 1 | 未使用
deleted_flag | 1 | 记录删除标记
min_rec_flag | 1 | B+树中非叶子节点最小记录标记
n_owned | 4 | 该记录对应槽所拥有记录数量
heap_no | 13 | 该记录在堆中的序号
record_type | 3 | 000=普通 001=B+树节点指针 010=infimum 011=supremum 1xx=保留
next_recorder | 16 | 页中下一条记录的相对位置

回滚指针用来构造当前记录的上一个版本，以此实现事务回滚和多版本并发控制；事务ID列用来判断当前记录对于其他事务是否可见，用来实现事务的隔离性以及多版本并发控制。

## 溢出页

对于BLOB和TEXT类型列，如果记录占用空间超出某些条件，需要将这些列的数据存放到溢出页上，称这种列的属性为extern。

名称 | 大小 |  说明
:----|:----|:----
BTR_EXTERN_SPACE_ID | 4 | 溢出页的SPACE ID
BTR_EXTERN_PAGE_NO | 4 | 溢出页的PAGE NO
BTR_EXTERN_OFFSET | 4 | 溢出页中记录存放的开始位置
BTR_EXTERN_LEN | 8 | 存放在溢出页的字节数

在Antelope文件格式（Compact，Redundant）中，在20个字节存放指向溢出页的信息之前，还有768个字节的前缀数据。新的Barracuda文件格式（Compressed，Dynamic）采用了完全的行溢出方式，即没有了前缀数据。Compressed行记录格式还会以zlib算法压缩BLOB，TEXT和VARCHAR这类大长度类型数据。

BTR_EXTERN_LEN的第一个字节，用来存放其他属性信息。
```cpp
#define BTR_EXTERN_OWNER_FLAG		128
#define BTR_EXTERN_INHERITED_FLAG	64
```
对大记录进行更新时，可能会遇到主键值需要更新，但是extern列不需要更新的情况。此时InnoDB引擎首先将原主键记录进行伪删除（置delete flag=1），然后插入一条含有
新主键的记录，最后将BTR_EXTERN_*指向原有溢出页。但是如果发生回滚操作，删除新主键记录，重置伪删除记录的delete flag，此时由于extern列继承前一个较早版本，其实不需要删除，因此只需要通过BTR_EXTERN_INHERITED_FLAG进行标识。

# **逻辑记录**

逻辑记录放在内存中，由数据结构dtuple_t表示，其中字段由数据结构dfield_t表示。
```cpp
/** Structure for an SQL data field */
struct dfield_t{
	void*		data;	/*!< pointer to data */
	unsigned	ext:1;	/*!< TRUE=externally stored, FALSE=local */
	unsigned	len:32;	/*!< data length; UNIV_SQL_NULL if SQL null */
	dtype_t		type;	/*!< type of data */
};

/** Structure for an SQL data tuple of fields (logical record) */
struct dtuple_t {
	ulint		info_bits;	/*!< info bits of an index record:
					the default is 0; this field is used
					if an index record is built from
					a data tuple */
	ulint		n_fields;	/*!< number of fields in dtuple */
	ulint		n_fields_cmp;	/*!< number of fields which should
					be used in comparison services
					of rem0cmp.*; the index search
					is performed by comparing only these
					fields, others are ignored; the
					default value in dtuple creation is
					the same value as n_fields */
	dfield_t*	fields;		/*!< fields */
	UT_LIST_NODE_T(dtuple_t) tuple_list;
					/*!< data tuples can be linked into a
					list using this field */
#ifdef UNIV_DEBUG
	ulint		magic_n;	/*!< magic number, used in
					debug assertions */
/** Value of dtuple_t::magic_n */
# define		DATA_TUPLE_MAGIC_N	65478679
#endif /* UNIV_DEBUG */
};
```

变量 | 类型 |  说明
:----|:----|:----
info_bits | ulint | 等同于物理记录里info_bits
n_fields | ulint | 记录中列的数量
n_fields_cmp | ulint | 在记录进行比较时，仅比较这些数量的列。
fields | dfield_t* | 数组类型，表示记录中的列
tuple_list | UT_LIST_NODE_T(dtuple_t) | 数组类型，表示由多个记录组成

![image01]({{site.baseurl}}/image/20180515/logic_record.png)

溢出数据在内存里用数据结构big_rec_t和big_rec_field_t表示。函数dtuple_convert_big_rec将逻辑记录转化为大记录格式；函数dtuple_convert_back_big_rec将大记录对象重新转化为普通逻辑记录。

# **记录比较**

InnoDB B+树叶子节点页需要通过二叉算法查找，才能最终定位到查询记录。这需要对记录进行比较。

数据结构dtype_t定义如下：
```cpp
struct dtype_t{
	unsigned	prtype:32;	/*!< precise type; MySQL data
					type, charset code, flags to
					indicate nullability,
					signedness, whether this is a
					binary string, whether this is
					a true VARCHAR where MySQL
					uses 2 bytes to store the length */
	unsigned	mtype:8;	/*!< main data type */

	/* the remaining fields do not affect alphabetical ordering: */

	unsigned	len:16;		/*!< length; for MySQL data this
					is field->pack_length(),
					except that for a >= 5.0.3
					type true VARCHAR this is the
					maximum byte length of the
					string data (in addition to
					the string, MySQL uses 1 or 2
					bytes to store the string length) */
#ifndef UNIV_HOTBACKUP
	unsigned	mbminmaxlen:5;	/*!< minimum and maximum length of a
					character, in bytes;
					DATA_MBMINMAXLEN(mbminlen,mbmaxlen);
					mbminlen=DATA_MBMINLEN(mbminmaxlen);
					mbmaxlen=DATA_MBMINLEN(mbminmaxlen) */
#endif /* !UNIV_HOTBACKUP */
};
```

可以将mtype理解为列的类型，但不是MySQL数据列类型，比如mtype类型不包括SMALLINT，TINYINT，ENUM等MySQL列类型。函数get_innobase_type_from_mysql_type负责将MySQL上层的列类型转化为数据结构dtype_t的mtype类型。

类型 | 值 |  说明
:----|:----|:----
DATA_VARCHAR | 1 | latin1字符集VARCHAR类型
DATA_CHAR | 2 | latin1字符集CHAR类型
DATA_FIXBINARY | 3 | BINARY类型
DATA_BLOB | 4 | 大记录类型
DATA_INT | 5 | INT类型
DATA_SYS_CHILD | 7 | B+树索引中非叶子节点记录的指针列
DATA_SYS | 8 | 系统列类型，如ROWID列，事务ID列，回滚指针列
DATA_FLOAT | 9 | FLOAT类型
DATA_DOUBLE | 10 | DOUBLE类型
DATA_DECIMAL | 11 | DECIMAL类型
DATA_VARMYSQL | 12 | 非latin1字符集VARCHAR类型
DATA_MYSQL | 13 | 非latin1字符集CHAR类型

prtype表示对于列更精准的表述，比如列的属性NOT NULL或者UNSIGNED。

类型 | 值 |  说明
:----|:----|:----
DATA_ROW_ID | 0 | row id主键列
DATA_TRX_ID | 1 | 事务id列
DATA_ROLL_PTR | 2 | 回滚指针列
DATA_NOT_NULL | 256 | NOT NULL属性
DATA_UNSIGNED | 512 | UNSIGNED属性

逻辑记录与物理记录的比较通过函数cmp_dtuple_rec_with_match完成；物理记录之间的比较通过函数cmp_rec_rec_with_match完成。

# **记录版本控制**

InnoDB引擎中，使用数据结构read_view_t来判断事务应该读取记录的哪个版本。

变量 | 类型 |  说明
:----|:----|:----
type | ulint | VIEW_NORMAL或VIEW_HIGH_GRANULARITY
undo_no | undo_no_t | VIEW_HIGH_GRANULARITY类型下的undo日志号，因为HIGH GRANULARITY限制下读操作看不到read_view_t创建后当前事务的改动。
low_limit_no | trx_id_t | 提交时间早于此值的事务，可以被purge线程回收
low_limit_id | trx_id_t | 大于等于此值的事务（trx->id >= low_limit_id)，当前ReadView均不可见；low_limit_id=trx_sys->max_trx_id
up_limit_id | trx_id_t |  小于此值的事务（trx->id < up_limit_id），当前ReadView一定可见；up_limit_id=ReadView创建时系统最小活跃事务ID
n_trx_ids | ulint | 活跃事务数量
trx_ids | trx_id_t* | 数组，表示事务开始时存在的其他事务
creator_trx_id | trx_id_t | 哪个事务创建了当前read_view_t结构
view_list | UT_LIST_NODE_T(read_view_t) | trx_sys里的read view链表

如果事务采用一致性非锁定读（consistent nonlocking read)，读取时会在事务初始化时候调用`trx_assign_read_view -> read_view_open_now`分配一个read_view_t数据结构。

假设当前全局事务链表（trx_sys)有6个活跃事务，根据事务ID逆序存放，因此最小的事务ID在尾端：

> cr_trx => trx9 -> trx7 -> trx6 -> trx5 -> trx3 -> trx2

在这个例子里，cr_trx为当前创建read_view_t的事务，其状态大致为：

> read_view -> creator_trx_id = cr_trx;  
> read_view -> up_limit_id = trx2;  
> read_view -> low_limit_id = trx9;  
> read_view -> trx_ids = [trx9, trx7, trx6, trx5, trx3, trx2];  
> read_view -> n_trx_ids = 6;

小于trx2的事务属于已提交事务，因此当前事务可以看见其修改记录；大于trx9的事务在当前事务之后开始，因此其记录对于当前事务不可见；对于low_limit_id和up_limit_id之间的事务，取决于事务隔离级别以及是否提交。