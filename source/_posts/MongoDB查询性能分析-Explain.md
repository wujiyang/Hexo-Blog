---
title: MongoDB查询性能分析-Explain
date: 2024-06-30 17:36:48
index_img: /img/MongoDB查询性能分析-Explain/explain.png
tags:
    - 性能分析
    - Explain
categories:
    - MongoDB
---

在日常的MongoDB开发和使用中，不可避免的会碰到一些慢查询的问题，针对这些问题，我们一般可以使用 explain 方法来获取查询的执行计划和性能分析信息。使用explain方法可以帮助我们理解查询是如何执行的，以及识别性能瓶颈和优化对象。
<!-- more -->   

## 1. Explain简介
### 1.1 基本使用
使用 explain 方法；你可以在任何查询、更新或删除操作中使用 explain 方法。以下是一些基本示例。   
假设你有一个集合 myCollection，你可以如下进行操作：
- 对查询操作使用 explain   
```sql 
db.myCollection.find({ a: { $gt: 1, $lt: 2 } }).explain() 
```
这将返回查询的执行计划，包括扫描类型、索引使用情况和其他详细信息。


- 对更新操作使用 explain   
```sql 
db.myCollection.updateMany({ a: { $gt: 1, $lt: 2 } }, { $set: { b: 1 } }).explain() 
```

- 对删除操作使用 explain    
```sql 
db.myCollection.deleteMany({ a: { $gt: 1, $lt: 2 } }).explain() 
```

### 1.2 输出含义解析
explain 方法返回一个包含以下信息的文档：

- queryPlanner：描述查询计划的信息。
    - plannerVersion：查询规划器的版本。
    - namespace：查询的命名空间（即数据库和集合）。
    - indexFilterSet：是否设置了索引过滤。
    - parsedQuery：解析后的查询条件。
    - winningPlan：实际执行的查询计划。<font color="red">（winningPlan比较重要，一般可以看出当前查询语句使用的索引计划等，方便问题排查）</font>
    - rejectedPlans：被拒绝的查询计划（如果有）。 

- executionStats：提供查询执行的统计信息（仅在使用 executionStats 或 allPlansExecution 模式时）。
    - executionSuccess：查询是否成功执行。
    - nReturned：返回的文档数量。
    - executionTimeMillis：查询执行时间（毫秒）。
    - totalKeysExamined：检查的索引键数量。
    - totalDocsExamined：检查的文档数量。
    - serverInfo：有关服务器的信息。

### 1.3 模式差别
explain 方法有三种模式, 可以通过传递参数来指定模式。

- queryPlanner（默认）：只返回查询计划，不包含实际执行统计信息。  
- executionStats：返回查询计划和执行统计信息。  
- allPlansExecution：返回所有执行计划和它们的执行统计信息。

### 1.4 具体案例分析   
```sql 
db.myCollection.find({ a: { $gt: 1, $lt: 2 } }).explain("executionStats") 
```

以下是一个 explain 方法的示例输出：

```json
{
  "queryPlanner": {
    "plannerVersion": 1,
    "namespace": "myDatabase.myCollection",
    "indexFilterSet": false,
    "parsedQuery": {
      "a": {
        "$gt": 1,
        "$lt": 2
      }
    },
    "winningPlan": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",
        "keyPattern": {
          "a": 1
        },
        "indexName": "a_1",
        "isMultiKey": false,
        "multiKeyPaths": {
          "a": []
        },
        "isUnique": false,
        "isSparse": false,
        "isPartial": false,
        "indexVersion": 2,
        "direction": "forward",
        "indexBounds": {
          "a": [
            "(1.0, 2.0)"
          ]
        }
      }
    },
    "rejectedPlans": []
  },
  "executionStats": {
    "executionSuccess": true,
    "nReturned": 3,
    "executionTimeMillis": 2,
    "totalKeysExamined": 3,
    "totalDocsExamined": 3,
    "executionStages": {
      "stage": "FETCH",
      "nReturned": 3,
      "executionTimeMillisEstimate": 0,
      "works": 4,
      "advanced": 3,
      "needTime": 0,
      "needFetch": 0,
      "saveState": 0,
      "restoreState": 0,
      "isEOF": 1,
      "docsExamined": 3,
      "alreadyHasObj": 0,
      "inputStage": {
        "stage": "IXSCAN",
        "nReturned": 3,
        "executionTimeMillisEstimate": 0,
        "works": 4,
        "advanced": 3,
        "needTime": 0,
        "needFetch": 0,
        "saveState": 0,
        "restoreState": 0,
        "isEOF": 1,
        "keyPattern": {
          "a": 1
        },
        "indexName": "a_1",
        "isMultiKey": false,
        "multiKeyPaths": {
          "a": []
        },
        "isUnique": false,
        "isSparse": false,
        "isPartial": false,
        "indexVersion": 2,
        "direction": "forward",
        "indexBounds": {
          "a": [
            "(1.0, 2.0)"
          ]
        },
        "keysExamined": 3,
        "seeks": 1,
        "dupsTested": 0,
        "dupsDropped": 0,
        "seenInvalidated": 0
      }
    }
  },
  "serverInfo": {
    "host": "localhost",
    "port": 27017,
    "version": "4.2.0",
    "gitVersion": "2b62a68cfa4efae44c9a66b09a38df7e48fdbe73"
  }
}
```

通过分析 explain 输出，你可以了解查询是如何执行的，并据此优化索引和查询语句。

## 2 ExecutionStats中的时间分析
在 MongoDB 的 explain 输出中，executionStats 部分包含了查询的执行统计信息。其中包括执行评估时间（executionTimeMillisEstimate）和实际执行时间（executionTimeMillis）。这两个时间指标有不同的含义和用途：

- executionTimeMillisEstimate：
    - 含义：执行评估时间（或执行时间估计）。
    - 作用：这是对特定阶段或步骤的执行时间的估计值。这个值是由 MongoDB 查询引擎内部计算得出的，用于提供一个大致的时间估计，帮助了解查询各个阶段所消耗的时间。
    - 使用场景：在分析查询计划中各个阶段的性能时，executionTimeMillisEstimate 可以帮助确定每个阶段的大致执行时间，从而识别可能的性能瓶颈。
- executionTimeMillis：
    - 含义：实际执行时间。
    - 作用：这是查询从开始到结束实际花费的时间，以毫秒为单位。这个时间包括整个查询执行过程中的所有步骤和阶段。
    - 使用场景：executionTimeMillis 提供了查询的总体执行时间，用于衡量查询的整体性能。  

假设有以下 explain 输出：
```json 
{
  "executionStats": {
    "executionSuccess": true,
    "nReturned": 3,
    "executionTimeMillis": 5,
    "totalKeysExamined": 3,
    "totalDocsExamined": 3,
    "executionStages": {
      "stage": "FETCH",
      "nReturned": 3,
      "executionTimeMillisEstimate": 2,
      "works": 4,
      "advanced": 3,
      "needTime": 0,
      "needFetch": 0,
      "saveState": 0,
      "restoreState": 0,
      "isEOF": 1,
      "docsExamined": 3,
      "alreadyHasObj": 0,
      "inputStage": {
        "stage": "IXSCAN",
        "nReturned": 3,
        "executionTimeMillisEstimate": 1,
        "works": 4,
        "advanced": 3,
        "needTime": 0,
        "needFetch": 0,
        "saveState": 0,
        "restoreState": 0,
        "isEOF": 1,
        "keyPattern": {
          "a": 1
        },
        "indexName": "a_1",
        "isMultiKey": false,
        "multiKeyPaths": {
          "a": []
        },
        "isUnique": false,
        "isSparse": false,
        "isPartial": false,
        "indexVersion": 2,
        "direction": "forward",
        "indexBounds": {
          "a": [
            "(1.0, 2.0)"
          ]
        },
        "keysExamined": 3,
        "seeks": 1,
        "dupsTested": 0,
        "dupsDropped": 0,
        "seenInvalidated": 0
      }
    }
  }
}
```
在这个例子中：
- executionTimeMillis：整个查询实际执行时间为 5 毫秒。
- executionTimeMillisEstimate（FETCH 阶段）：这个阶段的执行时间估计为 2 毫秒。
- executionTimeMillisEstimate（IXSCAN 阶段）：这个阶段的执行时间估计为 1 毫秒。

通过结合这两个时间指标，可以更好地优化查询，改进性能。例如，如果某个阶段的 executionTimeMillisEstimate 特别高，可以考虑优化该阶段的索引或查询逻辑。

## 3. WinningPaln中的Stage有哪些

在 MongoDB 查询计划（explain 输出的 winningPlan 部分）中，查询计划由多个阶段（Stage）组成，每个阶段代表查询执行过程中的一个步骤。主要包括如下类型：

- COLLSCAN（Collection Scan）：
    - 扫描整个集合的所有文档，通常是因为没有可用的索引。
    - 使用场景：没有适用的索引，或者是全表扫描。- 
- IXSCAN（Index Scan）：
    - 扫描索引，查找符合条件的索引条目。
    - 使用场景：查询条件可以利用索引。
- FETCH：
    - 从磁盘中读取文档，通常跟在 IXSCAN 之后，通过索引查找后获取实际文档。
    - 使用场景：通过索引查找后，需要获取完整文档。
- LIMIT：
    - 限制结果集的数量。
    - 使用场景：查询中使用 limit 操作。
- SKIP：
    - 跳过结果集中的前若干个文档。
    - 使用场景：查询中使用 skip 操作。
- SORT：
    - 对结果集进行排序。
    - 使用场景：查询中使用 sort 操作。
- PROJECTION：
    - 对结果集进行字段投影，只返回指定的字段。
    - 使用场景：查询中使用字段选择器。
- SHARD_MERGE：
    - 从多个分片中合并结果集。
    - 使用场景：在分片集群环境中，跨分片查询。
- TEXT：
    - 执行全文搜索查询。
    - 使用场景：查询中使用 $text 操作符。
- GEO_NEAR：
    - 执行地理位置查询，查找距离某点最近的文档。
    - 使用场景：查询中使用 $near 或 $geoNear 操作符。
- GROUP：
    - 执行分组操作，类似于 SQL 的 GROUP BY。
    - 使用场景：查询中使用聚合管道中的 $group 操作符。
- UNWIND：
    - 拆分数组字段，将每个数组元素作为单独的文档。
    - 使用场景：查询中使用聚合管道中的 $unwind 操作符。
- LOOKUP：
    - 执行联表查询，将一个集合中的文档与另一个集合中的文档进行连接。
    - 使用场景：查询中使用聚合管道中的 $lookup 操作符。
- REDUCE：
    - 在 MapReduce 操作中，执行 reduce 阶段。
    - 使用场景：使用 MapReduce 执行复杂聚合操作。  


假设有一个集合 myCollection，使用以下查询并结合 explain 查看查询计划：  
```sql 
db.myCollection.find({ a: { $gt: 1, $lt: 2 } }).sort({ b: -1 }).limit(5).explain()
```

得到的 explain 输出可能包含以下阶段：

```json 
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "LIMIT",
      "limitAmount": 5,
      "inputStage": {
        "stage": "SORT",
        "sortPattern": { "b": -1 },
        "inputStage": {
          "stage": "FETCH",
          "inputStage": {
            "stage": "IXSCAN",
            "keyPattern": { "a": 1 },
            "indexName": "a_1",
            "isMultiKey": false,
            "direction": "forward",
            "indexBounds": {
              "a": [
                "(1.0, 2.0)"
              ]
            }
          }
        }
      }
    }
  }
}
```
- LIMIT 阶段：限制结果集数量为 5。
- SORT 阶段：按 b 字段进行降序排序。
- FETCH 阶段：通过索引查找后，从磁盘中读取完整文档。
- IXSCAN 阶段：扫描 a_1 索引，查找 a 字段在 (1, 2) 范围内的索引条目。   


通过理解这些阶段，可以更好地分析和优化查询性能。例如，确保查询尽可能利用索引，减少全表扫描（COLLSCAN）。