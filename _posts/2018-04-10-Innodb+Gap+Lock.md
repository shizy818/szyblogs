---
title:  “Innodb Gap Lock”
mathjax: true
layout: post
date:   2018-04-10 08:00:12 +0800
categories: database
---

在Innodb的锁机制里，有以下几种类型：
- 记录锁（Record Lock），仅仅锁住索引记录的一行
- 区间锁（Gap Lock），对索引记录之间的间隙中加锁，或者是在某一条索引记录之前或者之后加锁
- 插入意向锁（Insert Intention Lock），插入操作中使用的特殊区间锁
- Next-Key Lock，锁住索引记录本身和索引之前的间隙，左开右闭区间

区间锁主要是要解决可重复读模式下的幻读问题。


兼容矩阵：

 请求\持有 | Gap | Insert Intention | Record | Next-Key
:----:|:----:|:----:|:----:|:----:
Gap | 兼容 | 兼容 | 兼容 | 兼容
Insert Intention | 冲突 | 兼容 | 兼容 | 冲突
Record | 兼容 | 兼容 | 冲突 | 冲突
Next-Key | 兼容 | 兼容 | 冲突 | 冲突

# 上锁条件

区间锁和Next-Key锁首先都要求REPEATABLE READ隔离级别。

区间锁：
- 非唯一索引（范围查询或者等值查询）
- 唯一索引 （范围查询）

Next-Key锁：
- select .. from .. lock in share mode: 在扫描到的任何索引记录上加共享next-key lock，主键聚集索引加排它锁
- select .. from .. for update: 在扫描到的任何索引记录上加排它next-key lock，主键聚集索引加排它锁
- update .. where / delete from where: 在扫描到的任何索引记录上加next-key lock，主键聚集索引加排它锁

# 测试

现有如下User表：

id | name (非唯一索引) | gender
:----:|:----:|:----:
1 | jiangcl | F
3 | shizy   | M
5 | shizy   | M
6 | zuoyu   | M
9 | chenww  | F

通过`SET GLOBAL innodb_status_output_locks=ON`命令收集lock状态信息，然后`SHOW ENGINE INNODB STATUS\G`显示。

- `SELECT * FROM user WHERE name = 'shizy' FOR UPDATE;`

```shell
---TRANSACTION 4356, ACTIVE 10 sec
4 lock struct(s), heap size 1136, 5 row lock(s)
MySQL thread id 6, OS thread handle 139811657422592, query id 22 localhost root starting
show engine innodb status
### 意向锁
TABLE LOCK table `test`.`user` trx id 4356 lock mode IX
### 非唯一索引Next-Key锁
RECORD LOCKS space id 36 page no 4 n bits 80 index idx_name of table `test`.`user` trx id 4356 lock_mode X
### (‘jiangcl’, 'shizy']
Record lock, heap no 3 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 5; hex 7368697a79; asc shizy;;
 1: len 4; hex 80000003; asc     ;;
### ('shizy', 'shizy']
Record lock, heap no 4 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 5; hex 7368697a79; asc shizy;;
 1: len 4; hex 80000005; asc     ;;

### 主键索引记录锁
RECORD LOCKS space id 36 page no 3 n bits 80 index PRIMARY of table `test`.`user` trx id 4356 lock_mode X locks rec but not gap
### 主键3
Record lock, heap no 3 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000003; asc     ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex a70000011b0110; asc        ;;
 3: len 5; hex 7368697a79; asc shizy;;
 4: len 1; hex 4d; asc M;;

### 主键5
Record lock, heap no 4 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
 1: len 6; hex 00000000090c; asc       ;;
 2: len 7; hex aa0000011e0110; asc        ;;
 3: len 5; hex 7368697a79; asc shizy;;
 4: len 1; hex 4d; asc M;;

### 非唯一索引区间锁
RECORD LOCKS space id 36 page no 4 n bits 80 index idx_name of table `test`.`user` trx id 4356 lock_mode X locks gap before rec
### ('shizy', 'zuoyu')
Record lock, heap no 5 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 5; hex 7a756f7975; asc zuoyu;;
 1: len 4; hex 80000006; asc     ;;
```

- `SELECT * FROM user WHERE id > 2 AND id <= 6 FOR UPDATE;`

```shell
---TRANSACTION 4357, ACTIVE 4 sec
2 lock struct(s), heap size 1136, 4 row lock(s)
MySQL thread id 6, OS thread handle 139811657422592, query id 26 localhost root starting
show engine innodb status
### 意向锁
TABLE LOCK table `test`.`user` trx id 4357 lock mode IX
### 主键索引Next-Key锁
RECORD LOCKS space id 36 page no 3 n bits 80 index PRIMARY of table `test`.`user` trx id 4357 lock_mode X
### (1, 3]
Record lock, heap no 3 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000003; asc     ;;
 1: len 6; hex 000000000907; asc       ;;
 2: len 7; hex a70000011b0110; asc        ;;
 3: len 5; hex 7368697a79; asc shizy;;
 4: len 1; hex 4d; asc M;;

### (3, 4]
Record lock, heap no 4 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
 1: len 6; hex 00000000090c; asc       ;;
 2: len 7; hex aa0000011e0110; asc        ;;
 3: len 5; hex 7368697a79; asc shizy;;
 4: len 1; hex 4d; asc M;;

### (4, 6]
Record lock, heap no 5 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000006; asc     ;;
 1: len 6; hex 00000000090d; asc       ;;
 2: len 7; hex ab0000011f0110; asc        ;;
 3: len 5; hex 7a756f7975; asc zuoyu;;
 4: len 1; hex 4d; asc M;;

### (6, 9]
Record lock, heap no 6 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000009; asc     ;;
 1: len 6; hex 000000000930; asc      0;;
 2: len 7; hex 24000001380551; asc $   8 Q;;
 3: len 6; hex 6368656e7777; asc chenww;;
 4: len 1; hex 46; asc F;;
```

此处有个行为比较奇怪，查询范围是id = (2, 6], 但是Mysql拿了(6, 9]的Next-Key锁，因此如果对主键9
记录作修改或者插入id=8的记录是无法成功的。个人觉得不应该拿这个Next-Key锁。
```shell
---TRANSACTION 4359, ACTIVE 748 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 4 row lock(s)
MySQL thread id 7, OS thread handle 139811657221888, query id 48 localhost root updating
update user set gender = 'F' where id =9
------- TRX HAS BEEN WAITING 6 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 36 page no 3 n bits 80 index PRIMARY of table `test`.`user` trx id 4359 lock_mode X locks rec but not gap waiting
Record lock, heap no 6 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000009; asc     ;;
 1: len 6; hex 000000000930; asc      0;;
 2: len 7; hex 24000001380551; asc $   8 Q;;
 3: len 6; hex 6368656e7777; asc chenww;;
 4: len 1; hex 46; asc F;;

------------------
TABLE LOCK table `test`.`user` trx id 4359 lock mode IX
RECORD LOCKS space id 36 page no 3 n bits 80 index PRIMARY of table `test`.`user` trx id 4359 lock_mode X locks rec but not gap waiting
Record lock, heap no 6 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000009; asc     ;;
 1: len 6; hex 000000000930; asc      0;;
 2: len 7; hex 24000001380551; asc $   8 Q;;
 3: len 6; hex 6368656e7777; asc chenww;;
 4: len 1; hex 46; asc F;;
```

# Insert死锁

Mysql在插入数据时，有一种很简单的死锁情况。

Transaction A |  Transaction B | Transaction C
:----|:----|:----
begin; | |
insert into user values(10, 'chenzj', 'M'); | |
 | begin; |
 | insert into user values(10, 'chenzj', 'M'); |
 | | begin;
 | | insert into user values(10, 'chenzj', 'M');
rollback; | |

```shell
------------------------
LATEST DETECTED DEADLOCK
------------------------
2018-07-07 18:04:48 0x7f287026f700
*** (1) TRANSACTION:
TRANSACTION 4361, ACTIVE 11 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 7, OS thread handle 139811657221888, query id 62 localhost root update
insert into user values(10, 'chenzj', 'M')
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 36 page no 3 n bits 80 index PRIMARY of table `test`.`user` trx id 4361 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) TRANSACTION:
TRANSACTION 4362, ACTIVE 4 sec inserting
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 8, OS thread handle 139811657021184, query id 64 localhost root update
insert into user values(10, 'chenzj', 'M')
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 36 page no 3 n bits 80 index PRIMARY of table `test`.`user` trx id 4362 lock mode S
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 36 page no 3 n bits 80 index PRIMARY of table `test`.`user` trx id 4362 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
```

加锁流程如下：

TA加隐式锁锁 | |
| TB将隐式锁转换为显式锁`LOCK_REC | LOCK_X | LOCK_REC_NOT_GAP`
| 主键冲突，TB等待S锁 `LOCK_REC | LOCK_S | LOCK_ORDINARY` |
| | TC等待S锁 `LOCK_REC | LOCK_S | LOCK_ORDINARY`
TA回滚，释放X记录锁 | |
| TB创建S锁 `LOCK_REC | LOCK_S | LOCK_ORDINARY` |
| | TC创建S锁 `LOCK_REC | LOCK_S | LOCK_ORDINARY` |
| TB请求X插入意向锁 `LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION`
| | TC请求X插入意向锁 `LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION`

TB和TC请求插入意向锁均需要等待对方释放Next-Key锁，因此造成死锁。值得思考的一个问题是为什么Mysql不引入
更新锁（U锁）？

参考:  
[Mysql加锁过程](http://www.cnblogs.com/crazylqy/p/7773492.html)  
[Mysql Insert锁机制](yeshaoting.cn/article/database/mysql insert锁机制/)  
[有趣的死锁](https://www.cnblogs.com/sunss/p/3166550.html)
