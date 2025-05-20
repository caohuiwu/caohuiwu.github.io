---
title: 《mysql》order by
date: 2020-08-15 22:01:31
categories:
  - [mysql]
---

    这是“mysql”系列的第五篇文章，主要介绍的是order by。

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

# 二、order by 的执行顺序
MySQL 中每个线程在执行排序时，都会被分配一块区域 - <code class='orange'>sort buffer</code>，它的大小通过 <code class='orange'>sort_buffer_size</code> 控制。

MySQL 在进行 Order By 操作排序时，通常有<code class='orange'>两种排序方式</code>：
- 全字段排序
    - 将要查询的字段，全都存入 <code class='orange'>sort buffer</code> 中，然后对 <code class='orange'>sort buffer</code> 进行排序，然后将结果返回给客户端。
- Row_id 排序
    - 将被排序的字段和对应主键索引的 ID 放入，<code class='orange'>sort buffer</code> 中，然后对 <code class='orange'>sort buffer</code> 进行排序，最后额外进行一次回表操作查找额外的信息，然后将结果返回给客户端。


全字段排序和 Row_id 排序的主要区别在：
- sort buffer 存入的内容不同
- 回表查询的次数不一致。


对于 InnoDB 表来说，在内存足够的情况下，会优先选择全字段排序的方式。在内存不足的情况下，可能会借用外部文件进行排序<code class='orange'>filesort</code>。

但如果单行内容较大时，会导致拆分的外部文件过多，进行归并排序时，效率变低。此时会采用 Row_id 的排序方式。

对于 Memory 表来说，会优先选择 Row_id 的排序方式。


## 2.1、全字段排序


## 2.2、Row_id 排序
在单行查询字段很多时，在 <code class='orange'>sort buffer</code> 中仅仅保存必要的字段，最后额外再进行一次统一回表的操作，查询必要的信息。

这里可以将 <code class='orange'>SET max_length_for_sort_data = 16</code>; 改小



创建测试表，结构如下：
```
CREATE TABLE `t` ( 
    `id` int(11) NOT NULL, 
    `city` varchar(16) NOT NULL, 
    `name` varchar(16) NOT NULL, 
    `age` int(11) NOT NULL, 
    `addr` varchar(128) DEFAULT NULL, 
    PRIMARY KEY (`id`), 
    KEY `city` (`city`) 
) ENGINE=InnoDB;
```

执行如下SQL，是如何执行的呢？
```
SELECT city,name,age FROM t WHERE city='杭州' order by name limit 10000;
```
执行过程
- 初始化<code class='orange'>sort_buffer</code>，确定放入<code class='orange'>city,name,age</code>三个字段。
  - <code class='orange'>sort_buffer</code>属于<code class='my-code'>server</code>层，它是MySQL为每个线程分配的一块内存，用于排序操作。在执行排序操作时，MySQL会将需要排序的数据放入<code class='orange'>sort buffer</code>中进行排序，然后再将排序后的数据返回给客户端。
- 从索引city中找到第一个满足city='杭州'的主键ID。
- 回表查找到对应的行数据，将<code class='orange'>city,name,age</code>三个字段添加到<code class='orange'>sort_buffer</code>中
- 从索引取下一个ID
- 重复3、4步骤，直到所有的数据都放入了<code class='orange'>sort_buffer</code>内
- 对<code class='orange'>sort_buffer</code>内的数据按name进行快速排序
- 取10000，将结果返回。

## 2.1、排序量过大，sort_buffer存不下怎么办？
- 利用磁盘临时文件，辅助排序。

## 2.2、若sort_buffer内字段过多，怎么优化查询？
- 优化思路1：让数据有序，创建<code class='orange'>联合索引（city, name)</code>
- 再优化：联合索引，还需要回表，创建<code class='orange'>覆盖索引（city,name,age)</code>，需要的东西都可以获得


