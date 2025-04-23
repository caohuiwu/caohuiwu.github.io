---
title: 《mysql》limit
date: 2020-08-15 12:09:31
categories:
  - [mysql]
---

    这是“mysql”系列的第五篇文章，主要介绍的是limit。

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

本章主要围绕InnoDB存储引擎死锁相关的一些概念、产生死锁的原因、死锁场景以及死锁的处理策略。

# 二、limit
创建如下表：
```
CREATE TABLE t (
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    key1 VARCHAR(100),
    common_field VARCHAR(100),
    PRIMARY KEY (id),
    KEY idx_key1 (key1)
) Engine=InnoDB CHARSET=utf8;
```
表t包含3个列，<code class='orange'>id列</code>是主键，<code class='orange'>key1列</code>是二级索引列。表中包含1万条记录。

## 2.1、limit查询过程
MySQL是在<code class='orange'>server层准备向客户端发送记录的时候</code>才会去处理LIMIT子句中的内容。拿下边这个语句举例子：
```
SELECT * FROM t ORDER BY key1 LIMIT 5000, 1;
```
如果使用<code class='orange'>idx_key1</code>执行上述查询，那么MySQL会这样处理：
- server层向InnoDB要<code class='orange'>第1条</code>记录，InnoDB从<code class='orange'>idx_key1</code>中获取到<code class='orange'>第1条</code>二级索引记录，然后进行回表操作得到完整的聚簇索引记录，然后返回给server层。server层准备将其发送给客户端，此时发现还有个<code class='orange'>LIMIT 5000, 1</code>的要求，意味着符合条件的记录中的第5001条才可以真正发送给客户端，所以在这里先做个统计，我们假设server层维护了一个称作<code class='orange'>limit_count</code>的变量用于统计已经跳过了多少条记录，此时就应该将<code class='orange'>limit_count</code>设置为1。
- server层再向InnoDB要下一条记录，InnoDB再根据二级索引记录的<code class='orange'>next_record</code>属性找到下一条二级索引记录，再次进行回表得到完整的聚簇索引记录返回给server层。server层在将其发送给客户端的时候发现<code class='orange'>limit_count</code>才是1，所以就放弃发送到客户端的操作，将<code class='orange'>limit_count</code>加1，此时<code class='orange'>limit_count</code>变为了<code class='my-code'>2</code>。
- ... 重复上述操作
- 直到limit_count等于5000的时候，server层才会真正的将InnoDB返回的完整聚簇索引记录发送给客户端。

从上述过程中我们可以看到，由于MySQL中是在实际向客户端发送记录前才会去判断LIMIT子句是否符合要求，所以如果使用二级索引执行上述查询的话，意味着要进行5001次回表操作。server层在进行执行计划分析的时候会觉得执行这么多次回表的成本太大了，还不如直接全表扫描+filesort快呢，所以就选择了后者执行查询。


## 2.2、怎么办？
由于MySQL实现LIMIT子句的局限性，在处理诸如LIMIT 5000, 1这样的语句时就无法通过使用二级索引来加快查询速度了么？其实也不是，只要把上述语句改写成：
```
SELECT * FROM t, (SELECT id FROM t ORDER BY key1 LIMIT 5000, 1) AS d
WHERE t.id = d.id;
```
这样，SELECT id FROM t ORDER BY key1 LIMIT 5000, 1作为一个子查询单独存在，由于该子查询的查询列表只有一个id列，MySQL可以通过仅扫描二级索引idx_key1执行该子查询，然后再根据子查询中获得到的主键值去表t中进行查找。

这样就省去了前5000条记录的回表操作，从而大大提升了查询效率！



参考文章：
[MySQL的LIMIT这么差劲的吗](https://mp.weixin.qq.com/s?__biz=MzIxNTQ3NDMzMw==&mid=2247486044&idx=1&sn=499e047f7181969017f59b86abbc2c17&chksm=979683aea0e10ab8420c442b0a04487a523cf09ee7242cd5f23e0a2a294713d360ddccbc4c07&scene=21#wechat_redirect)

