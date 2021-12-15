---
title: "MySQL锁的实现原理"
date: 2021-12-10T00:11:38+08:00
draft: false
categories: ["MySQL"]
---

## 一、锁类型
### 1.1、自增锁
> *An AUTO-INC lock is a special table-level lock taken by transactions inserting into tables with AUTO_INCREMENT columns. In the simplest case, if one transaction is inserting values into the table, any other transactions must wait to do their own inserts into that table, so that rows inserted by the first transaction receive consecutive primary key values.*

自增锁是一种特殊的**表级别锁**（table-level lock），专门针对事务插入AUTO_INCREMENT类型的列。

最简单的情况，如果一个事务正在往表中插入记录，所有其他事务的插入必须等待，以便第一个事务插入的行，是连续的主键值。

### 1.2、共享锁/排他锁

Innodb中实现了标准的行级锁，也就是共享锁/排他锁

#### 2.1、共享锁

目标：事务拿到某一行记录的共享S锁，才能**读取**这一行，实现读读可并行

语句：select xxx from ... share in mode;

#### 2.2、排他锁

目标：事务拿到某一行记录的排他X锁，才可以**修改**或**删除**这一行，实现读写/写写互斥

语句：select xxx from ... for update

### 1.3、意向锁

> InnoDB supports multiple granularity locking which permits coexistence of row-level locks and locks on entire tables. To make locking at multiple granularity levels practical, additional types of locks called intention locks are used. Intention locks are table-level locks in InnoDB that indicate which type of lock (shared or exclusive) a transaction will require later for a row in that table. There are two types of intention locks used in InnoDB (assume that transaction T has requested a lock of the indicated type on table t):
> Intention shared (IS): Transaction T intends to set S locks on individual rows in table t.
> Intention exclusive (IX): Transaction T intends to set X locks on those rows.
>
> Before a transaction can acquire an S lock on a row in table t, it must first acquire an IS or stronger lock on t. Before a transaction can acquire an X lock on a row, it must first acquire an IX lock on t.
>
> The main purpose of IX and IS locks is to show that someone is locking a row, or going to lock a row in the table.

意向锁分为意向共享锁（IS锁）和意向排它锁（IX锁）。根据官方说明，事务请求S锁前先获取IS锁，事务请求X锁前先获取IX锁。

意向锁是指，未来某个时刻，事务可能要加共享/排它锁，先提前声明一个意向。

- 意向共享锁：它预示，事务有意向想对表中的某些行加共享S锁
- 意向排它锁：它预示，事务有意向对表中的某些行加排它X锁

意向锁间不互斥，但与S、X锁可能存在互斥关系

- S与IS兼容，S与IX互斥、
- X与IS互斥、X与IX互斥

### 1.4、插入意向锁

> Insert Intention Lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap.

对已有数据行的**修改与删除**，必须加强互斥锁X锁，那对于**数据的插入**，是否还需要加这么强的锁，来实施互斥呢？插入意向锁，孕育而生。 

**插入意向锁**，是间隙锁(Gap Locks)的一种（所以，也是实施在索引上的），它是专门针对insert操作的。它的玩法是多个事务，在同一个索引，同一个范围区间插入记录时，**如果插入的位置不冲突，不会阻塞彼此**。

本质上，两个写动作，分别写不同的行，通过插入意向锁试探是否有冲突，若冲突那就gap锁开始作用，若无冲突那就实现插入并发，提升性能。

- InnoDB使用**共享锁**，可以提高读读并发；
- 为了保证数据强一致，InnoDB使用强**互斥锁**，保证同一行记录修改与删除的串行性；
- InnoDB使用**插入意向锁**，可以提高插入并发；

### 1.5、记录锁（Record Locks）

**说明：该论述的前提是，在可重复读隔离下进行事务动作**

> If the table has no PRIMARY KEY or suitable UNIQUE index, InnoDB internally generates a hidden clustered index on a synthetic column containing row ID values. The rows are ordered by the ID that InnoDB assigns to the rows in such a table. The row ID is a 6-byte field that increases monotonically as new rows are inserted. Thus, the rows ordered by the row ID are physically in insertion order.

针对主键索引在内的唯一索引记录上加锁（若表无任何索引，则会创建一个隐藏主键索引），需注意的是是索引而不是记录，以阻止其他事务插入，更新，删除某一行。

**记录锁的执行前提是执行当前读，而不是快照读。其中，间隙锁和临键锁也同理，下面论述也遵循这个原则。**

- 快照读：select ... from xxx
- 当前读：update/delete/select ... from xxx for update

### 1.6、间隙锁（Gap Locks）

**说明：该论述的前提是，在可重复读隔离下进行事务动作**

**间隙锁**，它封锁索引记录中的间隔，或者第一条索引记录之前的范围，又或者最后一条索引记录之后的范围。

存在于**非唯一索引**中，是Innodb在提交下为了解决幻读问题时引入的锁机制，间隙锁不互斥。

1. 加锁的基本单位是（next-key lock）,他是前开后闭原则
2. 插叙过程中访问的对象会增加锁
3. 索引上的等值查询
   1. 给唯一索引加锁的时候，next-key lock升级为行锁
   2. 向右遍历时最后一个值不满足查询需求时，next-key lock 退化为间隙锁
4. 唯一索引上的范围查询会访问到不满足条件的第一个值为止

### 1.7、临键锁（NextKey Locks）

**说明：该论述的前提是，在可重复读隔离下进行事务动作**

> A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record.
>
> By default, InnoDB operates in REPEATABLE READ transaction isolation level and with the innodb_locks_unsafe_for_binlog system variable disabled. In this case, InnoDB uses next-key locks for searches and index scans, which prevents phantom rows。

临键锁，是记录锁和间隙锁的组合，它的封锁范围，既包括索引记录，又包含索引区间。

默认情况下，innodb使用next-key locks来锁定记录。但当查询的索引含有唯一属性的时候，Next-Key Lock 会进行优化，将其降级为Record Lock，即仅锁住索引本身，不是范围。

假设有个数据库表 test(id PK, uname)

| id(primary key) | name (key) | value |
| --------------- | ---------- | ----- |
| 1               | 1          | 1     |
| 5               | 5          | 5     |
| 10              | 10         | 10    |
| 15              | 15         | 15    |

PK存在以下潜在的临键锁

```
(-∞, 1]
(1, 5]
(5, 10]
(10, 15]
```

## 二、案例分析

### 2.1、唯一索引等值查询且有结果

| 事务A                                                        | 事务B                                         | 事务C                                        |
| ------------------------------------------------------------ | --------------------------------------------- | -------------------------------------------- |
| SELECT * FROM `test` WHERE `id` = 5 FOR UPDATE;<br>SELECT SLEEP(60); |                                               |                                              |
|                                                              | 正常<br> INSERT INTO `test` VALUES (3, 3, 3); | 正常<br>INSERT INTO `test` VALUES (8, 8, 8); |
| Commit;                                                      |                                               |                                              |
| 当使用等值查询且存在结果时，只使用记录锁，不产生间隙锁。     |                                               |                                              |

### 2.2、唯一索引等值查询不存在结果

| 事务A                                                        | 事务B                                         | 事务C                                                  |
| ------------------------------------------------------------ | --------------------------------------------- | ------------------------------------------------------ |
| /* 开启事务 */ <br>BEGIN; <br>/* 查询并加记录锁 */<br> SELECT * FROM `test` WHERE `id` = 7 FOR UPDATE; <br> SELECT SLEEP(60); |                                               |                                                        |
|                                                              | 阻塞<br> INSERT INTO `test` VALUES (8, 8, 8); | 正常 <br>INSERT INTO `test` VALUES (10, 'wangle', 10); |
| commit;                                                      |                                               |                                                        |

当使用等值查询且不存在结果时，使用临键锁。

id=7不存在记录，即加临键锁 (5, 10]

由于数据是等值查询，并且范围的最后数据id=10不满足要求，退化为间隙锁(5,10)

id = 8 写入失败，写id=10 成功

### 2.3、唯一索引范围查询

#### 2.3.1、范围开区间

| 事务A                                                        | 事务B                                        | 事务C                                                     |
| ------------------------------------------------------------ | -------------------------------------------- | --------------------------------------------------------- |
| BEGIN;<br> SELECT * FROM `test` WHERE id < 13 for update;<br> SELECT SLEEP(60); |                                              |                                                           |
|                                                              | #阻塞<br> insert into test values(14,14,14); | #阻塞 <br>update test set value = value +1 where id = 15; |
| commit;                                                      |                                              |                                                           |

命中区间为(-∞, 13)，13命中的临键锁区间为(10, 15]，因为不是等值查询，所以不会降级为间隙锁(10, 15)，综合看锁止的区间为(-∞, 15]；



| 事务A                                                        | 事务B                                        | 事务C                                                     |
| ------------------------------------------------------------ | -------------------------------------------- | --------------------------------------------------------- |
| BEGIN;<br>  SELECT * FROM `test` WHERE id > 13 for update; <br>SELECT SLEEP(60); |                                              |                                                           |
|                                                              | #阻塞<br> insert into test values(11,11,11); | #正常 <br>update test set value = value +1 where id = 10; |
| commit;                                                      |                                              |                                                           |

命中的区间为(13, +∞)，13命中的临键锁区间为(10, 15]，所以综合看锁止的区间为(10, +∞)；

#### 2.3.2、范围闭区间

| 事务A                                                        | 事务B                                     | 事务C                                                     |
| ------------------------------------------------------------ | ----------------------------------------- | --------------------------------------------------------- |
| BEGIN;<br>SELECT * FROM `test` WHERE id >= 10 for update;<br> SELECT SLEEP(60); |                                           |                                                           |
|                                                              | #正常<br> insert into test values(9,9,9); | #阻塞<br> update test set value = value +1 where id = 10; |
| commit;                                                      |                                           |                                                           |

命中的区间为(10, +∞)，因为条件包含等于，所以左区间升级为闭区间，[10, +∞)。



| 事务A                                                        | 事务B                                     | 事务C                                        | 事务D                                                |
| ------------------------------------------------------------ | ----------------------------------------- | -------------------------------------------- | ---------------------------------------------------- |
| BEGIN;<br>SELECT * FROM `test` WHERE id <= 10 for update; <br>SELECT SLEEP(60); |                                           |                                              |                                                      |
|                                                              | #阻塞<br> insert into test values(3,3,3); | #阻塞<br> insert into test values(14,14,14); | #阻塞update test set value = value +1 where id = 15; |
| commit;                                                      |                                           |                                              |                                                      |

id <= 10表示的范围为(-∞, 10]，因为是范围查询，所以会遍历下一个区间(10,15]，因为是范围查询，即使15未命中也不会退化为开区间。综合看锁定的范围为(-∞, 15]

### 2.4、普通索引等值查询存在结果

| 事务A                                                        | 事务B                                     | 事务C                                     | 事务D                                                      |
| ------------------------------------------------------------ | ----------------------------------------- | ----------------------------------------- | ---------------------------------------------------------- |
| BEGIN;<br> SELECT * FROM `test` WHERE name = 5 for update;<br> SELECT SLEEP(60); |                                           |                                           |                                                            |
|                                                              | #阻塞<br> insert into test values(3,3,3); | #阻塞<br> insert into test values(9,9,9); | #正常<br>update test set value = value +1 where name = 10; |
| commit;                                                      |                                           |                                           |                                                            |

普通索引等值查询会直接使用间隙锁，并且会向右遍历一个区间。

name = 5 命中(1, 5]，向右遍一个区间为(5,10]，因为是等值查询且10不满足条件，所以降级为(5,10)，综合来看锁止的空间为(1,10)，所以事务B和C会被阻塞，事务D正常。

### 2.5、普通索引等值查询不存在结果

| 事务A                                                        | 事务B                                     | 事务C                                     | 事务D                                                      |
| ------------------------------------------------------------ | ----------------------------------------- | ----------------------------------------- | ---------------------------------------------------------- |
| BEGIN;<br> SELECT * FROM `test` WHERE name = 6 for update;<br>SELECT SLEEP(60); |                                           |                                           |                                                            |
|                                                              | #正常<br> insert into test values(3,3,3); | #阻塞<br> insert into test values(9,9,9); | #正常<br>update test set value = value +1 where name = 10; |
| commit;                                                      |                                           |                                           |                                                            |

name = 6 不存在记录，所以命中的区间为(5,10]，因为10 不满足条件，所以降级为(5,10)。事务B和D正常，事务C被阻塞。

### 2.6、普通索引范围查询

| 事务A                                                        | 事务B                                    | 事务C                                        | 事务D                                                       |
| ------------------------------------------------------------ | ---------------------------------------- | -------------------------------------------- | ----------------------------------------------------------- |
| BEGIN;<br> SELECT * FROM `test` WHERE name < 13 for update;<br> SELECT SLEEP(60); |                                          |                                              |                                                             |
|                                                              | #阻塞<br>insert into test values(3,3,3); | #阻塞<br/>insert into test values(14,14,14); | #阻塞<br/>update test set value = value +1 where name = 15; |
| commit;                                                      |                                          |                                              |                                                             |

name < 13 表示的范围为 (-∞, 13)，因为13 位于(10,15]区间，并且非等值查询，即使15未命中也不会降级为开区间，综合看锁止的范围为(-∞, 15]。



| 事务A                                                        | 事务B                                    | 事务C                                         | 事务D                                                       |
| ------------------------------------------------------------ | ---------------------------------------- | --------------------------------------------- | ----------------------------------------------------------- |
| BEGIN;<br/> SELECT * FROM `test` WHERE name <= 10 for update;<br/>SELECT SLEEP(60); |                                          |                                               |                                                             |
|                                                              | 阻塞<br/>insert into test values(3,3,3); | #阻塞<br/> insert into test values(14,14,14); | #阻塞<br/>update test set value = value +1 where name = 15; |
| commit;                                                      |                                          |                                               |                                                             |

name <=10 表示的范围为(-∞, 10]，10命中的区间为(5,10]；因为是范围查询，所以会向右遍历一个区间(10,15]，并且即使15未命中也不会退化为(10,15)；综合来看，命中的区间为(-∞, 15]。所以事务B、C、D都会被阻塞。



| 事务A                                                        | 事务B                                         | 事务C                                                        |
| ------------------------------------------------------------ | --------------------------------------------- | ------------------------------------------------------------ |
| BEGIN;<br> SELECT * FROM `test` WHERE name > 13 for update; <br/>SELECT SLEEP(60); |                                               |                                                              |
|                                                              | #阻塞<br/> insert into test values(11,11,11); | #正常<br/> update test set value = value +1 where name = 10; |
| commit;                                                      |                                               |                                                              |

name > 13 表示的范围为(13, +∞)，13命中的间隙范围为(10,15]，所以综合看，锁定的范围为(10, +∞)，事务B阻塞，事务C正常。

【分析总结】

|                | 唯一索引                                                     | 普通索引                                                     |
| :------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| 等值查询有结果 | 记录锁                                                       | 间隙锁，取值的前后相邻两个区间                               |
| 等值查询无结果 | 临键锁，取值所在的区间                                       | 临键锁，取值所在的区间                                       |
| < x  \|\| <=x  | (-∞, y]<br>x在表中存在时，y为x所在区间向右一个区间的上界<br>x在表中不存在时，y为x所在区间的上界 | (-∞, y]<br>x在表中存在时，y为x所在区间向右一个区间的上界<br>x在表中不存在时，y为x所在区间的上界 |
| >x             | (y, +∞)<br>x在表中存在时，y=x<br>x在表中不存在时，y为x所在区间的下界 | (y, +∞)<br>x在表中存在时，y=x<br>x在表中不存在时，y为x所在区间的下界 |
| >=x            | [y, +∞)<br>x在表中存在时，y=x<br>x在表中不存在时，y为x所在区间的下界 | [y, +∞)<br>x在表中存在时，y=x<br>x在表中不存在时，y为x所在区间的下界 |

## 三、死锁分析

> INSERT sets an exclusive lock on the inserted row. This lock is an index-record lock, not a next-key lock (that is, there is no gap lock) and does not prevent other sessions from inserting into the gap before the inserted row.
>
> Prior to inserting the row, a type of gap lock called an insert intention gap lock is set. This lock signals the intent to insert in such a way that multiple transactions inserting into the same index gap need not wait for each other if they are not inserting at the same position within the gap. Suppose that there are index records with values of 4 and 7. Separate transactions that attempt to insert values of 5 and 6 each lock the gap between 4 and 7 with insert intention locks prior to obtaining the exclusive lock on the inserted row, but do not block each other because the rows are nonconflicting.
>
> If a duplicate-key error occurs, a shared lock on the duplicate index record is set. This use of a shared lock can result in deadlock should there be multiple sessions trying to insert the same row if another session already has an exclusive lock.

简单说就是insert执行时会给目标行加排他锁，是记录锁；不过在insert执行前，会先加insertion intention gap lock，意向插入锁，是间隙锁。

如果发生了唯一键冲突错误，那么将会在重复的索引记录上加读锁。当有多个session同时插入相同的行记录时，如果另外一个session已经获得该行的排它锁，那么将会导致死锁。

insert into xxx values(...) on duplicate key update .....  产生死锁的原因分析

| 事务A                                                   | 事务B                                                      |
| ------------------------------------------------------- | ---------------------------------------------------------- |
| acquire X lock;duplicate;release X lock;acquire S lock; |                                                            |
|                                                         | acquire X lock;   duplicate;release X lock;acquire S lock; |
| prepare update;try acqure X lock;                       | prepare update;try acqure X lock;                          |
| waiting S lock from B                                   | waiting S lock from A                                      |

## 参考资料

- [MySQL锁类型@架构师之路](https://mp.weixin.qq.com/s/f4_o-6-JEbIzPCH09mxHJg)


