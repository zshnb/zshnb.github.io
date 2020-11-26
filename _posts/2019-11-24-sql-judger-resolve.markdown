---
layout: post
title: 一次线上sql评测机的bug排查
date: 2020-11-24 20:27
author: zsh
catalog: true
tags:
    - Kotlin
    - MYSQL
---

###场景
昨天线上的mysql的rabbitMQ消息队列是不是就会出现消息阻塞的现象，影响了正常学生的提交，这个问题比较紧急，因此马上就着手排查。

###排查
首先检查是否是rabbitMQ本身的问题，进入rabbitMQ的管理页面，发现一切正常，排除。

然后就怀疑是否消费者不消化消息了，这里消费者是sql评测机，是在k8s集群上运行的一个容器，通过`kubectl exec -it`进入容器后，我们想到了用阿里巴巴的arthas工具trace慢方法，
在经过几个小时的trace后，终于找到执行慢的方法是在`JdbcTemplate.execute()`，卡在了执行sql语句的地方，于是我们怀疑是不是mysql卡住了。

连上评测机的mysql，执行命令`show processlist`，发现一个疑似异常的query，描述是waiting for table metadata lock，谷歌后大概了解到这个错误的出现情景是ddl语句所操作的表在一个进行中的事务里。
仔细排查了评测机的代码，发现并没有任何手动控制事务的代码，于是我们怀疑是不是学生提交的内容中开启了事务。

查看容器日志，找到让评测卡住的最近几个提交id，捞出了学生的提交内容，发现有一个提交是这样的
```sql
start transaction;
delete xxxx
commit
```
这几行会手动开启事务，然后删除数据，最后提交事务，看起来是美好的，但是由于后面2句没有加分号，因此执行的时候报错了，commit根本没有执行，意味着此次事务没有提交。
由于评测机的评测过程中会删除表， 于是后面所有同一个题目的提交都会卡住。因为拿不到表的锁

###验证
1. 在本地数据库随便新建一张表user
2. 开启session1，然后手动`start transaction;`
3. 开启session2. 然后执行语句`drop table user;`
4. 开启session3，执行`show processlist`查看结果
最后结果如图

![image-20200721112228867](/img/2020-11-24-sql-judger-resolve/image_20201126200746.png)

可以看到drop table语句一直在卡着，直到等待时间超过lock timeout或者在开启事务的session里commit;

###解决
当时想了2个方案
1. 设置lock timeout。这个方案不会让我们的评测机卡住，但是会把被手动开启事务提交影响的提交的答案误评，因此pass。
2. 评测机来控制事务，因为mysql的commit与transaction没有一一对应，因此只要在评测结束后commit就可以了。

临时修复：在评测结束后无论有没有开启事务，都用`jdbcTemplate.execute("commit;")`，保证每次评测后当前会话中没有事务。
当然此方案是临时性的修复，因为执行学生提交的连接和最后我们提交事务的连接不是同一个连接。所以有偶然的概率依然会触发之前的bug。

最终修复方案：在评测前先手动获取连接，之后在过程中一直使用这个连接去执行学生的提交以及提交事务，因为是同一个连接，可以保证事务提交生效。

###结语
