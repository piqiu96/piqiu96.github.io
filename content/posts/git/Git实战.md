---
title: "Git速查"
date: 2020-12-04T13:12:33+08:00
draft: true
---

## 一、概念
### 1.1. Git是什么？

GIT是分布式版本控制系统，简单分解就是：分布式 + 版本控制 + 系统，核心要素就是版本控制。

**问题：怎么理解版本控制，想想我们生活工作中的场景？**  
1、假设没有版本控制系统，你上线代码会做什么事情？（假设你不做，出现问题你怎么办？） 
-  备份原文件，重命名 niubi.php.20200520
-  复制修改后文件移动到制定项目路径

2、当我们还是单机mysql数据库，对稳定要求不高，只需具备快速切换能力即可，这种场景你会怎么做？
- 方案一：对数据库做版本控制，定时dump数据库文件，数据库文件名（XX.sql.时间时刻）
- 方案二：做数据库主从，主库挂掉，脚本快速切换到从库

3、当你写毕业论文的时候，写了N年的论文丢了你怕不怕，为了防止这操作你常见的操作是什么？
- 老师想看论文，阿邱同学提交论文，命名：阿邱同学毕业论文_V1
- 老师想看论文，阿邱同学提交论文，命名：阿邱同学毕业论文_V2
- ......
- 老师想看论文，阿邱同学提交论文，命名：阿邱同学毕业论文_V死不改版

本质上，版本控制本质上就是把每次文件（数据）集合打个标记（版本），让自己快速跳转到目标版本。**最终，实现代码可以随意切换到随意版本的代码，当出现问题能快速切换到稳定版本。** 

### 1.2 为什么要用Git？
不管人类还是软件，都会遇到类似的问题，版本控制系统诞生无非是苦秦久矣罢了。还有细心思考下，不光是代码，凡是需要存储的内容（文件、数据库）都需要版本控制，不然任何一环故障都会影响系统。

其实，在Git出现前就出现很多版本控制系统，主流思想无非就是增量或全量，Git采用的全量快照的方式，把修改的文件集合打个版本。

**增量：每次记录变化的内容，恢复的时候依据 基准版本 + 增量版本回滚。**
![](https://cdn.jsdelivr.net/gh/piqiu96/aqiucdn/imgs/git/deltas.jpeg)

**全量：每次记录本次变更的所有文件放入缓存区，恢复的时候直接缓存区找到全量版本回滚。**
![](https://cdn.jsdelivr.net/gh/piqiu96/aqiucdn/imgs/git/snapshots.jpeg)


于是，GIt有几个核心流程和核心概念。
核心概念：文件流转状态（已提交、已修改、已暂存），为了可通过文件状态跟踪出文件内容，于是git项目有三个阶段：工作区、暂存区以及 Git 目录。
![](https://cdn.jsdelivr.net/gh/piqiu96/aqiucdn/imgs/git/areas.jpeg)


核心流程：
1. 在工作区中修改文件。
2. 将你想要下次提交的更改选择性地暂存，这样只会将更改的部分添加到暂存区。
3. 提交更新，找到暂存区的文件，将快照永久性存储到 Git 目录。

## 二、实战

![](https://cdn.jsdelivr.net/gh/piqiu96/aqiucdn/imgs/git/cmdlist.jpeg)

**场景一：添加如何代码，提cr**
```Bash
git pull 
git add <file> 
git commit -m "commit message" 
```

**场景二：回滚commit**
{{< admonition tip "说明" >}}
提交非预期代码，需要回滚重新提交
{{< /admonition >}}
```Bash
git log
git reset --soft '版本号'
git reset --soft HEAD^ 回滚到上次提交
```

**场景三:  切换分支且拉取代码**  
{{< admonition tip "说明" >}}
主要用于项目初始化，切换分支且拉取代码
{{< /admonition >}}
```Bash
git branch -a
git checkout -b <branch_local> orign/<branch_remote>

baranch_local: 本地分支名，可自定义随意
branch_remote：远程分支名，必须是仓库内名
```

**场景四：多个commit合并**  
{{< admonition tip "说明" >}}
通常情况，我们设置一次CR只允许一次commit，如何把多次commit合并到同一commit？
{{< /admonition >}}
```Bash
git reset --soft HEAD^
git commit --amend
git push origin <remote>
```

**场景五：代码冲突**  
{{< admonition tip "说明" >}}
代码冲突是最常见的问题，也是很头大的问题
{{< /admonition >}}
```Bash
方法一：保留服务器改动，仅合入远程仓库新增配置项，标记出冲突代码！
git stash 
git pull
git stash pop

方法二：用远程仓库代码库文件完全覆盖本地文件，强制回滚！（慎用）
git reset --hard
git pull
```

**场景6: 解决同次冲突提交**
```Bash
git fetch origin
git rebase origin/master

git add -u
git rebase --continue
git push origin HEAD:refs/for/master
```

## 参考资料
- https://www.liaoxuefeng.com/wiki/896043488029600
- https://git-scm.com/book/zh/v2
