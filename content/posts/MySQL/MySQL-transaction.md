---
title: "MySQL事务实现原理"
date: 2021-12-09T11:07:23+08:00
draft: false
categories: ["MySQL"]
---

# MySQL事务

### 1、事务是什么？
事务通常指的是逻辑上的一组操作，要么全部执行成功，要么全部执行失败。总体来说，他们具备ACID四大特性，分别是原子性（Atomioc）、一致性（Consistency）、隔离性（Isolation）和持久性（Durability）。其中，事务的隔离型由锁和MVCC机制实现啊，原子性和持久性由RedoLog实现，一致性由UndoLog实现的。

- 原子性：事务的所有操作要么全部成功，要么全部失败。
- 一致性：事务执行之前和执行之后，数据始终处于一致的状态。
- 隔离型：并行执行的两个事务互相不受干扰。
- 持久性：事务提交之后，此事务对数据的更改操作被持久化到数据库中，并且不会被回滚。

### 2、事务并发会产生什么问题？
并发问题是程序无法避免的问题，核心是并发事务对同一临界资源进行操作，不进行管控就会产生不一致。

举个例子：假设有一张数据表，user(uid, account)，实际数据为user（123, 100）
**A、问题1：脏读**

```Bash
1. 事务A：insert into user value(456, 200);  未提交
2. 事务B：select account from user where uid = 456; 读到account的值为200

事务B中读到了事务A未提交数据，说明是脏读。
```


**B、问题2: 不可重复读**
```Bash
1. 事务A：update account = account + 100 where uid = 123
2. 事务B：select account from user where uid = 123，该处读到的acount = 100（A未提交）
3. 事务A：commit
4. 事务B：select account from user where uid = 123，该处读到的acount = 200（A已提交）

事务B两次读同一数据，读到前后结果分别是100、200，说明是不可重复读。
```

**C、问题3: 幻读**
```Bash
1. 事务A：select * from user;   // 查询结果: user(123, 100)
2. 事务B：insert into user value(456, 200); commit;
3. 事务A: select * from user;  // 查询结果：user(123, 100)
4. 事务A：update user set account = 300 where uid = 456; commit;  // 刚刚查询结果还不存在uid=456，更新居然成功了！
5. 事务A: select * from user;  // 查询结果：user(123, 100)、user(456, 300)
```

从例子看，都是已提交的事务A对事务B造成了影响。不可重复读和幻读的区别是什么呢？
- 不可重复读：主要是针对修改和删除动作，两次读的结果是不一样的，被称为不可重读（变化）。
- 幻读：主要是针对插入动作，第一次读不存在，第二次读存在，被称为幻读（有无）。

### 3、如何解决并发事务产生的问题？
并发问题在程序中通常控制手段有两种：悲观锁（互斥锁、读写锁）、乐观锁（CAS + 数据多版本）。

#### 3.1、在并发控制中，锁机制是如何演进的？
我们知道，并发问题的核心是数据竞争，解决办法是临界资源管控(锁)。然而，锁机制演进的过程在本质上是**性能**和**安全**的折中。我们在应用程序中，解决并发问题通常手段有如下几种。

- 互斥锁：最简单粗暴的办法是互斥锁，读写全部都串行化。它的优点是安全，缺点是性能差。
- 读写锁：按理说读读是对数据一致性不会产生影响，于是读写锁产生实现读读并行，读写、写写串行，提升性能。
- CAS+数据多版本：CAS是自旋锁摆脱锁机制，但是CAS会存在ABA问题，于是通过数据多版本来解决ABC问题保障安全。同时，实现读写并行，进一步提升性能。

名次解释：CAS（Compare And Swap）、ABA问题（百度一下）

总体思路，简单总结一下。
- 互斥锁：无论读还是写全部串行执行。
- 读写锁：读读并行执行，读写、写写串行执行。
- 数据多版本：读写并行执行。

#### 3.2、MySQL中是如何解决并发问题的？
MySQL中是通过隔离性来保证，隔离性的底层实现是通过锁和MVCC（数据多版本）保证。从总体思路说，它的解决办法思路跟通常手段是一致的。谈到隔离性就不得不谈到隔离级别，事务中存在四种隔离级别分别是读提交、读已提交、可重复读、串行化。

##### 3.2.1、这几种隔离级别是什么含义呢？

举个例子：假设有一张数据表，user(uid, account)，实际数据为user（123, 100）
**A、读未提交**

1. 事务A：update account = account + 100 where uid = 123
2. 事务B：select account from user where uid = 123，该处读到的account = 200（A未提交）

点评：事务A修改数据且提交，事务B马上可见，存在脏读。

**B、读提交**
1. 事务A：update account = account + 100 where uid = 123
2. 事务B：select account from user where uid = 123，该处读到的acount = 100（A未提交）
3. 事务A：commit
4. 事务B：select account from user where uid = 123，该处读到的acount = 200（A已提交）

点评：事务A修改数据且提交，事务B马上可见，存在不可重复读。

**C、可重复读**
简单说，事务在未提交前查询的同一条数据，无论读多少遍都不会改变。
1. 事务B：select account from user where uid = 123，该处读到的acount = 100
2. 事务A：pdate account = account + 100 where uid = 123；commit； （更新且提交）
3. 事务B：select account from user where uid = 123，该处读到的acount = 100（A提交不影响B）

点评：事务A修改数据无论是否提交，事务B都不可见。

**D、串行化**
简单说，前一个事务没有提交事务前，下一个事务不允许进行数据操作，不过多赘述。

##### 3.2.2、不同隔离级别分别能解决那些事务并发问题（脏读、不可重复读、幻读）？
| 事务隔离级别 | 脏读 | 不可重复读 | 幻读 |
| :----:  | :----:   | :----: | :----: |
| 读未提交 | 可能   | 可能 | 可能 |
| 读已提交 | 不可能 | 可能 | 可能 |
| 可重复读 | 不可能 | 不可能 | 可能 |
| 串行化   | 不可能 | 不可能 | 不可能 |

- 读未提交：普通select不加锁
- 串行化：普通select隐式加锁，普通select全部转换成加锁select（select ... in share mode）
- 可重复读（RR）：普通select为快照读，update/delete/加锁select（select ... in share mode / for update）为当前读（行锁）
     - 唯一索引查询条件（记录锁）、范围查询（间隙锁）
- 读已提交（RC）：普通select为快照读，update/delete/加锁select（select ... in share mode / for update）为当前读（行锁）
     - 唯一索引查询条件（记录锁），会产生幻读现象

##### 3.2.3、Mysql是如何在可重复读隔离级别解决幻读问题的？
众所周知，数据多版本只能解决可重复读问题（行记录多版本），并不能控制记录新增产生的幻读问题。MySQL中解决幻读问题，主要是通过间隙锁和记录锁。

间隙锁：本质上是事务未结束前，不允许其他记录新增记录。其中，间隙锁住要作用于范围查询。
```Bash
假设有个数据库表,user(id, uid, account)，存在如下记录
user(1, 111, 100)
user(2, 222, 200)
user(20, 333, 300)
user(24, 444, 400)

该数据表存在间系：[3,20)、[20,24),[25,+∞)

事务：select * from user id > 20; // 这时候间隙[21,24)、[25, +∞]都会被间隙锁锁住，24会被记录锁锁住
```

##### 3.2.4、MVCC（Multi Version Concurrent Control）是如何实现的？
###### 3.2.4.1、**MVVC被称为多版本并发控制，什么是数据多版本？**

例3.2-1：假设有张数据表，数据表为user(uid, account)，初始化数据为user(123, 100)

```Bash
执行SQL语句: 
start;
update user set account = 200 where uid = 123;
update user set account = 300 where uid = 123;
update user set account = 400 where uid = 123;
commit;
```

按照正常思想，执行完SQL的数据存储结构  
- user(123, 400)

按照数据多版本思想，执行玩SQL的数据存储结构
- user(123, 100, V1)
- user(123, 200, V2)
- user(123, 300, V3)
- user(123, 400, V4)

**注意：这里的快照是基于每一行的行数据变更快照。**

据上总结，数据多版本就是将数据做冗余，存储每一个时刻行记录变更的镜像。那么，事务之间的数据镜像无依赖且能找到历史数据版本，从而实现数据读写并发、事务回滚。

###### 3.2.4.2、**MySQL的MVCC机制底层是如何实现的？**

**行记录的隐藏列**

假设我们来设计数据多版本，按照最简单思路（例3.2-1），不考虑资源限制条件下直接在磁盘存多份数据就能实现这种效果，但显然存在存储资源问题，故不会被采用。

那在Innodb存储引擎中如何实现做的呢？首先，在数据库中有个概念叫做隐藏列，也就是每一行数据都有隐藏列数据，它们分别是TRX_ID、ROW_ID、ROLLBACK_POINTER

- TRX_ID：更新该行数据的事务ID
- ROW_ID：该行的唯一标识ID（当无主键时存在）
- ROLLBACK_POINTER：回滚指针，它指向上一个版本，我们把指向的地址回滚段中undolog的指针

<div align=left><img src="./../../static/imgs/MySQL-MVCC.png" alt="MVCC存储结构" />
**undolog**

**上面提到的undolog，什么意思？**undolog是食事务未提时，会将事务旧版本的数据存放于undolog日志里，当事务回滚或数据崩溃时，可以利用undo日志，撤销未提交事务对数据的影响。

**undolog存什么？**
- insert：undolog日志存储新数据的ROW_ID(PK)，回滚时删除即可
- delete/update：undolog存储旧版本数据的行记录，回滚时直接恢复即可

**回滚段**
回滚段是存储undolog的地方，一个回滚段对应一次事务执行的undolog集合（事务与回滚段的映射关系存在，便于快速查找到事务的历史数据版本快速回滚）。

**快照（ReadView）**
快照是什么？快照就是基于**整库的某个时刻的镜像**。在不同隔离级别下快照（READVIEW）产生时机不同，可重复读隔离级别是**事务启动时产生快照**，读已提交隔离级别是**执行更新动作时产生快照**。

首先，基于整库镜像，当然不是物理存储，假设整库100G，每个镜像100G，那数据库得崩溃。于是，就有了ReadView的数据结构来定义。
- m_ids：当前系统中存活的事务id的列表
- min_trx_id：当前系统中存活事务的最小事务id，即m_ids的最小值
- max_trx_id：系统下一个即将分配的事务id，**即m_ids的最大值 + 1，**并不是m_ids的最大值
- creator_trx_id：当前事务ID

其中，(-∞，min_trx_id)为已提交事务，[min_trx_id,max_trx_id)为未提交事务, [max_trx_id, +∞]为未开始事务。通过这个规则来定义进行快照读的时候，确认行记录是否可读。

- 如果落在已提交事务区间，表示这个版本是已提交的事务或者是当前事务自己生成的，这个数据是可见的；
- 如果落在未提交事务区间，表示这个版本是由将来启动的事务生成的，是肯定不可见的；
- 如果落在未开始事务区间，那就包括两种情况
  - 若 row trx_id 在数组中，表示这个版本是由还没提交的事务生成的，不可见；
  - 若 row trx_id 不在数组中，表示这个版本是已经提交了的事务生成的，可见。

总之，根据数据行中ROLLBAKCK_PTR能找到所有的回滚日志（历史版本数据），通过快照中事务活跃关系和数据行中TRX_ID来比对确定是否可读来执行快照读，最终实现了MVCC中数据多版本并发读写效果。

### 4、事务的执行流程

#### 4.1、事务是如何执行的？
#### 4.2、事务是如何回滚的？
根据事务ID找到对应的回滚段中的undolog，将回滚段里的日志进行清空即可。



### 5、数据库事务是怎么实现的？
参考：https://draveness.me/mysql-transaction/

原子性和持久性由RedoLog实现，一致性由UndoLog实现，隔离性由MVCC和锁实现。

#### 5.1、原子性&持久性
严格说，原子性和持久性由RedoLog和UndoLog保证。当然，说RedoLog也是可接受的，因为UndoLog也会产生RedoLog。UndoLog的完整性和可靠性需要RedoLog保证，因此数据库崩溃时需要先做RedoLog恢复，然后再做UndoLog回滚。

总之，RedoLog的持久化保证了数据持久化到磁盘，又影响到UndoLog的可靠和完整，最终实现原子性和持久性。UndoLog用于对事务的影响进行撤销，RedoLog在错误处理时，对已提交的事务进行重做。

- 发生错误或需要回滚的事务能够成功回滚（原子性）；
- 在事务提交后，数据没得急写磁盘就宕机时，在下次重新启动后能够恢复数据（持久性）；

#### 5.3、一致性

原子性&持久性&原子性都保证了，一致性就保证了。

不一致的时机：unlog、redolog、磁盘都可能出现不一致

- A时机：写了redolog，写磁盘失败
  - redolog：数据页记录与磁盘页比较，通过回放实现磁盘和redolog的一致
- B时机：事务已提交，redolog写失败
  - undolog/binlog：数据进行比较实现一致

#### 5.4、隔离性
MVCC和锁

### 参考
1. MySQL技术内幕InnoDB存储引擎
2. 深入理解分布式事务
1. [https://draveness.me/mysql-transaction/](https://draveness.me/mysql-transaction/)
2. [InnoDB-事务原理@www.corgiboy.com](https://www.corgiboy.com/InnoD%E5%BC%95%E6%93%8E/InnoDB-%E4%BA%8B%E5%8A%A1%E5%8E%9F%E7%90%86/)
2. [InnoDB并发如此高，原因竟然在这？@架构师之路](https://mp.weixin.qq.com/s/fmzaIobOihKKZ7kyZQInTg)
2. [MySQL-InnoDB究竟如何巧妙实现，4种事务的隔离级别@架构师之路](https://mp.weixin.qq.com/s/C25UbyVaRjqGP3ybIjQr-g)
2. [https://www.cnblogs.com/rjzheng/p/10841031.html](https://www.cnblogs.com/rjzheng/p/10841031.html)
