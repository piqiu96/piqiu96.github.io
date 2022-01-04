---
title: "MySQL优化"
date: 2022-01-04T23:32:06+08:00
draft: false
categories: ["MySQL"]
---

# SQL优化

# Sql执行顺序

- (8) `SELECT`(9) `DISTINCT` column,…
  选择字段 、去重
- (6) `AGG_FUNC(column or expression)`,…
  聚合函数
- (1) `FROM [left_table]`
  选择表
- (3) <join_type> `JOIN` <right_table>
  链接
- (2) `ON` <join_condition>
  链接条件
- (4) `WHERE` <where_condition>
  条件过滤
- (5) `GROUP BY` <group_by_list>
  分组
- (7) `HAVING` <having_condition>
  分组过滤
- (10) `ORDER BY` <order_by_list>
  排序
- (11) `LIMIT` count OFFSET count;
  分页

# 基础Sql优化

## 查询SQL尽量不要使用select *，而是具体字段

反例：

SELECT * FROM student



正例：



SELECT id,NAME FROM student





理由：

- 字段多时，大表能达到100多个字段甚至达200多个字段
- 只取需要的字段，节省资源、减少网络开销
- select * 进行查询时，很可能不会用到索引，就会造成全表扫描



## 避免在where子句中使用or来连接条件

查询id为1或者薪水为3000的用户：

反例：

SELECT * FROM student WHERE id=1 OR salary=30000



正例：
使用union all





SELECT * FROM student WHERE id=1

UNION ALL

SELECT * FROM student WHERE salary=30000



分开两条sql写



SELECT * FROM student WHERE id=1

SELECT * FROM student WHERE salary=30000



理由：

- 使用or可能会使索引失效，从而全表扫描
- 对于or没有索引的salary这种情况，假设它走了id的索引，但是走到salary查询条件时，它还得全表扫描。也就是说整个过程需要三步：全表扫描+索引扫描+合并。如果它一开始就走全表扫描，直接一遍扫描就搞定。虽然mysql是有优化器的，处于效率与成本考虑，遇到or条件，索引还是可能失效的





## 使用varchar代替char

反例：

`deptname` char(100) DEFAULT NULL COMMENT '部门名称'



**正例：**



`deptname` varchar(100) DEFAULT NULL COMMENT '部门名称'



理由：

- varchar变长字段按数据内容实际长度存储，存储空间小，可以节省存储空间
- char按声明大小存储，不足补空格
- 其次对于查询来说，在一个相对较小的字段内搜索，效率更高

##  

## 尽量使用数值替代字符串类型

- 主键（id）：primary key优先使用数值类型int，tinyint
- 性别（sex）：0-代表女，1-代表男；数据库没有布尔类型，mysql推荐使用tinyint
- 支付方式（payment）：1-现金、2-微信、3-支付宝、4-信用卡、5-银行卡
- 服务状态（state）：1-开启、2-暂停、3-停止
- 商品状态（state）：1-上架、2-下架、3-删除



## 查询尽量避免返回大量数据

如果查询返回数据量很大，就会造成查询时间过长，网络传输时间过长。同时，大量数据返回也可能没有实际意义。如返回上千条甚至更多，用户也看不过来。
通常采用分页，一页习惯10/20/50/100条。





## 使用explain分析你SQL执行计划

SQL很灵活，一个需求可以很多实现，那哪个最优呢？SQL提供了explain关键字，它可以分析你的SQL执行计划，看它是否最佳。Explain主要看SQL是否使用了索引。





EXPLAIN

SELECT * FROM student WHERE id=1



返回结果：


![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtybbs/132/978/613/0220086000132978613.20210422111204.18237127036943912923263679436760:50521228062829:2800:76234CF68DA582ADEBEF874BB15C994D33D7447FABD28442AF7068D84873C5AE.png)

## 是否使用了索引及其扫描类型

SQL索引概念（详解B+树）



```
type：
```

- ALL 

  全表扫描，没有优化，最慢的方式

- index 

  索引全扫描

- range

  索引范围扫描，常用语<，<=，>=，between等操作

- ref 

  使用非唯一索引扫描或唯一索引前缀扫描，返回单条记录，常出现在关联查询中

- eq_ref

  类似ref，区别在于使用的是唯一索引，使用主键的关联查询

- const

  当查询是对主键或者唯一键进行精确查询，系统会把匹配行中的其他列作为常数处理

- null MySQL 

  不访问任何表或索引，直接返回结果

- System 表只有一条记录(实际中基本不存在这个情况)



> 性能排行：
> System > const > eq_ref > ref > range > index > ALL

```
possible_keys：
```

- 显示可能应用在这张表中的索引

```
key：
```

- 真正使用的索引方式

## 创建name字段的索引

提高查询速度的最简单最佳的方式



ALTER TABLE student ADD INDEX index_name (NAME)



## 优化like语句：

模糊查询，程序员最喜欢的就是使用like，但是like很可能让你的索引失效

反例：

**反例：**



EXPLAIN

SELECT id,NAME FROM student WHERE NAME LIKE '%1'

EXPLAIN

SELECT id,NAME FROM student WHERE NAME LIKE '%1%'



**正例：**



EXPLAIN

SELECT id,NAME FROM student WHERE NAME LIKE '1%'



**理由：**

未使用索引：故意使用sex非索引字段



EXPLAIN

SELECT id,NAME FROM student WHERE NAME=1 OR sex=1



![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtybbs/132/978/613/0220086000132978613.20210422111356.39863519415012733987118882966912:50521228062829:2800:2D342C8640AA3821DB14FEB248F1408834153D38B48EED3A2902930F312DFBC3.png)

主键索引生效



EXPLAIN

SELECT id,NAME FROM student WHERE id=1



![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtybbs/132/978/613/0220086000132978613.20210422111411.46067537915041441861044073375128:50521228062829:2800:19F06774582A18932088538B3CDC257E8B73C4A6102FB343259ED772226F71D6.png)

索引失效，type=ALL，全表扫描



EXPLAIN

SELECT id,NAME FROM student WHERE id LIKE '%1'



![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtybbs/132/978/613/0220086000132978613.20210422111427.71603680519366143113156873361784:50521228062829:2800:1C1CA6AAF123AB4612E5C3004489FD8F9D4E6A12608F76B610ED7BC96E935E36.png)



## 字符串怪现象

反例：

\#未使用索引

EXPLAIN

SELECT * FROM student WHERE NAME=123



**正例：**



\#使用索引

EXPLAIN

SELECT * FROM student WHERE NAME='123'



理由：

- 为什么第一条语句未加单引号就不走索引了呢？这是因为不加单引号时，是字符串跟数字的比较，它们类型不匹配，MySQL会做隐式的类型转换，把它们转换为数值类型再做比较

##  

## 索引不宜太多，一般5个以内

- 索引并不是越多越好，虽其提高了查询的效率，但却会降低插入和更新的效率
- 索引可以理解为一个就是一张表，其可以存储数据，其数据就要占空间
- 再者，索引表的一个特点，其数据是排序的，那排序要不要花时间呢？肯定要
- insert或update时有可能会重建索引，如果数据量巨大，重建将进行记录的重新排序，所以建索引需要慎重考虑，视具体情况来定
- 一个表的索引数最好不要超过5个，若太多需要考虑一些索引是否有存在的必要

##  

## 索引不适合建在有大量重复数据的字段上

如性别字段。因为SQL优化器是根据表中数据量来进行查询优化的，如果索引
列有大量重复数据，Mysql查询优化器推算发现不走索引的成本更低，很可能就放弃索引了。

## where限定查询的数据

数据中假定就一个男的记录

反例：



SELECT id,NAME FROM student WHERE sex='男'



**正例：**



SELECT id,NAME FROM student WHERE id=1 AND sex='男'



理由：

- 需要什么数据，就去查什么数据，避免返回不必要的数据，节省开销

##  

## 避免在索引列上使用内置函数

业务需求：查询最近七天内新生儿（用学生表替代下）

给birthday字段创建索引：

ALTER TABLE student ADD INDEX idx_birthday (birthday)



**当前时间加7天：**



SELECT NOW() 

SELECT DATE_ADD(NOW(), INTERVAL 7 DAY)



**反例：**



EXPLAIN

SELECT * FROM student

WHERE DATE_ADD(birthday,INTERVAL 7 DAY) >=NOW();



**正例：**



EXPLAIN

SELECT * FROM student

WHERE birthday >= DATE_ADD(NOW(),INTERVAL 7 DAY);



理由：

- 使用索引列上内置函数
- 索引失效：

![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtybbs/132/978/613/0220086000132978613.20210422111628.66666154078568279806262611250732:50521228062829:2800:4B21859EB729572049C8FAFC9D0F359762F3C2CE35015DB2E6737656BA53C31A.png)

- 索引有效：

![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtybbs/132/978/613/0220086000132978613.20210422111643.86708000394627663730620537378184:50521228062829:2800:2D0F694F927A89812B5F1ACA8AB8ACF35F8666B117269D8F26A8D58680B2633A.png)

## 避免在where中对字段进行表达式操作

反例：



EXPLAIN

SELECT * FROM student WHERE id+1-1=+1



**正例：**



EXPLAIN

SELECT * FROM student WHERE id=+1-1+1



EXPLAIN

SELECT * FROM student WHERE id=1



理由：

- SQL解析时，如果字段相关的是表达式就进行全表扫描
- 

![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtybbs/132/978/613/0220086000132978613.20210422111717.99721597714739349493735689204552:50521228062829:2800:B730A47102433BCB59552DE7B272342D5645DD3BA376A5134B4CD77B8C0955DF.png)

- 字段干净无表达式，索引生效

![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtybbs/132/978/613/0220086000132978613.20210422111723.51677555163914612053523825340208:50521228062829:2800:88ED6F0B3BABE9B7DF17C40352F0FD6FE61250D6E7ED40F10D83C8E90DD7FE12.png)

## 避免在where子句中使用!=或<>操作符

应尽量避免在where子句中使用!=或<>操作符，否则引擎将放弃使用索引而进行全表扫描。记住实现业务优先，实在没办法，就只能使用，并不是不能使用。如果不能使用，SQL也就无需支持了。

反例：



EXPLAIN

SELECT * FROM student WHERE salary!=3000



EXPLAIN

SELECT * FROM student WHERE salary<>3000



理由：

- 使用!=和<>很可能会让索引失效

![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtybbs/132/978/613/0220086000132978613.20210422111755.87767520236793350189779492802257:50521228062829:2800:A7380E9DCAAACE5F583325F93C0DA33E130ECD4B6DC1786B831274E93519894D.png)

## 去重distinct过滤字段要少

\#索引失效

EXPLAIN

SELECT DISTINCT * FROM student



\#索引生效

EXPLAIN

SELECT DISTINCT id,NAME FROM student



EXPLAIN

SELECT DISTINCT NAME FROM student



理由：

- 带distinct的语句占用cpu时间高于不带distinct的语句。因为当查询很多字段时，如果使用distinct，数据库引擎就会对数据进行比较，过滤掉重复数据，然而这个比较、过滤的过程会占用系统资源，如cpu时间



## where中使用默认值代替null

环境准备：

\#修改表，增加age字段，类型int，非空，默认值0

ALTER TABLE student ADD age INT NOT NULL DEFAULT 0;



\#修改表，增加age字段的索引，名称为idx_age

ALTER TABLE student ADD INDEX idx_age (age);



**反例：**



EXPLAIN

SELECT * FROM student WHERE age IS NOT NULL



**正例：**



EXPLAIN

SELECT * FROM student WHERE age>0



理由：

- 并不是说使用了is null 或者 is not null 就会不走索引了，这个跟mysql版本以及查询成本都有关
- 如果mysql优化器发现，走索引比不走索引成本还要高，就会放弃索引，这些条件 !=，<>，is null，is not null经常被认为让索引失效，其实是因为一般情况下，查询的成本高，优化器自动放弃索引的
- 如果把null值，换成默认值，很多时候让走索引成为可能，同时，表达意思也相对清晰一点

#  

# 高级SQL优化

## 批量插入性能提升

大量数据提交，上千，上万，批量性能非常快，mysql独有

多条提交：

INSERT INTO student (id,NAME) VALUES(4,'name1');

INSERT INTO student (id,NAME) VALUES(5,'name2');



**批量提交：**



INSERT INTO student (id,NAME) VALUES(4,'name1'),(5,'name2');



理由：

- 默认新增SQL有事务控制，导致每条都需要事务开启和事务提交；而批量处理是一次事务开启和提交。自然速度飞升
- 数据量小体现不出来



## 批量删除优化

避免同时修改或删除过多数据，因为会造成cpu利用率过高，会造成锁表操作，从而影响别人对数据库的访问。

反例：

\#一次删除10万或者100万+？

delete from student where id <100000;



\#采用单一循环操作，效率低，时间漫长

for（User user:list）{

 delete from student;

}



正例：



\#分批进行删除，如每次500

for(){

delete student where id<500;

}



delete student where id>=500 and id<1000;



理由：

- 一次性删除太多数据，可能造成锁表，会有lock wait timeout exceed的错误，所以建议分批操作

## 伪删除设计

商品状态（state）：1-上架、2-下架、3-删除

理由：

- 这里的删除只是一个标识，并没有从数据库表中真正删除，可以作为历史记录备查
- 同时，一个大型系统中，表关系是非常复杂的，如电商系统中，商品作废了，但如果直接删除商品，其它商品详情，物流信息中可能都有其引用。
- 通过where state=1或者where state=2过滤掉数据，这样伪删除的数据用户就看不到了，从而不影响用户的使用
- 操作速度快，特别数据量很大情况下

##  

## 提高group by语句的效率

可以在执行到该语句前，把不需要的记录过滤掉

反例：先分组，再过滤

select job，avg（salary） from employee 

group by job 

having job ='president' or job = 'managent';



**正例：先过滤，后分组**



select job，avg（salary） from employee 

where job ='president' or job = 'managent' 

group by job;



## 复合索引最左特性

创建复合索引，也就是多个字段

ALTER TABLE student ADD INDEX idx_name_salary (NAME,salary)



满足复合索引的左侧顺序，哪怕只是部分，复合索引生效



EXPLAIN

SELECT * FROM student WHERE NAME='name1'



没有出现左边的字段，则不满足最左特性，索引失效



EXPLAIN

SELECT * FROM student WHERE salary=3000



复合索引全使用，按左侧顺序出现 name,salary，索引生效



EXPLAIN

SELECT * FROM student WHERE NAME='陈子枢' AND salary=3000



虽然违背了最左特性，但MYSQL执行SQL时会进行优化，底层进行颠倒优化



EXPLAIN

SELECT * FROM student WHERE salary=3000 AND NAME='name1'



理由：

- 复合索引也称为联合索引
- 当我们创建一个联合索引的时候，如(k1,k2,k3)，相当于创建了（k1）、(k1,k2)和(k1,k2,k3)三个索引，这就是最左匹配原则
- 联合索引不满足最左原则，索引一般会失效，但是这个还跟Mysql优化器有关的

##  

## 排序字段创建索引

什么样的字段才需要创建索引呢？原则就是where和order by中常出现的字段就创建索引。



\#使用*，包含了未索引的字段，导致索引失效

EXPLAIN

SELECT * FROM student ORDER BY NAME;



EXPLAIN

SELECT * FROM student ORDER BY NAME,salary



\#name字段有索引

EXPLAIN

SELECT id,NAME FROM student ORDER BY NAME



\#name和salary复合索引

EXPLAIN

SELECT id,NAME FROM student ORDER BY NAME,salary



EXPLAIN

SELECT id,NAME FROM student ORDER BY salary,NAME



\#排序字段未创建索引，性能就慢

EXPLAIN

SELECT id,NAME FROM student ORDER BY sex



## 删除冗余和重复的索引

SHOW INDEX FROM student 



\#创建索引index_name

ALTER TABLE student ADD INDEX index_name (NAME)



\#删除student表的index_name索引

DROP INDEX index_name ON student ;



\#修改表结果，删除student表的index_name索引

ALTER TABLE student DROP INDEX index_name ;



\#主键会自动创建索引，删除主键索引

ALTER TABLE student DROP PRIMARY KEY ;



![img](https://alliance-communityfile-drcn.dbankcdn.com/FileServer/getFile/cmtybbs/132/978/613/0220086000132978613.20210422112151.20020993916009081085766960518663:50521228062829:2800:A64B3826F29B8C9B8ABA1D236ADD9B305175A921111541E31821133978F067AD.png)

## 不要有超过5个以上的表连接

- 关联的表个数越多，编译的时间和开销也就越大
- 每次关联内存中都生成一个临时表
- 应该把连接表拆开成较小的几个执行，可读性更高
- 如果一定需要连接很多表才能得到数据，那么意味着这是个糟糕的设计了
- 阿里规范中，建议多表联查三张表以下



## inner join 、left join、right join，优先使用inner join

三种连接如果结果相同，优先使用inner join，如果使用left join左边表尽量小

- inner join 内连接，只保留两张表中完全匹配的结果集
- left join会返回左表所有的行，即使在右表中没有匹配的记录
- right join会返回右表所有的行，即使在左表中没有匹配的记录



理由：

- 如果inner join是等值连接，返回的行数比较少，所以性能相对会好一点
- 同理，使用了左连接，左边表数据结果尽量小，条件尽量放到左边处理，意味着返回的行数可能比较少。这是mysql优化原则，就是小表驱动大表，小的数据集驱动大的数据集，从而让性能更优

##  

## in子查询的优化

日常开发实现业务需求可以有两种方式实现：

- 一种使用数据库SQL脚本实现
- 一种使用程序实现
  如需求：查询所有部门的所有员工：

\#in子查询

SELECT * FROM tb_user WHERE dept_id IN (SELECT id FROM tb_dept);

\#这样写等价于：



\#先查询部门表

SELECT id FROM tb_dept



\#再由部门dept_id，查询tb_user的员工

SELECT * FROM tb_user u,tb_dept d WHERE u.dept_id = d.id



假设表A表示某企业的员工表，表B表示部门表，查询所有部门的所有员工，很容易有以下程序实现，可以抽象成这样的一个嵌套循环：



List<> resultSet;

for(int i=0;i<B.length;i++) {

 for(int j=0;j<A.length;j++) {

  if(A[i].id==B[j].id) {

   resultSet.add(A[i]);

   break;

  }

 }

}



上面的需求使用SQL就远不如程序实现，特别当数据量巨大时。



理由：

- 数据库最费劲的就是程序链接的释放。假设链接了两次，每次做上百万次的数据集查询，查完就结束，这样就只做了两次；相反建立了上百万次链接，申请链接释放反复重复，就会额外花费很多实际，这样系统就受不了了，慢，卡顿



## 尽量使用union all替代union

反例：

SELECT * FROM student

UNION

SELECT * FROM student



**正例：**



SELECT * FROM student

UNION ALL

SELECT * FROM student



理由：

- union和union all的区别是，union会自动去掉多个结果集合中的重复结果，而union all则将所有的结果全部显示出来，不管是不是重复
- union：对两个结果集进行并集操作，不包括重复行，同时进行默认规则的排序
- union在进行表链接后会筛选掉重复的记录，所以在表链接后会对所产生的结果集进行排序运算，删除重复的记录再返回结果。实际大部分应用中是不会产生重复的记录，最常见的是过程表与历史表UNION
