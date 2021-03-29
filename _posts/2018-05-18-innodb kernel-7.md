---
title:  “InnoDB 内核阅读笔记（七）- 索引页”
mathjax: true
layout: post
date:   2018-05-18 08:00:12 +0800
categories: database
---

InnoDB存储引擎中，聚集索引（clustering index)的叶子节点中存放着完整的记录，辅助索引（Secondary index)中记录存放着指向聚集索引叶子节点的书签。

# **page文件夹**

文件名称 |  说明
:----|:----
page0cur.cc | 索引页游标，记录的定位，插入，删除
page0page.cc | 索引页的维护
page0zip.cc | 索引页压缩


# **页结构**

MySQL 5.6中，定义了如下类型的页：
```cpp
/** File page types (values of FIL_PAGE_TYPE) @{ */
#define FIL_PAGE_INDEX		17855	/*!< B-tree node */
#define FIL_PAGE_UNDO_LOG	2	/*!< Undo log page */
#define FIL_PAGE_INODE		3	/*!< Index node */
#define FIL_PAGE_IBUF_FREE_LIST	4	/*!< Insert buffer free list */
/* File page types introduced in MySQL/InnoDB 5.1.7 */
#define FIL_PAGE_TYPE_ALLOCATED	0	/*!< Freshly allocated page */
#define FIL_PAGE_IBUF_BITMAP	5	/*!< Insert buffer bitmap */
#define FIL_PAGE_TYPE_SYS	6	/*!< System page */
#define FIL_PAGE_TYPE_TRX_SYS	7	/*!< Transaction system data */
#define FIL_PAGE_TYPE_FSP_HDR	8	/*!< File space header */
#define FIL_PAGE_TYPE_XDES	9	/*!< Extent descriptor page */
#define FIL_PAGE_TYPE_BLOB	10	/*!< Uncompressed BLOB page */
#define FIL_PAGE_TYPE_ZBLOB	11	/*!< First compressed BLOB page */
#define FIL_PAGE_TYPE_ZBLOB2	12	/*!< Subsequent compressed BLOB page */
```

## Page Header

Page Header用来保存页的信息，一共占用56个字节，位于File Header之后（存储管理里的FIL_PAGE_DATA）。

名称 | 大小 |  说明
:----|:----|:----
PAGE_N_DIR_SLOTS | 2 | page directory中槽的数量
PAGE_HEAP_TOP | 2 | 堆中空闲空间的位置偏移
PAGE_N_HEAP | 2 | 堆中的记录数量
PAGE_FREE | 2 | 指向页中空闲空间的位置偏移
PAGE_GARBAGE | 2 | 已删除记录的字节数
PAGE_LAST_INSERT | 2 | 最后插入记录的位置偏移
PAGE_DIRECTION | 2 | 最后插入方向：PAGE_LEFT, PAGE_RIGHT, PAGE_SAME_REC, PAGE_SAME_PAGE, PAGE_NO_DIRECTION
PAGE_N_DIRECTION | 2 | 一个方向连续插入记录的数量
PAGE_N_RECS | 2 | 该页中用户记录的数量
PAGE_MAX_TRX_ID | 8 | 修改当前页的事务ID，仅在辅助索引中使用
PAGE_LEVEL | 2 | 当前页在索引树的位置，0代表叶子节点
PAGE_INDEX_ID | 8 | 索引ID，表示当前页属于哪个索引
PAGE_BTR_SEG_LEAF | 10 | B+树叶子节点所在段的segment header。该值仅在B+树的root页中定义
PAGE_BTR_SEG_TOP | 10 | B+树非叶子节点所在段的segment header。该值仅在B+树的root页中定义

每页有2个伪记录，infinum和supremum。之间的记录称为用户记录。当记录被删除时，PAGE_FREE指向最近删除的记录空间。通过记录头中的next record可以串联得到一个可用空间链表。当页空间不足时，通过函数btr_page_reorganize进行页的重新组织，整理碎片空间。

页记录在物理上可能是无序的。PAGE_LAST_INSERT，PAGE_DIRECTION，PAGE_N_DIRECTION的作用是进行页的分裂操作。根据上述值判断插入方向(升序插入还是降序插入，亦或是无序随机插入)，采用不同的分裂策略。

## Page Directory

精确定位记录在页中的位置需要通过Page Directory。Page Directory由槽（slot）构成，每个槽占用2个字节，指向记录在页中的偏移量。槽根据记录主键逆序存放，因此二分法可以快速定位查询记录。为了提高效率，Page Directory是稀疏的，不是每一个记录都对应一个槽。不在槽中的记录通过链表查询，记录头中的n_owned属性表示槽“拥有“记录数量。一个例子如下

记录ID | heap_no | n_owned
:----|:----|:----
infimum | 0 | 1
supremum | 1 | 3
80 00 00 01 | 2 | 0
80 00 00 02 | 3 | 2
80 00 00 03 | 4 | 0
80 00 00 04 | 5 | 0

# **游标**

游标的作用就是用来定位（lookup）记录，通过查询模式向前或者向后扫描记录。InnoDB存储引擎定义了4种查询模式：
- PAGE_CUR_G 大于
- PAGE_CUR_GE 大小等于
- PAGE_CUR_L 小于
- PAGE_CUR_LE 小于等于

函数page_cur_search_with_match通过查询tuple来定位页中的记录，并通过变量cursor（类型page_cur_t*）返回。查询记录与页中记录的比较通过函数cmp_dtuple_rec_with_match函数完成。

由于记录集合中存在重复记录，因此定位时需要判断查询模式。对于大于等于，定位的是第一条记录值为目标值的记录，因此需要循环直到low和up的距离差为1。示意图如下：

![image01]({{site.baseurl}}/image/20180518/page_cur_search.png)

```cpp
    // slot内部的二分法，如果low_rec和up_rec相差1则结束循环，否则继续
    while (page_rec_get_next_const(low_rec) != up_rec) {
        // 这里并没有除以2作为mid_rec而是简单的取下一行，因为rec是单链表这样显然很容易完成
        mid_rec = page_rec_get_next_const(low_rec);

        ut_pair_min(&cur_matched_fields, &cur_matched_bytes,
                low_matched_fields, low_matched_bytes,
                up_matched_fields, up_matched_bytes);
        // 获得记录的各个字段的偏移数组
        offsets = rec_get_offsets(mid_rec, index, offsets,
                    dtuple_get_n_fields_cmp(tuple),
                    &heap);
        //进行比较，0为相等，1元组大于记录，-1记录大于元组，并且传出field和bytes
        cmp = cmp_dtuple_rec_with_match(tuple, mid_rec, offsets,
                        &cur_matched_fields,
                        &cur_matched_bytes);
        if (UNIV_LIKELY(cmp > 0)) { // 如果元组大于mid_rec记录
low_rec_match:
            low_rec = mid_rec;
            low_matched_fields = cur_matched_fields;
            low_matched_bytes = cur_matched_bytes;

        } else if (UNIV_EXPECT(cmp, -1)) { // 如果元组小于mid_rec记录
up_rec_match:
            up_rec = mid_rec;
            up_matched_fields = cur_matched_fields;
            up_matched_bytes = cur_matched_bytes;
        }
            // 如果元组等于mid_rec 
            else if (mode == PAGE_CUR_G || mode == PAGE_CUR_LE) { // 如果mode是PAGE_CUR_G或者PAGE_CUR_LE
            goto low_rec_match;
        } else { // 如果mode是PAGE_CUR_GE或者PAGE_CUR_L
            goto up_rec_match;
        }
	}

    // 此时low_rec和up_rec相差1
    // 如果mode是PAGE_CUR_G和PAGE_CUR_GE,取up_rec
    // 如果mode是PAGE_CUR_L和PAGE_CUR_LE,取low_rec
    if (mode <= PAGE_CUR_GE) {
		page_cur_position(up_rec, block, cursor);
	} else {
		page_cur_position(low_rec, block, cursor);
	}

	*iup_matched_fields  = up_matched_fields;
	*iup_matched_bytes   = up_matched_bytes;
	*ilow_matched_fields = low_matched_fields;
	*ilow_matched_bytes  = low_matched_bytes;
```

变量iup_matched_fields，iup_matched_bytes，ilow_matched_fields和ilow_matched_bytes用来返回进行二叉查找算法记录比较时，左右各已经匹配的字段数量和字节数。

上图查询>=3的记录，最后返回变量值iup_matched_fields=1， iup_matched_types=0， ilow_matched_fields=0， ilow_matched_bytes=3。在第二个步骤元组大于mid_rec记录，代码走到low_rec_match标签，ilow_matched_fields=0，ilow_matched_bytes=3；第三个步骤，元组等于mid_rec记录，且查询模式等于PAGE_CUR_GE，代码回到up_rec_match标签，iup_matched_fields=1，iup_matched_bytes=0。

## 插入记录

函数page_cur_insert_rec_low用来插入数据，变量current_rec指向插入之前的记录，变量rec为待插入的物理记录。插入的索引查询模式为PAGE_CUR_LE。插入操作完成后，对page directory平衡进行维护。

## 删除记录

函数page_cur_delete_rec将物理记录“彻底”删除，即将记录占用空间放入PAGE_FREE队列队首，同时更新PAGE_LAST_INSERT，PAGE_DIRECTION，PAGE_N_DIRECTION。然后对page directory平衡进行维护。

**对索引页的并发不是在page模块控制，而是由上层调用模块btr中负责控制**

参考：  
[简书：InnoDB中查询定位方法](https://www.jianshu.com/p/0cdd573a8232)