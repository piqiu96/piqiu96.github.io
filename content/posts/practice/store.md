---
title: "如何做存储选型?"
date: 2022-09-17T22:24:46+08:00
draft: true
categories: ["工程实践"]
---

在做存储选型的时候需要考虑如下要素，数据量、并发量、实时性、一致性要求、读写分布和类型、安全性、运维性等。

根据这些指标，软件系统可分成几类。

1. 管理型系统，如运营类系统，首选关系型
2. 大流量系统，如电商单品页的某个服务，后台选关系型，前台选内存型
3. 日志型系统，原始数据选列式，日志搜索选倒排索引
4. 搜索型系统，指站内搜索，非通用搜索，如商品搜索，后台选关系型，前台选倒排索引
5. 事务型系统，如库存/交易/记账，选关系型+缓存+一致性协议，或新型关系型数据库
6. 离线计算，如大量数据分析，首选列式，关系型也可以
7. 实时计算，如实时监控，可以选时序数据库或列式数据库