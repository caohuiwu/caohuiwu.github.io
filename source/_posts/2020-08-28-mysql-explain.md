---
title: 《mysql》explain
date: 2020-08-28 12:09:31
categories:
  - [ 数据库, mysql, explain ]
---

    这是“mysql”系列的第十篇文章，主要介绍的是explain。

<style>
.my-code {
   color: green;
}
.orange {
   color: rgb(255, 53, 2)
}
.red {
   color: red
}
</style>

# 一、mysql

<code>MySQL</code> 是一种广泛使用的开源关系型数据库管理系统（RDBMS--Relational Database Management System）

<!-- more -->

# 二、explain

explain 是MySQL中用于分析SQL查询执行计划的工具。它显示MySQL如何执行一条查询（例如使用哪些索引、是否全表扫描、表的连接顺序等），帮助开发者优化查询性能。

## 2.1、如何使用explain？

示例

```
mysql> explain select * from t2 where (a between 1 and 1000) and (b between 50000 and 100000) order by b,a limit 1;
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t2    | NULL       | range | a,b           | a    | 5       | NULL | 1000 |    50.00 | Using index condition; Using where; Using filesort |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+----------------------------------------------------+
1 row in set, 1 warning (0.01 sec)

mysql>
```

## 2.2、explain参数详解

| id  | 	Columns	     | JSON Name       |                                                                	Meaning |
|:----|---------------|-----------------|------------------------------------------------------------------------:|
| 1	  | id            | 	select_id      |                	每个select子句的标识id，表示查询中执行select子句或操作表的顺序，id越大的table读取越在前面 |
| 2	  | select_type   | 	None	          |                       select语句的类型，如SIMPLE 简单查询、PRIMARY主查询、SUBQUERY 子查询等 |
| 3	  | table         | 	table_name     |                                                                   	当前表名 |
| 4	  | partitions	   | partitions      |                                                                  	匹配的分区 |
| 5	  | type          | 	access_type	   | 访问类型（关键指标，性能从优到差排序）：system > const > eq_ref > ref > range > index > ALL |
| 6	  | possible_keys | 	possible_keys	 |                                                                可能使用到的索引 |
| 7	  | key           | 	key	           |                                                          经过优化器评估最终使用的索引 |
| 8	  | key_len       | 	key_length     |                                                         	使用到的索引长度（越短越好） |
| 9	  | ref           | 	ref	           |                                                              引用到的上一个表的列 |
| 10	 | rows	         | rows	           |                                                          预估要扫描的行数（越小越好） |
| 11	 | filtered      | 	filtered	      |                                                             按表条件过滤行的百分比 |
| 12	 | Extra         | 	None           |                       	额外的信息说明（Using where，Using index，Using temporary等 |



## 2.3、重点解读

### 2.3.1、type列
访问的类型
<code class='orange'>NULL > system > const > eq_ref > ref > ref_or_null > index_merge > range > index > ALL</code>
要记住以下10个状态，（从左往右，越靠左边的越优秀）

- const：通过主键或唯一索引直接找到一行（最优）。
- ref：使用非唯一索引查找。
- range：索引范围扫描（如between，and）。
- ALL：全表扫描（需优化，通常因无索引或索引未命中）。

### 2.3.2、extra列
- Using index：查询仅通过索引获取数据（覆盖索引，高效）。
- Using temporary：使用了临时表（常见于 group by或 order by未用索引），
- Using filesort：需额外排序（考虑优化order by）


## 2.4、总结
- 全表扫描（type：all）：是性能杀手，尽量通过索引避免。
- 覆盖索引（Using index）：能显著减少I/O操作。
- 联合索引的顺序需匹配查询条件（最左前缀原则）。
