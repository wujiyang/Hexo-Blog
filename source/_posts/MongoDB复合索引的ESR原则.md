---
title: MongoDB复合索引的ESR原则
date: 2024-02-21 22:30:40
index_img: /img/MongoDB复合索引的ESR原则/1.png
tags:
    - MongoDB
    - 索引
categories:
    - MongoDB
---

提到索引，大家都不会陌生，它的重要性在数据库中不言而喻。常见的存储如MysQL，MongoDB都支持多种类型的索引。本文主要以实际工作中碰到的例子聊一下MongoDB中的复合索引。

<!-- more -->  

所谓复合索引，就是包含多个字段的索引，比如index(a,b)。复合索引在使用的过程中主要有两个原则： 
- 最左匹配原则 
- ESR原则 

### 1. 最左匹配原则  
最左匹配原则比较好理解，他和MySQL中的最左匹配原则一致，即最左优先：在检索数据时从复合索引的最左边开始匹配。

复合索引创建时一个基本的原则就是：将选择性最强的列放到最前面。

选择性最高指的是数据的重复值最少，因为区分度高的列能够很容易过滤掉很多的数据。如果组合索引中第一次能够过滤掉很多的数据，后面的索引查询的数据范围就小了很多了。

### 2. ESR原则
[The ESR (Equality, Sort, Range) Rules](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-rule/) 

简单来说就是我们在构建复合索引时，需要根据以下三项原则的顺序进行构建：  
- 等值查询字段放在最前面
- 中间放排序字段
- 最后是范围查询字段  

E 放在前面比较好理解，等值匹配过滤掉大量数据，「那为什么是 ESR 不是 ERS 呢?」
![](/img/MongoDB复合索引的ESR原则/1.png)  
如图所示，如果把范围匹配放在中间，那么后续我们排序的时候只能进行「内存排序」，内存排序是比较消耗资源的，数据量大时可能会面临着「多次的磁盘读取刷内存操作」，对性能影响比较显著。

### 3. 实际案例分析
以上两个原则看起来比较简单，但笔者在实际应用中还是踩了一些坑。

我有一个这样的mongo集合： 
``` go
type Demo struct {
// QueueID 队列id
QueueID string `bson:"queue_id,omitempty"`

// Status 任务状态
Status int `bson:"status,omitempty"`

// UpdateTime 更新时间
UpdateTime time.Time `bson:"update_time,omitempty"`

// 其他字段
xxxxx

} 
```

mongo在查询时需要执行类似如下条件的查询：
``` go
db.collections.find({ queue_id: 123, status: { $in: [1, 2, 3] } }).
sort({ updateTime: 1 }).limit(1)
```  
即：查询指定queue_id, 指定status范围，并按照updateTime进行升序的一条记录。   

在初步了解ESR原则后，我一想，这不就是queue_id为等值查询、updateTime为排序查询、status为范围查询的经典情况吗，大手一挥就创建了一个（queue_id, updateTime, status）的复合索引并愉快的上线了。

然而待数据量增大后，发现一个比较严重的问题，Mongo实例的CPU飙到了60%，排查原因发现索引执行不太正常。

参考官方资料后，发现in在执行的时候，可能是等值查询也可能是范围查询 
![](/img/MongoDB复合索引的ESR原则/2.png)    
- 如果in单独使用，就是等值查询
- 如果in和sort一起使用，就是范围查询（针对同一个字段）

所以在本例中，status的使用是等值查询，应该放在updateTime的前面，将索引修改为（queue_id, status, updateTime）后，CPU使用率即刻下降到30%左右。