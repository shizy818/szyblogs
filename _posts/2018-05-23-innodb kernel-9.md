---
title:  “InnoDB 内核阅读笔记（九）- B+树索引”
date:   2018-05-23 08:00:12 +0800
categories: database
---

二叉查找树中，左子树键值小于根键值，右子树键值大于根键值。为了优化性能，引入平衡二叉树（AVL），满足任何节点的左右子树高度最大差为1。维护一棵平衡二叉树，对于插入或者删除操作，通常需要1次或者多次左旋和右旋。

B-树是一种平衡多叉树，搜索从根节点开始，对关键字进行二分查找，如果命中即结束；否则进入关键字所属范围的子节点查找（键值可以存放在非叶节点）。B+树是B-树的变体，也是一种平衡多叉树。其特点在于所有键值都在叶子节点出现，且所有叶子节点通过链表连接。B+树更适合用于文件系统和数据库中：
- 查询效率稳定，所有查询均从根节点到叶节点
- **对应数据库中频繁使用的范围查询，B+树可以通过链表遍历，效率更高**


B+树插入关键字时，如果叶节点满了，通常情况需要做拆分（Split）操作，并更新非叶子节点。数据库系统中，B+树结构主要用于磁盘，拆分意味着磁盘操作，所以应尽可能避免。当叶节点左右兄弟节点没有满的情况下，可以通过旋转（Rotation）将记录转移到兄弟节点。(DB2中有这种操作吗？没印象)  
![image01]({{site.baseurl}}/image/20180523/b+tree.png)

插入键值70，结果如下图：  
![image01]({{site.baseurl}}/image/20180523/b+tree_rotation.png)

B+树删除关键字时，如果该关键字同时也在非叶子节点中，需要用后面一个关键字代替。当叶子节点填充因子（Fill Factor）小于0.5时，进行合并操作。

数据库的B+树索引基于磁盘，具有较高的扇出率；索引高度较小，一般3-4层。 B+树分裂由于是内存结构，不需要考虑分裂方向；而B+树索引中，为了充分利用磁盘的顺序特性，InnoDB存储引擎根据不同的插入情况考虑不同的分裂方向。

# **btr文件夹**

文件名称 |  说明
:----|:----
btr0btr.cc | B+树实现
btr0cur.cc | 索引树游标
btr0pcur.cc | 索引树持久游标
btr0sea.cc | 索引自适应搜索

# **数据结构**

B+树索引通过数据结构dict_index_t定义：

变量名 | 类型 |  说明
:----|:----|:----
id | index_id_t | 索引id
space | unsigned | 索引所在表空间
page | unsigned | root页编号
type | unsigned | DICT_CLUSTERED, DICT_UNIQUE, DICT_UNIVERSAL, DICT_IBUF, DICT_CORRUPT
fields | dict_field_t* | 域描述信息数组
indexes | UT_LIST_NODE_T(dict_index_t) | 索引链表
lock | rw_lock_t | 读/写锁，控制索引并发
trx_id | trx_id_t | 创建索引事务id

B+树索引并发控制通过两部分控制；每个叶子节点都有一个读写锁在数据结构buf_block_t中，通过s-latch或者x-latch进行并发控制；此外索引树的内存对象dict_index_t中还有一个读写锁，可以理解为非叶子节点的读写锁，对整棵B+树索引的并发控制，比如分裂等结构变化。

## 整理

InnoDB存储页中的记录是无序存放的，记录间通过record header的next record指向进行串联。分配记录空间时，首先查找PAGE_FREE链表的第一个空间是否满足要求，否则从PAGE_HEAP_TOP处进行分配。当对页进行DML操作时，若空间不足，首先进行整理操作，若还是空间不足，则进行分裂操作。

函数btr_page_reorganize_low用来完成页的整理操作，操作前需对整理的页加x-latch。由于整理后会改变记录的heap_no值，因此还需要调用函数lock_move_reorganize_page更新记录上锁信息。

## 分裂

当页进行分裂操作时，首先需要通过page header中的值判断记录的插入模式
- PAGE_LAST_INSERT
- PAGE_DIRECTION
- PAGE_N_DIRECTION

往右分裂通过函数btr_page_get_split_rec_to_right确定分裂点。PAGE_N_DIRECTION表示往同一方向插入了多少记录，老版本中当该值大于BTR_PAGE_SEQ_INSERT_LIMIT（默认值5），InnoDB认为插入是顺序的。后面的版本中，假设新插入记录在同一页上一次插入记录的后面，即可认为是顺序插入。分裂时判断next_rec是否是suprenum记录，然后分别处理，往左分裂基本同理。无序插入时，取页的中间记录作为分裂点，通过函数page_get_middle_rec完成。
```cpp
    /* We use eager heuristics: if the new insert would be right after
	the previous insert on the same page, we assume that there is a
	pattern of sequential inserts here. */

    // insert_point总是定位到待插入记录之前的一条记录
    if (page_header_get_ptr(page, PAGE_LAST_INSERT) == insert_point) {

		rec_t*	next_rec;

		next_rec = page_rec_get_next(insert_point);

		if (page_rec_is_supremum(next_rec)) {
split_at_new:
			/* Split at the new record to insert */
			*split_rec = NULL;
		} else {
			rec_t*	next_next_rec = page_rec_get_next(next_rec);
			if (page_rec_is_supremum(next_next_rec)) {

				goto split_at_new;
			}

			/* If there are >= 2 user records up from the insert
			point, split all but 1 off. We want to keep one because
			then sequential inserts can use the adaptive hash
			index, as they can do the necessary checks of the right
			search position just by looking at the records on this
			page. */

			*split_rec = next_next_rec;
		}

		return(TRUE);
	}
```

分裂操作函数为btr_page_split_and_insert。为了保证分裂时并发数据的一致性，需要对B+树索引内存对象加x-latch，同时对分裂页加x-latch。步骤为：
1. 确定分裂点记录(split_rec)，如果split_rec是NULL，说明新插入记录应该放在右边的新页中
2. 从索引数据段分配一个新页，并对这个页加上x-latch
3. 确定需要移动页中开始的第一个记录first_rec以及移动到的记录move_limit
4. 更新上层节点记录
5. 将记录移动到新页
6. 确定待插入的记录插入到哪一页
7. 重定位游标，插入新纪录
8. 若插入操作失败，对页进行重新组织，然后重新进行插入操作
9. 若上述操作失败，回到步骤1再次进行分裂操作

步骤4由函数btr_attach_half_pages完成，根据插入方向更新上层节点记录，并插入分裂点记录。由于在上层节点进行插入操作，可能再次导致分裂操作。btr_attach_half_pages结束后调用btr_page_insert_fits判断是否会再次导致分裂。如果没有，且插入是在叶子节点，则释放内存索引对象dict_index_t上的x-latch（读写锁），以减少竞争。叶子节点索引页的x-latch不在函数btr_page_split_and_insert中释放。

往左分裂，步骤5调用函数page_move_rec_list_start和lock_update_split_left完成分裂页中记录移动和对应锁信息更新；往右分裂，分别调用page_move_rec_list_end和lock_update_split_right。分裂完成后，插入逻辑记录tuple到页中。

若root页进行分裂，B+树索引的分裂操作导致高度增加，此过程由函数btr_root_raise_and_insert完成。root页的分裂需要进行两次分裂操作，第一次为产生一个新的root页，第二次是对插入的记录再次进行分裂。

## 合并

当页面进行删除或更新，填充率低于BTR_CUR_PAGE_COMPRESS_LIMIT（默认页面大小的50%），InnoDB尝试将记录合并到页的左兄弟页或者右兄弟页中。若待被合并的页为当前层最后一页，则需要对B+树的高度进行调整。合并操作需要对索引内存对象以及待合并页加x-latch。

合并操作由函数btr_compress完成，需要对内存索引对象，待合并页以及进行合并的兄弟页加x-latch。

# **查找**

前面索引页章节介绍如何在页中定位查询的记录，此处是介绍如果通过B+树索引定位到记录。简单来说，B+树索引的查找是通过从root页开始查找记录，通过二分法最终得到查询结果。模块page（索引页）不涉及并发控制，并发操作是在模块btr（B+索引树）中完成的。

函数btr_cur_search_to_nth_level用来查找指定的记录。参数包括：
- mode  
	* 插入操作： PAGE_CUR_LE
	* 主键和唯一索引查询： PAGE_CUR_GE。实际页的存储中，可能包含多个键值相同记录，只是仅一个记录的delete flag为0（未被删除），且该记录总是在最后一个。
	* 非叶子节点查询：只能是PAGE_CUR_LE或者PAGE_CUR_L （B+树的特点，非叶子节点的键值是右子树键值最小值）

	有时候函数btr_cur_search_to_nth_level查询得到的结果为记录supremum，而不是直接查询得到主键记录。这是因为虽然主键查询模式为PAGE_CUR_GE，但是非叶子节点的查询模式为PAGE_CUR_L。因此函数最后返回了伪记录supremum，但在函数row_search_for_mysql中，InnoDB引擎会自动搜索下一个页得到最终正确的数据。这种情况还可能对加锁操作产生一定影响，因为会对左页记录supremum上s-lock，从而影响并发的插入。

- latch_mode
	* BTR_SEARCH_LEAF：索引对象加s-latch，查询页加s-latch，当查询到叶子节点时释放索引对象的s-latch
	* BTR_MODIFY_LEAF：索引对象加s-latch，查询页加x-latch，当查询到叶子节点时释放索引对象的s-latch
	* BTR_NO_LATCHES：索引对象加s-latch，查询页不加任何latch
	* BTR_MODIFY_TREE：索引对象加x-latch，查询页加x-latch
	* BTR_CONT_MODIFY_TREE：函数btr_cur_search_to_nth_level开始前已经对索引对象加x-latch，查询页加x-latch
	* BTR_SEARCH_PREV：查询页加s-latch
	* BTR_MODIFY_PREV：查询页加x-latch

- cursor  
	btr_cur_t数据结构对象，用来保存查询得到的记录结果。其中变量flag表示使用何种查询得到记录结果，有效值为：
	* BTR_CUR_HASH：使用自适应哈希查询查询得到结果
	* BTR_CUR_HASH_FAIL：使用自适应哈希失败，之后使用B+树索引成功
	* BTR_CUR_BINARY：直接使用B+树索引查询得到结果
	* BTR_CUR_INSERT_TO_IBUF：使用插入缓存进行记录的插入操作
	* BTR_CUR_DELETE_IBUF：使用插入缓存进行记录的删除

	变量path用来保存查询路径，其中nth_rec表示查询得到的记录是页中的第几个记录，n_recs表示页中一共有多少个记录。通过这些信息可以预估得到一个范围查询返回的记录数量，这通过btr_estimate_n_rows_in_range完成。

# **DML操作**

## 插入

对于插入操作而言，首先调用乐观插入函数btr_cur_optimistic_insert；若发现此次操作会导致页的分裂，则调用悲观插入函数btr_cur_pessimistic_insert，悲观插入调用分裂函数btr_page_split_and_insert。

乐观插入函数大致流程如下：
1. 插入记录开始为逻辑记录，需要将其转化为物理记录，并计算页是否有足够的空间，若没有函数返回DB_FAIL。注意InnoDB需要页为UPDATE操作预留空间。
2. 如插入记录tuple需要转化为大记录格式（LOB)，将溢出页部分放入变量big_rec；等btr_cur_optimistic_insert完成后再插入列溢出部分。
3. 函数btr_cur_ins_lock_and_undo检查锁信息(调用lock_rec_insert_check_and_lock)，并生成对应undo日志。如果其他查询事务获得了下一条记录的Next-Key锁，则返回DB_WAIT_LOCK。
4. 调用page模块的page_cur_insert_rec进行记录插入，若插入失败，调用函数btr_page_reorganize进行页整理，再次尝试记录插入操作。
5. 更新页的自适应哈希索引
6. 若插入操作需要更新该页锁信息（如插入时发生了等待）或者flag不包含BTR_NO_LOCKING_FLAG（BTR_NO_LOCKING_FLAG表示不需要对插入数据上锁保护，例如Insert Buffer，不会有并发的读取操作），调用lock_update_insert
7. 若插入对象是插入缓冲（Insert Buffer），更新对应插入缓存位图页信息

函数btr_cur_optimistic_insert完成后，会释放持有页的x-latch。若big_rec不为NULL，表示有行溢出数据，这时候需要对索引内存对象加x-latch，再对页加上x-latch，之后调用函数btr_store_big_rec_extern_fields将溢出的列数据存放到溢出页中。

悲观插入前，会再尝试一次乐观插入btr_cur_insert_if_possible（这时候仍持有内存索引对象的x-latch)，这是因为可能事务在乐观插入失败后和本次悲观插入前，有其他事务插入导致了页的分裂。悲观插入函数的mini-transation持有索引对象和叶子页的x-latch，同时如果兄弟页存在还必须持有兄弟页的x-latch，因为如果兄弟页同时发生分裂可能造成死锁。

## 更新

- 非主键更新

	非主键更新分为原地更新（Update in place)和普通更新，原地更新指更新记录中各个列的大小在更新过程中没有发生改变。更新操作发生变化列的新值保持在数据结构upd_t中。一般辅助索引更新都不是原地更新，而是delete mark + insert操作，这是因为辅助索引列更新会导致B+树索引列的值发生变化。有特殊情况是如果辅助索引列字符集发生变化，而排序规则没有发生改变，则可以进行原地更新。

	乐观更新btr_cur_optimistic_update不对主键进行修改，仅需对持有记录所在页上x-latch。首先将更新记录可能存在的隐式锁转为显示锁，然后尝试加x-lock（可能需要等待其他事务释放更新记录所持有的lock对象）。若不能进行原地更新，那么先删除原记录，然后插入新记录，但是记录的主键并没有改变，因此记录还是在同一页中，只是heap_no发生变化，需要更新锁信息。将原来锁信息移动到伪记录infimum，待记录更新完成再将伪记录上的锁移动到新纪录上。
	
	悲观更新由函数btr_cur_pessimistic_update完成，函数开始前已经持有了索引内存对象的x-latch以及更新记录所在页的x-latch。悲观更新可以处理更新导致页面分裂的情况，其插入操作使用函数btr_cur_pessimistic_insert，在某些情况下，需要对伪记录锁进行修正。

- 主键更新

	1. 将原主键记录delete flag置为1
	2. 插入新主键记录的值
	3. purge线程判断是否有其他事务引用原主键记录，若无，彻底删除

## 删除

删除操作包括2个步骤：
1. 更新记录delete flag（delete mark）
2. purge线程彻底删除记录

步骤1只是将记录的delete flag设置为1，记录仍存在于页中。步骤2是彻底删除，会将记录占用空间放入页的PAGE_FREE链表中。

函数btr_cur_del_mark_set_clust_rec和btr_cur_del_mark_set_sec_rec分别对聚集索引记录和辅助索引记录进行delete mark操作。上述操作都需要对记录加显式锁，分别调用lock_clust_rec__modify_check_and_lock和lock_sec_rec__modify_check_and_lock函数对记录加锁。对于聚集索引记录的伪删除操作，还需产生undo日志。

# **持久游标**

在大多数情况下，上层并不直接调用btr0cur中的查询函数，比如btr_cur_search_to_nth_level，而是通过一个称为持久游标（Persistent Cursor）的对象来处理查询，并通过这个对象来调用btr0cur中的查询函数，将查询的记录信息保存在持久游标中。

在进行SELECT/UPDATE/DELETE操作时，首先需要定位到第一条记录，然后开始扫描下一条记录（fetch next record），直到不符合条件记录为止。持久游标用于保存每次查询到的记录，待查询下一次记录时，首先恢复上一次查询的记录，然后再获取下一条记录。

这样设计的原因是当用户扫描记录时，页中的记录可能会发生变化。这时若按照之前的记录进行扫描，可能得到错误情况，比如页发生了分裂或者合并操作。将每次查询得到记录通过函数btr_pcur_store_position保存在变量old_rec中，待扫描下一记录时，对记录进行恢复，若页没有发生改变则可以直接使用old_rec来定位下一条记录，否则根据old_rec中的索引键重新定位记录再进行查询，这个过程在函数btr_pcur_restore_position完成。

唯一键值的等值查询且锁模式不为LOCK_X时，不需要持有游标保存查询记录。（返回一条匹配记录）。

# **自适应哈希索引**

InnoDB存储引擎支持一种称为自适应的哈希索引（Adaptive Hash Index），即根据查询模式，对活跃查询页中的记录进行哈希索引，从而实现快速查找。自适应哈希是维护InnoDB叶子页记录的索引键值(或键值前缀)的到叶子节点记录的哈希映射关系，能够根据索引键值(或前缀)定位到记录的地址。这样可以不用再从B+树root节点搜索到到叶子节点。

哈希前缀参数(n_fields, n_bytes, left_side)的确定需要根据查询结束后btr_cur_t结构中的参数来确定；主要是比较cursor->up_match，cursor->up_bytes，cursor->low_match，cursor->low_bytes，即查询结束后查询游标指向的前后记录的匹配程度来确定前缀的参数。以匹配少的记录作为前缀，即cursor->up_match，cursor->up_bytes组合与cursor->low_match, cursor->low_bytes比较结果，如果前者大则用后者来更新info->n_fields和info->n_bytes，此时info->left_side
 = TRUE，当遇到相同记录前缀时，选择最左边插入hash表；后者大则用前者来更新info->n_fields和info->n_bytes，此时info->left_side = FALSE，当遇到相同记录前缀时，选择最右边的记录插入hash表。

所谓自适应，是指哈希索引的创建与维护都是由存储引擎自身完成，用户不能自己进行创建。此外，并不是对整张表的数据进行索引，而是对热点页所在的数据进行哈希索引。

| B+树索引 |  自适应哈希索引
:----|:----|:----
查询时间复杂度 | O(logN) | O(1)
是否持久化 | 是 | 否
索引对象 | 页 | 热点页记录

InnoDB存储引擎通过变量hash_analysis和n_hash_potential来判断是否创建哈希索引，步骤为首先判断变量hash_analysis是否大于BTR_SEARCH_HASH_ANALYSIS（默认值17），即该索引已经至少被访问了17次（通过B+树查询）。条件成立开始对页的访问模式进行统计，判断当前访问模式下（n_fields + n_bytes)是否已经被请求了BTR_SEARCH_BUILD_LIMIT（默认值100），条件成立则对页中的记录创建哈希索引。可见，不同的SQL查询语句可能代表不同的访问模式，同一个页面的前一个哈希索引，在下一个查询中可能就失效了。

每个索引内存对象（dict_index_t)都包含数据结构btr_search_t，用于保存索引的查询模式；每个页结构buf_block_struct中也包含页访问模式的信息。

![image03]({{site.baseurl}}/image/20180523/adaptive_hash_struct.png)

对象btr_search_latch是一个全局读写锁，用于保护变量n_field, n_bytes, side等访问模式信息。这是一个频繁访问热点。另外，事务对象trx_t中has_search_latch用于保护事务是否持有自适应哈希；search_latch_timeout用于记录事务通过自适应哈希搜索次数，超过一定阈值后，事务便不再每通过自适应哈希读取一行记录后释放btr_search_latch，减小竞争开销。

创建自适应哈希流程如下：

![image04]({{site.baseurl}}/image/20180523/adaptive_hash_build.png)

每次函数调用btr_cur_search_to_nth_level对B+树索引进行查询时，记录下访问模式，并调用函数btr_search_info_update_slow判断是否需要创建哈希索引。InnoDB首先通过变量hash_analysis判断该索引是否至少被访问了BTR_SEARCH_HASH_ANALYSIS（默认17）次，然后判断访问模式是否已经被请求了BTR_SEARCH_BUILD_LIMIT（默认100）次。

函数btr_search_build_page_hash_index用于对页中的记录创建哈希索引，变量left_side用于控制对于相同键值记录，在最左或最右的记录上创建索引。函数rec_fold生成哈希键值。若存在相同哈希键值记录，InnoDB存储引擎会进行更新，而不是通过链表解决冲突。哈希表创建完成后，根据相同模式对页中记录的查询，会通过btr_search_guess_on_hash实现从哈希表的查询。得到哈希查询结果后，还需要通过btr_search_check_guess判断是否为需要的查询记录。

参考：  
[CSDN: InnoDB Adaptive Hash Index](https://blog.csdn.net/zwleagle/article/details/39010435)  
[CSDN: MySQL AHI实现解析](https://blog.csdn.net/tengxy_cloud/article/details/53884226)  
[CSDN: InnoDB Adaptive Hash Index浅析](https://blog.csdn.net/heizistudio/article/details/72904058)  
[CSDN: InnoDB Adaptive hash index介绍](https://blog.csdn.net/u010725670/article/details/50629995)