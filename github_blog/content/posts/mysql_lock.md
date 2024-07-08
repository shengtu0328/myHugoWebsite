+++
title = 'Mysql_lock'
date = 2024-06-17T09:32:16+08:00
draft = true

+++

## 5.锁



```
-- 测试数据
CREATE TABLE `a` (
`uid` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT '' COMMENT '审核记录主表 guid',
`approval_status` int(11) NULL DEFAULT 1 COMMENT '审核状态(1:审核中,2:通过)',
`id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键 id',
PRIMARY KEY (`id`) USING BTREE,
INDEX `idx_guid`(`uid`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 22 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci COMMENT = '审核记录主表' ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of a
-- ----------------------------
INSERT INTO `a` VALUES ('a1', 1, 1);
INSERT INTO `a` VALUES ('a2', 1, 2);



CREATE TABLE `a_detail` (
`approval_status` int(11) NULL DEFAULT 1 COMMENT '审核状态(1:审核中;2:通过)',
`id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键 id',
`auid` varchar(200) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
PRIMARY KEY (`id`) USING BTREE,
INDEX `idx_aid`(`auid`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 74 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci COMMENT = '审核记录明细表' ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of a_detail
-- ----------------------------
INSERT INTO `a_detail` VALUES (1, 1, 'a1');
INSERT INTO `a_detail` VALUES (1, 2, 'a1');
INSERT INTO `a_detail` VALUES (1, 3, 'a2');
INSERT INTO `a_detail` VALUES (1, 4, 'a2');

```



```
-- mysql连接语句 用了mysql8数据库
mysql -uroot –proot wifi 
mysql -h 127.0.0.1 -P 3308 -uroot –proot wifi 
mysql -h 10.18.5.103 -P 3306 -uroot –pG3H8HnwHmA wifi 


mysql -h 127.0.0.1 -P 3308 -uroot –proot testbase

```



```
-- mysql8
select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from
performance_schema.data_locks;

-- mysql5
select * from information_schema.innodb_locks;

```



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




左侧为客户端一，对指定表加了读锁，不会影响右侧客户端二的读，但是会阻塞右侧客户端的写，而且加锁的会话自己也不能写

表读锁例子

| sessionA                                                     | sessionB                                                   |
| ------------------------------------------------------------ | ---------------------------------------------------------- |
| lock tables a read;                                          |                                                            |
| -- 正常查询<br/>select * from a;                             |                                                            |
|                                                              | -- 正常查询<br/>select * from a;                           |
| -- 报错Table 'a' was locked with a READ lock and can't be updated，切解锁表后也不会成功<br/>update a set approval_status=2 WHERE id=2; | -- （阻塞）<br/>update a set approval_status=2 WHERE id=1; |
| UNLOCK tables;                                               |                                                            |
|                                                              | 表解锁后update语句执行成功                                 |





B. 写锁

![](表写锁.png)

左侧为客户端一，对指定表加了写锁，会阻塞右侧客户端的读和写。但客户端一可以写

表写锁例子

| sessionA                                               | sessionB                                           |
| ------------------------------------------------------ | -------------------------------------------------- |
| lock tables a write;                                   |                                                    |
| select * from a;（正常查询）                           |                                                    |
|                                                        |                                                    |
| update a set approval_status=2 WHERE id=2;（正常修改） |                                                    |
|                                                        | select * from a;（阻塞）                           |
|                                                        | update a set approval_status=2 WHERE id=1;（阻塞） |
| UNLOCK tables;                                         |                                                    |
|                                                        | unlock tables 后上一句update执行成功               |



#### **5.3.3** **元数据锁**

meta data lock , 元数据锁，简写MDL。

MDL加锁过程是系统**自动控制**，无需显式使用，在访问一张表的时候会自动加上。MDL锁主要作用是维

护表元数据的数据一致性，在表上有活动事务的时候，不可以对元数据进行写入操作。为了避免DML与

DDL冲突，保证读写的正确性。**（你不能在别人查询的时候修改表结构或者把表删了）**。

这里的元数据，大家可以简单理解为就是一张表的表结构。 也就是说，**某一张表涉及到未提交的事务**

**时，是不能够修改这张表的表结构的。**

在MySQL5.5中引入了MDL**，** 有两种MDL

**当对一张表进行增删改查的时候，加MDL读锁(共享)；** MDL读锁间兼容，MDL读锁与MDL写锁互斥

**当对表结构进行变更操作的时候，加MDL写锁(排他)。**



常见的SQL操作时，所添加的元数据锁：

![](MDL.jpeg)







例子：MDL读锁间兼容 

| sessionA                     | sessionB                                               |
| ---------------------------- | ------------------------------------------------------ |
| begin;                       | begin;                                                 |
| select * from a;（正常查询） | select * from a;（正常查询）                           |
|                              | update a set approval_status=2 WHERE id=1;（正常修改） |
|                              |                                                        |
| commit;                      | commit;                                                |





例子：MDL读锁和MDL写锁 不兼容

| sessionA                     | sessionB                                            |
| ---------------------------- | --------------------------------------------------- |
| begin;                       |                                                     |
| select * from a;（正常查询） |                                                     |
|                              | alter table a add column s_name varchar(500);(阻塞) |
| commit;                      |                                                     |
|                              | alter语句执行成功                                   |





元数据锁开启

```
update performance_schema.setup_instruments
set enabled = 'YES', timed = 'YES'
where name = 'wait/lock/metadata/sql/mdl';
```

我们可以通过下面的SQL，来查看数据库中的**元数据锁**的情况：

```
select object_type,object_schema,object_name,lock_type,lock_duration from performance_schema.metadata_locks ;

```



只有在事务开启后，执行增删改查的sql后，才会有元数据锁记录的产生。在执行了select 和update语句查询元数据锁结果如下

```
mysql> select object_type,object_schema,object_name,lock_type,lock_duration from performance_schema.metadata_locks ;
+-------------+--------------------+----------------+--------------+---------------+
| object_type | object_schema      | object_name    | lock_type    | lock_duration |
+-------------+--------------------+----------------+--------------+---------------+
| TABLE       | wifi               | a              | SHARED_READ  | TRANSACTION   |
| TABLE       | wifi               | a              | SHARED_WRITE | TRANSACTION   |
| TABLE       | performance_schema | metadata_locks | SHARED_READ  | TRANSACTION   |
```

如果再去执行alter语句

```
mysql> select object_type,object_schema,object_name,lock_type,lock_duration from performance_schema.metadata_locks ;
+-------------+--------------------+----------------+---------------------+---------------+
| object_type | object_schema      | object_name    | lock_type           | lock_duration |
+-------------+--------------------+----------------+---------------------+---------------+
| TABLE       | wifi               | a              | SHARED_READ         | TRANSACTION   |
| TABLE       | wifi               | a              | SHARED_WRITE        | TRANSACTION   |
| GLOBAL      | NULL               | NULL           | INTENTION_EXCLUSIVE | STATEMENT     |
| SCHEMA      | wifi               | NULL           | INTENTION_EXCLUSIVE | TRANSACTION   |
| TABLE       | wifi               | a              | SHARED_UPGRADABLE   | TRANSACTION   |
| TABLE       | wifi               | a              | EXCLUSIVE           | TRANSACTION   |
| TABLE       | performance_schema | metadata_locks | SHARED_READ         | TRANSACTION   |
+-------------+--------------------+----------------+---------------------+---------------+
```



#### 5.3.4 意向锁

**为了避免DML在执行时，加的行锁与表锁的冲突，在InnoDB中引入了意向锁，使得表锁不用检查每行**

数据是否加锁，使用意向锁来减少表锁的检查。意向锁是表级别。



**分类**

意向共享锁(**IS**): 由语句select ... lock in share mode添加 。 select语句不会添加

与 表锁共享锁(read)兼容，与表锁排他锁(write)互斥。



意向排他锁(**IX**): 由insert、update、delete、select...for update添加 。

与表锁共享锁(read)及排他锁(write)都互斥，意向锁之间不会互斥。



一旦事务提交了，意向共享锁、意向排他锁，都会自动释放。





A. 意向共享锁与表读锁是兼容的

| sessionA                                                 | sessionB                            |
| -------------------------------------------------------- | ----------------------------------- |
| begin;                                                   |                                     |
| select * from a where id=1 lock in share mode;(正常查询) |                                     |
|                                                          | lock tables a read;(加表读锁成功)   |
|                                                          | lock tables a write;(加表写锁 阻塞) |
| commit;                                                  |                                     |
|                                                          | 事务a提交后，表写锁成功             |





**可以通过以下SQL，查看意向锁及行锁的加锁情况：**

```
//
select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from
performance_schema.data_locks;

mysql> select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from
    -> performance_schema.data_locks;
+---------------+-------------+------------+-----------+---------------+-----------+
| object_schema | object_name | index_name | lock_type | lock_mode     | lock_data |
+---------------+-------------+------------+-----------+---------------+-----------+
| wifi          | a           | NULL       | TABLE     | IS            | NULL      |
| wifi          | a           | PRIMARY    | RECORD    | S,REC_NOT_GAP | 1         |
```









B. 意向排他锁与表读锁、写锁都是互斥的



| sessionA                                              | sessionB                          |
| ----------------------------------------------------- | --------------------------------- |
| begin;                                                |                                   |
| update a set approval_status=7 where  id=1;(正常查询) |                                   |
|                                                       | lock tables a read;(加表读锁阻塞) |
|                                                       | lock tables a wtie;(加表写锁阻塞) |
| commit;                                               |                                   |



```
mysql> select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from
    -> performance_schema.data_locks;
+---------------+-------------+------------+-----------+---------------+-----------+
| object_schema | object_name | index_name | lock_type | lock_mode     | lock_data |
+---------------+-------------+------------+-----------+---------------+-----------+
| wifi          | a           | NULL       | TABLE     | IX            | NULL      |
| wifi          | a           | PRIMARY    | RECORD    | X,REC_NOT_GAP | 1         |
+---------------+-------------+------------+-----------+---------------+-----------+
2 rows in set (0.00 sec)

```

### **5.4** **行级锁**

#### **5.4.1** **介绍**

行级锁，每次操作锁住对应的行数据。锁定粒度最小，发生锁冲突的概率最低，并发度最高。应用在

InnoDB存储引擎中。

InnoDB的数据是基于索引组织的，行锁是通过对索引上的索引项加锁来实现的，



行锁（**Record Lock**）：锁定单个行记录的锁，防止其他事务对此行进行update和delete。在

RC、RR隔离级别下都支持。

间隙锁（**Gap Lock** 前开后开）：锁定索引记录间隙（不含该记录），确保索引记录间隙不变，防止其他事

务在这个间隙进行insert，产生幻读。只存在于可重复读隔离级别，目的是为了解决可重复读隔离级别下幻读的现象。

临键锁（**Next-Key Lock **前开后闭区间）：行锁和间隙锁组合，同时锁住数据，并锁住数据前面的间隙Gap。

在RR隔离级别下支持。前开后闭区间



#### **5.4.2** **行锁**

1). 介绍

InnoDB实现了以下两种类型的行锁：

共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排它锁。

排他锁（X）：允许获取排他锁的事务更新数据，阻止其他事务获得相同数据集的共享锁和排他

锁

![](行锁.jpeg)

![](行锁语句.jpeg)

**默认情况下，InnoDB在 REPEATABLE READ事务隔离级别运行，InnoDB使用 next-key 锁进行搜**

**索和索引扫描，以防止幻读。**

**针对唯一索引进行检索时，对已存在的记录进行等值匹配时，将会自动优化为行锁。**

**InnoDB的行锁是针对于索引加的锁，不通过索引条件检索数据，那么InnoDB将对表中的所有记**

**录加锁，此时 就会升级为表锁。**



##### 共享锁+共享锁

| sessionA                                                     | sessionB                                                     | session c                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------- |
| begin;                                                       | begin;                                                       |                                   |
| select * from a where id=1 lock in share mode;(正常查询,行锁共享锁) |                                                              |                                   |
|                                                              |                                                              | ![](行锁_共享共享_lockdata1.jpeg) |
|                                                              | select * from a where id=1 lock in share mode;(正常查询,行锁共享锁) |                                   |
|                                                              |                                                              | ![](行锁_共享共享_lockdata2.jpeg) |
|                                                              | commit;                                                      |                                   |
|                                                              |                                                              | ![](行锁_共享共享_lockdata1.jpeg) |
| commit;                                                      |                                                              |                                   |
|                                                              |                                                              | emptyset                          |

##### 共享锁+排他锁

如果一个select * from a where id=1 lock in share mode;，一个session执行的是update a set approval_status=7 where id=1;

意向锁会有两个一IS个IX，会有两条记录。

一个事务里 相同的意向锁（IS，IX）只会加一次。

| sessionA                                                     | sessionB                                                     | session c                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------- |
| begin;                                                       | begin;                                                       |                                   |
| select * from a where id=1 lock in share mode;(正常查询,行锁共享锁) |                                                              |                                   |
|                                                              | update a set approval_status=7 where id=2;(正常执行，因为和事务a的id不一样，锁定了两条不一样的记录的行锁) |                                   |
|                                                              |                                                              | ![](行锁_共享排他_lockdata1.jpeg) |
|                                                              | update a set approval_status=7 where id=1;(阻塞，id相同的a事务没提交) |                                   |
|                                                              |                                                              | ![](行锁_共享排他_lockdata2.jpeg) |
| commit;                                                      |                                                              |                                   |
|                                                              | update xx where id=1; 执行成功                               |                                   |
|                                                              | commit;                                                      |                                   |



##### 排他锁+排他锁

如果有两个update 同时update一张表，意向锁有两条



| sessionA                                             | sessionB                                         | session c                         |
| ---------------------------------------------------- | ------------------------------------------------ | --------------------------------- |
| begin;                                               | begin;                                           |                                   |
| update a set approval_status=7 where id=1;(正常执行) |                                                  |                                   |
|                                                      | update a set approval_status=8 where id=1;(阻塞) |                                   |
|                                                      |                                                  | ![](行锁_排他排他_lockdata2.jpeg) |
| commit;                                              |                                                  |                                   |
|                                                      | 事务a提交后，事务b update 提交                   |                                   |
|                                                      | commit;                                          |                                   |



##### where条件没有加索引的情况



```
-- 添加name列
ALTER TABLE a ADD COLUMN name VARCHAR(200);

delete from a;

INSERT INTO `wifi`.`a`(`uid`, `approval_status`, `id`, `name`) VALUES ('a1', 1, 1, 'alice');
INSERT INTO `wifi`.`a`(`uid`, `approval_status`, `id`, `name`) VALUES ('a2', 1, 2, 'ben');

```

| sessionA                                                     | sessionB                                          | session c                  |
| ------------------------------------------------------------ | ------------------------------------------------- | -------------------------- |
| begin;                                                       | begin;                                            |                            |
| update a set name='ricky' where name='alice';(正常执行,但是锁住了整表的所有记录) |                                                   |                            |
|                                                              |                                                   | ![](行锁_升级为表锁1.jpeg) |
|                                                              | update a set approval_status=8 where id=1;(阻塞,) |                            |
|                                                              |                                                   | ![](行锁_升级为表锁2.jpeg) |
| commit;                                                      |                                                   |                            |
|                                                              | 事务a提交后，事务b update 提交                    |                            |

##### 非唯一索引上进行等值查询



在RR隔离级别下支持。

| lock_type | lock_mode（**Next-Key Lock**） | lock_mode（**Record Lock**) | lock_mode（**Gap Lock**) |
| --------- | ------------------------------ | --------------------------- | ------------------------ |
| RECORD    | S                              | S,REC_NOT_GAP               | S,GAP                    |
| RECORD    | X                              | X,REC_NOT_GAP               | X,GAP                    |



```
-- 测试数据
CREATE TABLE `a_detail`  (
  `approval_status` int(0) NULL DEFAULT 1 COMMENT '审核状态(1:审核中;2:通过)',
  `id` bigint(0) NOT NULL AUTO_INCREMENT COMMENT '主键 id',
  `auid` varchar(200) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL,
  `age` int(0) NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `idx_aid`(`auid`) USING BTREE,
  INDEX `idx_age`(`age`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 74 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci COMMENT = '审核记录明细表' ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of a_detail
-- ----------------------------
INSERT INTO `a_detail` VALUES (1, 1, 'a1', 22);
INSERT INTO `a_detail` VALUES (1, 3, 'a1', 23);
INSERT INTO `a_detail` VALUES (1, 5, 'a2', 25);
INSERT INTO `a_detail` VALUES (1, 7, 'a2', 27);
INSERT INTO `a_detail` VALUES (1, 8, 'a3', 28);
INSERT INTO `a_detail` VALUES (1, 10, 'a4', 30);
INSERT INTO `a_detail` VALUES (1, 11, 'a5', 30);
```

```
select * from a_detail where age=25 lock in share mode;
```

```
mysql> select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from performance_schema.data_locks;
+---------------+-------------+------------+-----------+---------------+-----------+
| object_schema | object_name | index_name | lock_type | lock_mode     | lock_data |
+---------------+-------------+------------+-----------+---------------+-----------+
| wifi          | a_detail    | NULL       | TABLE     | IS            | NULL      |
| wifi          | a_detail    | idx_age    | RECORD    | S             | 25, 5     |
| wifi          | a_detail    | PRIMARY    | RECORD    | S,REC_NOT_GAP | 5         |
| wifi          | a_detail    | idx_age    | RECORD    | S,GAP         | 27, 7     |
+---------------+-------------+------------+-----------+---------------+-----------+
4 rows in set (0.00 sec)


--  正常
INSERT INTO `wifi`.`a_detail`(`approval_status`, `id`, `auid`, `age`) VALUES (1, 103, 'a2', 22);
--  阻塞
INSERT INTO `wifi`.`a_detail`(`approval_status`, `id`, `auid`, `age`) VALUES (1, 104, 'a2', 23);
--  阻塞
INSERT INTO `wifi`.`a_detail`(`approval_status`, `id`, `auid`, `age`) VALUES (1, 105, 'a2', 24);


--  阻塞
INSERT INTO `wifi`.`a_detail`(`approval_status`, `id`, `auid`, `age`) VALUES (1, 99, 'a2', 25);  



--  阻塞
INSERT INTO `wifi`.`a_detail`(`approval_status`, `id`, `auid`, `age`) VALUES (1, 100, 'a2', 26);

--  成功
INSERT INTO `wifi`.`a_detail`(`approval_status`, `id`, `auid`, `age`) VALUES (1, 101, 'a2', 27);
--  成功
INSERT INTO `wifi`.`a_detail`(`approval_status`, `id`, `auid`, `age`) VALUES (1, 102, 'a2', 28);

```

注意age=23 插入失败， age=27插入成功。是因为 临键锁（**Next-Key Lock**）（零件所）| 25, 5   2



但是同样是age=23，id=1却可以插入成功，不会阻塞

```
INSERT INTO `wifi`.`a_detail`(`approval_status`, `id`, `auid`, `age`) VALUES (1, 2, 'a2', 23);

```

如果age=27

```
INSERT INTO `wifi`.`a_detail`(`approval_status`, `id`, `auid`, `age`) VALUES (1, 6, 'a2', 27);
```







在对非唯一索引上进行等值查询，比如age=25，锁的范围是【3-25） 25  （25-27）,所以这种手法不正确。

二级索引要结合主键索引 索引树的分布 来判断临界数据是否可以插入，

有关二级索引 的 间隙锁是否能插入 临界数据，要以  

二级索引树是按照二级索引值（age列）按顺序存放的，在相同的二级索引值情况下， 再按主键 id 的顺序存放。知道了这个前提，我们才能知道执行插入语句的时候，插入的位置的下一条记录是谁。

判断，如果 二级索引（age+id）  组合在间隙锁内，就不能插入。如果（age+id）  不在间隙锁内，就可以插入。

参考：https://www.xiaolincoding.com/mysql/lock/how_to_lock.html#%E9%9D%9E%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95%E7%AD%89%E5%80%BC%E6%9F%A5%E8%AF%A2



如果出a事务用到了b事务的间隙锁 ，a是通过update语句， b是通过lock in share mode 加上的，死锁的时候，不分S，X，只要一个事务的间隙锁与另一个事务用到的间隙锁冲突就会产生阻塞，

如果事务a和事务b 分别执行相同的代码，最终会死锁。
