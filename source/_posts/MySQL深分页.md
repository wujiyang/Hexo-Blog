---
title: MySQL 深分页问题
date: 2025-02-16 21:12:49
index_img: /img/MySQL深分页/0.png
tags:
    - MySQL
    - 深分页
categories:
    - MySQL
---  

> 分页是一个很普通的功能，基本有数据展示的地方就会有分页，那为什么要分页呢？分页可能会存在哪些问题呢？

<!-- more -->  


## 1. 前言
分页是一个很普通的功能，基本有数据展示的地方就会有分页，那为什么要分页呢？
- 从业务上来讲，即使系统返回所有数据，用户绝大多数情况下是不会看后面的数据的。
- 技术上，因为要考虑取数据的成本，目标服务器磁盘、内存、网络带宽，以及请求发起方自身是否能承受大批量数据。
日常做分页需求时，一般会用limit实现，但是当偏移量特别大的时候，查询效率就变得低下。本文将介绍一下百万级数据构建和查询分析的深分页问题，同时讨论一些优化方案来解决MySQL的深分页问题。
## 2. 数据构造
### 2.1 MySQL搭建
- Docker镜像拉取、启动MySQL容器、连接到MySQL登录
```bash
docker pull mysql
# --name指定容器名称，-p指定端口映射 -e 设置环境变量
docker run --name mysql -p 3307:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql
# 输入密码即可登录到常规的数据库环境中
docker exec -it mysql mysql -u root -p
```

### 2.2 数据插入
- 创建数据库
```sql
create database demo;
use demo
```
- 创建数据表
```sql
CREATE TABLE account (  
    id int(11) NOT NULL AUTO_INCREMENT COMMENT '主键Id',  
    name varchar(255) DEFAULT NULL COMMENT '账户名',  
    balance int(11) DEFAULT NULL COMMENT '余额',  
    create_time datetime NOT NULL COMMENT '创建时间',  
    update_time datetime NOT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',  
    PRIMARY KEY (id),  
    KEY idx_name (name),  
    KEY idx_update_time (update_time)
 ) ENGINE=InnoDB AUTO_INCREMENT=1570068 DEFAULT CHARSET=utf8 ROW_FORMAT=REDUNDANT COMMENT='账户表';
```

- 插入数据
    - 首先初始化插入一条数据，后面自动生成的数据会以这条数据为基础进行生成
    ```sql
    insert into account values(1, "name_1", 1000, "2025-02-08 00:00:00", "2025-02-08 00:01:00")
    set @i=1;
    ```
    - 重复拷贝一下逻辑执行十多次，即可生成符合预期的数据量级；
    ```sql
    insert into account(name, balance, create_time, update_time) select
        concat('user_',@i:=@i+1),           
        FLOOR(20+RAND() *(5000000 - 20 + 1)) as balance,  
        date_add(create_time,interval +@i*cast(rand()*1000 as signed) SECOND), 
        date_add(date_add(create_time,interval +@i*cast(rand()*100 as signed) SECOND), interval + cast(rand()*1000000 as signed) SECOND) #生成有时间大顺序的随机的更新时间
    from account;

    方案参考：https://blog.csdn.net/mysqltop/article/details/105230327
    ```
    ```sql
    mysql> select count(*) from account;
    +----------+
    | count(*) |
    +----------+
    |  4194304 |
    +----------+
    1 row in set (0.50 sec)
    // 已经生成419w条数据
    ```
## 3. 查询场景分析
针对以上构建的数据表，进行一些数据查询的场景进行分析。
### 3.1 没有查询条件，是否有排序
查询200W条以后得数据，耗时分别如下：
```sql
mysql> select id, name, balance from account limit 2000000, 10;
+---------+--------------+---------+
| id      | name         | balance |
+---------+--------------+---------+
| 3897711 | user_1999998 | 3565249 |
| 3897712 | user_1999999 | 3063459 |
| 3897713 | user_2000000 | 1978909 |
| 3897714 | user_2000001 |  622278 |
| 3897715 | user_2000002 | 3144910 |
| 3897716 | user_2000003 | 3298460 |
| 3897717 | user_2000004 | 1632420 |
| 3897718 | user_2000005 | 2740997 |
| 3897719 | user_2000006 | 2933984 |
| 3897720 | user_2000007 | 4333621 |
+---------+--------------+---------+
10 rows in set (0.37 sec)

mysql> select id, name, balance from account order by id limit 2000000, 10;
+---------+--------------+---------+
| id      | name         | balance |
+---------+--------------+---------+
| 3897711 | user_1999998 | 3565249 |
| 3897712 | user_1999999 | 3063459 |
| 3897713 | user_2000000 | 1978909 |
| 3897714 | user_2000001 |  622278 |
| 3897715 | user_2000002 | 3144910 |
| 3897716 | user_2000003 | 3298460 |
| 3897717 | user_2000004 | 1632420 |
| 3897718 | user_2000005 | 2740997 |
| 3897719 | user_2000006 | 2933984 |
| 3897720 | user_2000007 | 4333621 |
+---------+--------------+---------+
10 rows in set (0.25 sec)
```

可以看到，带上id排序进行检索时，耗时略少一些，可以通过执行计划发现：一个走了全表扫描，一个走了索引扫描。

```sql
mysql> explain select id, name, balance from account limit 2000000, 10;
+----+-------------+---------+------------+------+---------------+------+---------+------+---------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra |
+----+-------------+---------+------------+------+---------------+------+---------+------+---------+----------+-------+
|  1 | SIMPLE      | account | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 4184857 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+------+---------+------+---------+----------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select id, name, balance from account order by id limit 2000000, 10;
+----+-------------+---------+------------+-------+---------------+---------+---------+------+---------+----------+-------+
| id | select_type | table   | partitions | type  | possible_keys | key     | key_len | ref  | rows    | filtered | Extra |
+----+-------------+---------+------------+-------+---------------+---------+---------+------+---------+----------+-------+
|  1 | SIMPLE      | account | NULL       | index | NULL          | PRIMARY | 4       | NULL | 2000010 |   100.00 | NULL  |
+----+-------------+---------+------------+-------+---------------+---------+---------+------+---------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
**结论：即使业务看起来没有任何查询条件且不需要排序，一般也可以加上主键排序，方便走索引检索**

### 3.2 查询有排序，但排序字段有无索引
由于表中update_time有索引，create_time无索引，所以分别用这两个字段做测试（理论上应该使用同样的数据和字段，分别建立一个有索引和无索引的表进行测试，这里偷懒一下）
```sql
mysql> select id, name, balance from account order by update_time desc limit 10000, 10;
+---------+--------------+---------+
| id      | name         | balance |
+---------+--------------+---------+
| 6118096 | user_4154864 | 1558264 |
| 6064556 | user_4101324 | 2858272 |
| 6018951 | user_4055719 | 4890846 |
| 5989855 | user_4026623 |  947321 |
| 6124844 | user_4161612 | 4356195 |
| 6015649 | user_4052417 | 3902340 |
| 6002851 | user_4039619 | 4286561 |
| 6086813 | user_4123581 |  173212 |
| 5932870 | user_3969638 | 2756974 |
| 6065088 | user_4101856 | 2274823 |
+---------+--------------+---------+
10 rows in set (0.04 sec)

mysql> select id, name, balance from account order by create_time desc limit 10000, 10;
+---------+--------------+---------+
| id      | name         | balance |
+---------+--------------+---------+
| 5990523 | user_4027291 | 3324558 |
| 6080659 | user_4117427 | 3808889 |
| 5889120 | user_3925888 |  751634 |
| 6131558 | user_4168326 | 3903289 |
| 6006085 | user_4042853 | 2574610 |
| 5915585 | user_3952353 | 2687494 |
| 5820701 | user_3857469 | 2927750 |
| 6048517 | user_4085285 | 2268068 |
| 6139663 | user_4176431 | 4055927 |
| 6123279 | user_4160047 | 2835164 |
+---------+--------------+---------+
10 rows in set (1.21 sec)
```

可以看到，使用索引字段进行排序时，耗时略少一些，可以通过执行计划发现：一个走了全表扫描，一个走了索引扫描。

```sql
mysql> explain select id, name, balance from account order by create_time desc limit 10000, 10;
+----+-------------+---------+------------+------+---------------+------+---------+------+---------+----------+----------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra          |
+----+-------------+---------+------------+------+---------------+------+---------+------+---------+----------+----------------+
|  1 | SIMPLE      | account | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 4184857 |   100.00 | Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+---------+----------+----------------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select id, name, balance from account order by update_time desc limit 10000, 10;
+----+-------------+---------+------------+-------+---------------+-----------------+---------+------+-------+----------+---------------------+
| id | select_type | table   | partitions | type  | possible_keys | key             | key_len | ref  | rows  | filtered | Extra               |
+----+-------------+---------+------------+-------+---------------+-----------------+---------+------+-------+----------+---------------------+
|  1 | SIMPLE      | account | NULL       | index | NULL          | idx_update_time | 5       | NULL | 10010 |   100.00 | Backward index scan |
+----+-------------+---------+------------+-------+---------------+-----------------+---------+------+-------+----------+---------------------+
1 row in set, 1 warning (0.00 sec)
```
**结论：给常用字段加索引，包括排序字段。**

### 3.3 排序字段有索引，但是深分页
```sql
mysql> explain select id, name, balance from account order by update_time desc limit 100000, 10;
+----+-------------+---------+------------+------+---------------+------+---------+------+---------+----------+----------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra          |
+----+-------------+---------+------------+------+---------------+------+---------+------+---------+----------+----------------+
|  1 | SIMPLE      | account | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 4184857 |   100.00 | Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+---------+----------+----------------+
```
当选择深分页时，即使有索引，执行计划也走的是全表扫描。因为mysql在处理时会自动优化分析，当sql查询条数超过一定比例时就会直接选择全表扫描读取。

### 3.4 带有查询条件，深分页耗时明显
当把sql查询语句中增加where查询条件时，发现深分页耗时明显增加
```sql
mysql> select id, name, balance from account where update_time > "2028-01-01" order by update_time limit 10000, 10;
+---------+-------------+---------+
| id      | name        | balance |
+---------+-------------+---------+
| 2116762 | user_350107 | 1399540 |
+---------+-------------+---------+
1 row in set (0.05 sec)

mysql> select id, name, balance from account where update_time > "2028-01-01" order by update_time limit 100000, 10;
+---------+-------------+---------+
| id      | name        | balance |
+---------+-------------+---------+
| 2720074 | user_887888 |  341752 |
+---------+-------------+---------+
1 row in set (0.23 sec)

mysql> select id, name, balance from account where update_time > "2028-01-01" order by update_time limit 1000000, 10;
+---------+--------------+---------+
| id      | name         | balance |
+---------+--------------+---------+
| 2871404 | user_1039218 | 2366353 |
+---------+--------------+---------+
1 row in set (2.99 sec)
```
### 3.5 深分页为什么变慢？
我们考虑如上的查询条件，分析执行计划如下：
```sql
mysql> explain select id, name, balance from account where update_time > "2028-01-01" order by update_time limit 1000000, 10;
+----+-------------+---------+------------+------+-----------------+------+---------+------+---------+----------+-----------------------------+
| id | select_type | table   | partitions | type | possible_keys   | key  | key_len | ref  | rows    | filtered | Extra                       |
+----+-------------+---------+------------+------+-----------------+------+---------+------+---------+----------+-----------------------------+
|  1 | SIMPLE      | account | NULL       | ALL  | idx_update_time | NULL | NULL    | NULL | 4184857 |    50.00 | Using where; Using filesort |
+----+-------------+---------+------------+------+-----------------+------+---------+------+---------+----------+-----------------------------+
```
这个SQL的执行流程：
1. 通过普通二级索引树idx_update_time，过滤update_time条件，找到满足条件的记录ID。
2. 通过ID，回到主键索引树，找到满足记录的行，然后取出展示的列（回表）
3. 扫描满足条件的100010行，然后扔掉前100000行，返回。
![](/img/MySQL深分页/1.png)
所以SQL变慢的原因主要有两个：
1. limit语句会先扫描offset+n行，然后再丢弃掉前offset行，返回后n行数据。也就是说limit 100000,10，就会扫描100010行，而limit 0,10，只扫描10行。
2. limit 100000,10 扫描更多的行数，也意味着回表更多的次数。

## 4. 优化方案
以上的SQL，回表了100010次，实际上，我们只需要10条数据，也就是我们只需要10次回表其实就够了。因此，我们可以通过减少回表次数来优化。
那么，如何减少回表次数呢？我们先来复习下B+树索引结构哈~
InnoDB中，索引分主键索引（聚簇索引）和二级索引
- 主键索引，叶子节点存放的是整行数据
- 二级索引，叶子节点存放的是主键的值。
![](/img/MySQL深分页/2.png)
如果我们能把查询条件，转移回到主键索引树，那就可以减少回表次数了。转移到主键索引树查询的话，查询条件得改为主键id了，所以优化思路需要往这个方面靠近。

### 4.1 子查询优化
原始查询SQL中的查询条件是update_time这个如何转换成些主键id呢？抽到子查询即可。
因为二级索引叶子节点是有主键ID的，所以我们直接根据update_time来查主键ID即可，同时我们把 limit 100000的条件，也转移到子查询中，具体SQL语句如下：
```sql
select id,name,balance FROM account where id > (select a.id from account a where a.update_time >= '2028-01-01' order by update_time limit 100000, 1) LIMIT 10
;
```
思想是对的，但是结果是错的，因为id的顺序和update_time的顺序没有关系，应该使用id in xxx才行
但是用id in xxx的时候，部分mysql版本的子查询不能有limit，需要升级mysql

### 4.2 延迟关联 INNER JOIN
延迟关联的优化思路，跟子查询的优化思路其实是一样的：都是把条件转移到主键索引树，然后减少回表。不同点是，延迟关联使用了inner join代替子查询。

```sql
SELECT  ac1.id, ac1.name, ac1.balance FROM account ac1 INNER JOIN (SELECT a.id FROM account a WHERE a.update_time >= '2028-01-01' ORDER BY a.update_time LIMIT 1000000, 10) AS  ac2 on ac1.id= ac2.id;
```
通过执行SQL语句，可以发现耗时对比明显，且结果集一致
```sql
mysql> select id, name, balance from account where update_time > "2028-01-01" order by update_time limit 1000000, 1;
+---------+--------------+---------+
| id      | name         | balance |
+---------+--------------+---------+
| 4666601 | user_2703369 |  518744 |
+---------+--------------+---------+
1 row in set (1.57 sec)

mysql> SELECT  ac1.id, ac1.name, ac1.balance FROM account ac1 INNER JOIN (SELECT a.id FROM account a WHERE a.update_time > '2028-01-01' ORDER BY a.update_time LIMIT 1000000, 1) AS  ac2 on ac1.id= ac2.id;
+---------+--------------+---------+
| id      | name         | balance |
+---------+--------------+---------+
| 4666601 | user_2703369 |  518744 |
+---------+--------------+---------+
1 row in set (0.14 sec)
```

### 4.3 标签记录法
limit 深分页问题的本质原因就是：偏移量（offset）越大，mysql就会扫描越多的行，然后再抛弃掉。这样就导致查询性能的下降。
其实我们可以采用标签记录法，就是标记一下上次查询到哪一条了，下次再来查的时候，从该条开始往下扫描。就好像看书一样，上次看到哪里了，你就折叠一下或者夹个书签，下次来看的时候，直接就翻到啦。
假设上一次记录到100000，则SQL可以修改为：
```sql
select  id,name,balance FROM account where id > 100000 order by id limit 10;
```
这样的话，后面无论翻多少页，性能都会不错的，因为命中了id索引。但是你，这种方式有局限性：需要一种类似连续自增的字段。

### 4.4 使用 Between...and...
很多时候，可以将limit查询转换为已知位置的查询，这样MySQL通过范围扫描between...and，就能获得到对应的结果。
如果知道边界值为100000，100010后，就可以这样优化：
```sql
select  id,name,balance FROM account where id between 100000 and 100010 order by id desc;
```

## 5. 参考资料
- 实战！聊聊如何解决MySQL深分页问题 https://juejin.cn/post/7012016858379321358
- 【得物技术】MySQL深分页优化 https://juejin.cn/post/6985478936683610149

## 6. 附录
以下是 MySQL 中常见的 type 类型及其含义：
1. system：这是最高级别的访问类型，表示 MySQL 只有一行数据，这行数据是从系统表中读取的。
2. const：这个类型表示 MySQL 在查询时使用了常量，通常是通过索引访问单个行时使用。
3. eq_ref：这个类型表示 MySQL 在查询时使用了唯一索引，通常是在连接操作中使用。
4. ref：这个类型表示 MySQL 在查询时使用了非唯一索引，通常是在连接操作中使用。
5. range：这个类型表示 MySQL 在查询时使用了索引范围查找，通常是在使用 BETWEEN、IN 或者 <、> 等操作符时使用。
6. index：这个类型表示 MySQL 在查询时使用了全索引扫描，通常是在查询结果集非常小的情况下使用。
7. all：这个类型表示 MySQL 在查询时进行了全表扫描，通常是在查询结果集非常大的情况下使用。