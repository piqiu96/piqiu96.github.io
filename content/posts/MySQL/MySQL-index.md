---
title: "MySQL索引"
date: 2022-01-04T23:33:50+08:00
draft: false
---

# MySQL索引

## 一、索引结构

- hash：读性能（等值查询高、范围查询低）
- 有序数组：读性能（等值查询高，范围查询高「二分」）、写性能（元素后移性能降低） -> 适合只读不写场景
- 二叉搜索树：读写性能高，平衡树有树高问题（读写磁盘有IO瓶颈） -> N叉树
- 跳表：
- B+树：



不管是哈希还是有序数组，或者 N 叉树，它们都是不断迭代、不断优化的产物或者解决方案。数据库技术发展到今天，跳表、LSM 树等数据结构也被用于引擎设计中，这里我就不再一一展开了。



## 二、索引类型

聚簇索引：主键索引，存储<主键， 行记录>

非聚簇索引：普通索引，存储<索引列，主键>

回表：先查非聚簇索引，然后查聚簇索引，这种行为称为回表

前缀索引：

覆盖索引：查询聚簇索引，假设已满足查询需求，则无需回表。该行为称为覆盖索引

索引下推：

​	无索引下推：查ID，回表，比较内容

​	有索引下推：查ID，匹配所有索引项，再回表比较内容

最左前缀原则：





## 索引优化

问题1：前缀匹配

SQL1：select * from t where title = "%xxx%"   =>  非前缀匹配

SQL2:  select * from where title = "xxx%"  => 前缀匹配



问题2：关联查询

全连接

左连接



## 资料参考

- https://mp.weixin.qq.com/s/pQsB_jLb-Qr59aYNk0-SJQ
- https://mp.weixin.qq.com/s/ap9tkaEuWDi39u0NFxnACA
- https://blog.csdn.net/qq_26545305/article/details/107962298
- https://segmentfault.com/a/1190000038749020
- https://www.cnblogs.com/vincently/p/4526560.html

