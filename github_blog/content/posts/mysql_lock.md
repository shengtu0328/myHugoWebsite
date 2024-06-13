---
title: "Mysql_lock"
date: 2024-06-11T15:30:24+08:00
draft: true
---

## 5.锁

### 5.1概述

MySQL中的锁，按照锁的粒度分，分为以下三类：

全局锁：锁定数据库中的所有表。

表级锁：每次操作锁住整张表。

行级锁：每次操作锁住对应的行数据。


### 5.2全局锁

全局锁就是对整个数据库实例加锁，加锁后整个实例就处于**只读**状态，后续的DML的写语句，DDL语

句，已经更新操作的事务提交语句都将被阻塞。

其典型的使用场景是做全库的逻辑备份，对所有的表进行锁定，从而获取一致性视图，保证数据的完整

性。

11111111111111111111

#### **5.2.2** **语法**

1). 加全局锁

```
flush tables with read lock ;
```

2). 数据备份】

 将wifi库的数据库数据导出到D:/wifidatabase.sql，不是在mysql 终端里执行，而是在终端外面

```
mysqldump -uroot –proot wifi > D:/wifidatabase.sql
```

3). 释放锁

```
unlock tables ;
```

#### **5.2.3** **例子**



| mysql 客户端 session a        | mysql 客户端  session b                           | 在客户端外面                                       |
| ----------------------------- | ------------------------------------------------- | -------------------------------------------------- |
| flush tables with read lock ; |                                                   |                                                    |
|                               | select * from a （正常查询）                      |                                                    |
|                               | update a set approval_status=2 where id=2（阻塞） |                                                    |
|                               |                                                   | mysqldump -uroot –proot wifi > D:/wifidatabase.sql |
| unlock tables ;               |                                                   |                                                    |
|                               | update语句在解锁表后执行成功                      |                                                    |

#### **5.2.4** **特点**

数据库中加全局锁，是一个比较重的操作，存在以下问题：

如果在主库上备份，那么在备份期间都不能执行更新，业务基本上就得停摆。

如果在从库上备份，那么在备份期间从库不能执行主库同步过来的二进制日志（binlog），会导

致主从延迟。

在InnoDB引擎中，我们可以在备份时加上参数 --single-transaction 参数来完成不加锁的一致

性数据备份。

```
mysqldump --single-transaction -uroot –p123456 itcast > itcast.sql
```

### **5.3** **表级锁**

#### **5.3.1** **介绍**

表级锁，每次操作锁住整张表。锁定粒度大，发生锁冲突的概率最高，并发度最低。应用在MyISAM、

InnoDB、BDB等存储引擎中。

对于表级锁，主要分为以下三类：

1. 表锁
2. 元数据锁（meta data lock，MDL）
3. 意向锁



#### **5.3.2** **表锁**

对于表锁，分为两类：

1. 表共享读锁（read lock，简称表读锁）
2. 表独占写锁（write lock，简称表写锁）





A.读锁

![](表读锁.jpeg)

语法：

1. 加锁：lock tables 表名... read/write。
2. 释放锁：unlock tables / 客户端断开连接 。




左侧为客户端一，对指定表加了读锁，不会影响右侧客户端二的读，但是会阻塞右侧客户端的写，而且加锁的会话也不能写

表读锁例子

| sessionA                                                     | sessionB                                                   |
| ------------------------------------------------------------ | ---------------------------------------------------------- |
| lock tables a read;                                          |                                                            |
| -- 正常查询<br/>select * from a;                             |                                                            |
|                                                              | -- 正常查询<br/>select * from a;                           |
| -- 报错Table 'a' was locked with a READ lock and can't be updated，切解锁表后也不会成功<br/>update a set approval_status=2 WHERE id=2; | -- （阻塞）<br/>update a set approval_status=2 WHERE id=1; |
| UNLOCK tables                                                |                                                            |
|                                                              | 表解锁后update语句执行成功                                 |





B. 写锁

![](表写锁.png)

左侧为客户端一，对指定表加了写锁，会阻塞右侧客户端的读和写。但客户端一可以写

表写锁例子



