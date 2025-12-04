---
title: 《mysql-源码分析》select流程
date: 2020-08-13 13:00:00
categories:
  - [mysql]
tags:
  - [mysql源码]
---
<style>
.orange {
   color: orange
}
.red {
   color: red
}
code {
   color: #0ABF5B;
}
</style>

    这是“mysql”系列的第五篇文章，主要介绍的是select流程部分源码。

# 一、mysql

<code>MySQL</code> 是一种广泛使用的开源关系型数据库管理系统（RDBMS--Relational Database Management System）

<!-- more -->


# 二、一条select语句的执行过程
SQL的执行过程如下：
```text
连接器 -> 分析器 -> 优化器（负责生成执行计划）  -> 执行器 -> innodb
```

## 2.1、命令执行
命令执行<code>mysql_execute_command</code>：（`sql/sql_parse.cc`）
```cpp
int
mysql_execute_command(THD *thd, bool first_level)
{
    switch (lex->sql_command) {
      /* ...... */
      case SQLCOM_SELECT: //查询
      {
        DBUG_EXECUTE_IF("use_attachable_trx",
                        thd->begin_attachable_ro_transaction(););
        // 重置查询成本统计
        thd->clear_current_query_costs();
        // 执行预检查：权限检查、视图权限检查、表结构验证
        res= select_precheck(thd, lex, all_tables, first_table);
        // 执行查询
        if (!res)
          res= execute_sqlcom_select(thd, all_tables);
        // 保存查询成本
        thd->save_current_query_costs();
        DBUG_EXECUTE_IF("use_attachable_trx",
                        thd->end_attachable_transaction(););
        break;
      }
      case SQLCOM_INSERT: //插入
      case SQLCOM_REPLACE_SELECT:
      case SQLCOM_INSERT_SELECT:
      {
        assert(first_table == all_tables && first_table != 0);
        assert(lex->m_sql_cmd != NULL);
        res= lex->m_sql_cmd->execute(thd);
        break;
      }
  }
}
```

## 2.2、执行SQL查询语句（execute_sqlcom_select）
`execute_sqlcom_select(thd, all_tables);`是处理SELECT查询的核心函数之一，负责执行SQL查询语句并返回结果。
```cpp
//sql/sql_parse.cc
static bool execute_sqlcom_select(THD *thd, TABLE_LIST *all_tables)
{
    if (lex->is_explain()) //如果是explain语句
    { 
      Query_result *const result= new Query_result_send; //Query_result_send标准结果集处理类，负责将数据发送到客户端
      if (!result)
        return true; /* purecov: inspected */
      res= handle_query(thd, lex, result, 0, 0);
    } else {
      Query_result *result= lex->result;
      if (!result && !(result= new Query_result_send()))
        return true;                            /* purecov: inspected */
      Query_result *save_result= result;
      Query_result *analyse_result= NULL;
      if (lex->proc_analyse)
      {
        if ((result= analyse_result= new Query_result_analyse(result, lex->proc_analyse)) == NULL)
          return true;
      }
      res= handle_query(thd, lex, result, 0, 0); //执行查询，并传入结果集对象
    }
}
```

## 2.3、执行查询（handle_query）
`handle_query`是`MYSQL`处理 `SQL` 查询的核心函数，其执行流程涵盖上下文分析、元数据准备、查询优化、执行计划生成及结果返回。

```cpp
// sql/sql_select.cc
bool handle_query(THD *thd, LEX *lex, Query_result *result,
                  ulonglong added_options, ulonglong removed_options)
{
    //
    SELECT_LEX *const select= unit->first_select();
    // 判断当前SQL查询单元（unit）是否为简单查询（不包含子查询、union、派生表等复杂结构）
    const bool single_query= unit->is_simple(); 
    // 1. 负责验证语法树的合法性、绑定字段、扩展通配符，为优化器生成执行计划提供基础。
    if (select->prepare(thd)) 
    // 2. 锁定表，
    if (lock_tables(thd, lex->query_tables, lex->table_count, 0))
    // 3. 存储查询结果到缓存（MYSQL 8.0+ 已移除）
    query_cache.store_query(thd, lex->query_tables);
    // 4. 优化器 生成执行计划，选择最优的查询执行路径
    if (select->optimize(thd))
    // 5. 根据优化后的执行计划，从存储引擎读取数据并返回结果
    select->join->exec();
}                  
                  
```
- `SELECT_LEX`：表示一个查询块（`query block`），即一个完整的`SELECT`语句。

可以分成3个执行阶段：
1. 准备阶段
2. 优化阶段
3. 执行阶段

### 2.3.1、准备阶段
`SELECT_LEX::prepare(THD *thd)`是SELECT查询块的准备阶段，负责：
- 名称解析（解析表名、列名、函数等）
- 语义检查（验证语法树是否合法）
- 权限检查（用户是否有权限访问表或列）
- 视图展开（如果查询涉及视图，展开视图为底层表）
- 子查询优化（处理子查询的关联上下文）
- 派生表物化（为派生表生成临时表结构）

`prepare(thd)`方法的执行，在`sql/sql_resolver.cc`文件内。
```cpp
// sql_resolver.cc
bool SELECT_LEX::prepare(THD *thd) {
  // 1. 解析表名、列名、函数名
  if (resolve_derived(thd, this)) return true;

  // 2. 处理 WHERE 子句的条件
  if (where && where->fix_fields(thd, &where)) return true;

  // 3. 处理 SELECT 列表的列
  for (Item *item : item_list) {
    if (item->fix_fields(thd, &item)) return true;
  }

  // 4. 处理 GROUP BY、HAVING 等子句
  // ... 其他逻辑

  return false;
}
```

### 2.3.2、优化阶段
`optimize(thd)`方法是查询处理的**核心优化阶段**，负责生成高效的执行计划，分为以下步骤：
- **逻辑优化**
  - 条件简化：去除冗余表达式（如1=1），合并相同条件。
  - 子查询优化：将 `in` 子查询转换为半连接（semi-join）或物化临时表。
  - 视图合并：将视图展开为基表查询。
  - 连接顺序调整：通过代价模型选择最优的表连接顺序。
- **物理优化**
  - 访问路径选择：决定使用全表扫描还是索引扫描，基于统计信息（如索引基数）估算代价。
  - 临时表策略：处理 `group by distinct`或排序时，选择内存临时表或磁盘临时表。
- **代价估算**
  - 使用存储引擎（如Innodb）提供的统计信息

关键源码判断示例：
```cpp
// sql/sql_select.cc
bool SELECT_LEX::optimize(THD *thd) {
  JOIN *join = new JOIN(thd, this, ...);
  if (join->optimize()) {  // 调用 JOIN 的优化逻辑
    delete join;
    return true;  // 优化失败
  }
  return false;  // 优化成功
}

// sql/sql_optimizer.cc
bool JOIN::optimize() {
  // 逻辑优化（如子查询转换）
  if (optimize_cond()) return true;

  // 物理优化（如索引选择）
  if (make_join_plan()) return true;

  // 代价估算与执行计划生成
  if (estimate_rowcount()) return true;

  return false;
}
```

### 2.3.3、执行阶段
`select->join->exec();`是查询执行阶段的核心入口，负责将优化后的执行计划转换为实际的数据操作。`JOIN::exec()`源码如下：
```cpp
// sql/sql_executor.cc
bool JOIN::exec() {
  if (error) return true; // 已存在错误则直接返回

  // 初始化执行环境
  if (init_execution()) return true;

  // 执行主循环：读取数据并处理
  List<Item> *columns = &fields_list;
  if (do_select(this, columns, NULL, 0)) {
    return true; // 执行失败
  }

  // 清理资源
  cleanup();
  return false;
}
```
核心逻辑
- （1）**初始化执行环境**
  - 分配内存缓存（如排序缓冲区、连接缓冲区）
  - 初始化临时表（处理group by , distinct或子查询物化）
- （2）**数据读取与处理**
  - 通过存储引擎接口（如Innodb的ha_innobase）读取数据
  - 执行表扫描、索引扫描或连接操作。
  - 处理过滤条件（WHERE、HAVING）和聚合（SUM、COUNT）
- （3）**结果返回**
  - 将最终结果写入网络缓冲区或临时表。
- （4）**清理资源**
  - 释放临时表、缓存内存。
  - 关闭存储引擎游标。

---

因为真正的执行内容较多，所以在第三章节具体说明。

# 三、【执行阶段】do_select 真正的执行
MySQL的`join`是采用`nested loop join`（嵌套循环查询）。在`do_select`函数中，通过调用`sub_select`函数来具体实现`join`功能。
```cpp
// sql/sql_executor.cc
static int
do_select(JOIN *join)
{
  // 1. 定位第一个非常量表的起始位置
  QEP_TAB *qep_tab= join->qep_tab + join->const_tables;
  // 2. 断言确保存在需要处理的表
  assert(join->primary_tables);
  // 3. 第一次调用：执行嵌套循环连接
  error= join->first_select(join,qep_tab,0);
  // 4. 第二次调用：清理资源
  if (error >= NESTED_LOOP_OK)
    error= join->first_select(join,qep_tab,1);
}      
```
首先来解析一下`first_select`，源码如下：
```cpp
// sql/sql_optimizer.h
class JOIN :public Sql_alloc
{
  JOIN(const JOIN &rhs);                        /**< not implemented */
  JOIN& operator=(const JOIN &rhs);             /**< not implemented */

public:
  JOIN(THD *thd_arg, SELECT_LEX *select)
    : select_lex(select),
    first_select(sub_select), JOIN类的成员变量
    ...
```
- `JOIN`类中的`first_select`是一个成员变量，它是一个函数指针（通常是 `Next_select_func` 类型），用于指向执行查询的函数。
- `sub_select`是一个函数，通常被赋值给`first_select`，作为嵌套循环连接的执行函数。

所以接下来解析`sub_select`函数，其是嵌套循环连接（`Nested Loop JOIN`）的核心逻辑，简化后代码如下：
```cpp
// sql/sql_executor.cc
enum_nested_loop_state
sub_select(JOIN *join, QEP_TAB *const qep_tab,bool end_of_records)
{
  ...
  // 1. 获取当前表的记录读取器
  READ_RECORD *info= &qep_tab->read_record;
  // 2. 获取当前表在执行计划中的索引位置
  const plan_idx qep_tab_idx= qep_tab->idx();
  // 3. 设置返回位置（用于控制嵌套层级）
  join->return_tab= qep_tab_idx;
  // 4. 标记是否为首次读取
  bool in_first_read= true;
  // 5. 主循环：处理当前表的所有记录
  while (rc == NESTED_LOOP_OK && join->return_tab >= qep_tab_idx)
  {
    int error;
    // 6. 首次读取使用特殊函数（初始化扫描）
    if (in_first_read)
    {
      in_first_read= false;
      error= (*qep_tab->read_first_record)(qep_tab);
    }
    else
    // 7. 后续读取使用标准函数
      error= info->read_record(info);
    // 8. 评估当前记录（连接条件检查：即时评估）
    rc= evaluate_join_record(join, qep_tab);
  }
}
```
关键函数说明：
- `read_first_record`函数指针：
  - 根据访问方法动态绑定
    - **全表扫描**：`rr_sequential()`
    - **索引扫描**：`rr_index_first()`
    - **范围扫描**：`rr_quick()`
  - 负责初始化扫描状态并获取第一条记录
- `read_record`：获取下一条记录
  - **全表扫描**：`rr_sequential_next()`
  - **索引扫描**：`rr_index_next()`
  - 可能返回HA_ERR_END_OF_FILE表示结束
- `evaluate_join_record()`
  - 检查连接条件是否满足
  - 如果满足
    - 递归调用下一层表的`next_select`函数
    - 处理内层表的连接
  - 返回状态控制循环继续或中断


## 3.1、read_first_record
`read_first_record`这个函数指针通常指向一个用于读取表中第一条记录的函数。具体的实现取决于表达访问方法（例如：全表扫描、索引扫描等）。在MySQL中，`read_first_record`是在优化阶段根据执行计划设置的。

几种常见的实现：
1. 全表扫描-- `rr_sequential()`
2. 索引扫描-- `rr_index_first()`


### 3.1.1、read_first_record设置时机
在优化阶段，当确定表的访问方式后，会设置`read_first_record`函数指针。例如在`make_join_queryplan`函数中，优化器根据选择的访问方式设置该指针。

### 3.1.2、全表扫描-- rr_sequential()
```cpp
// sql/records.cc
int rr_sequential(READ_RECORD *info)
{
  int tmp;
  // 读取第一条数据
  while ((tmp=info->table->file->ha_rnd_next(info->record)))
  {
    if (info->thd->killed || (tmp != HA_ERR_RECORD_DELETED))
    {
      tmp= rr_handle_error(info, tmp);
      break;
    }
  }
  return tmp;
}
```
- 调用 `ha_rnd_next` 获取第一条记录


### 3.1.3、索引扫描--rr_index_first()
```cpp
// sql/record.cc
static int rr_index_first(READ_RECORD *info)
{
  int tmp= info->table->file->ha_index_first(info->record);
  info->read_record= rr_index;
  if (tmp)
    tmp= rr_handle_error(info, tmp);
  return tmp;
}
```
`ha_index_first`的实现是索引扫描的入口函数，它负责定位并读取索引中的第一条记录。
```cpp
// sql/handler.cc
int handler::ha_index_first(uchar * buf)
{
  int result;
  DBUG_ENTER("handler::ha_index_first");
  //1. 前置条件断言
  assert(table_share->tmp_table != NO_TMP_TABLE ||
         m_lock_type != F_UNLCK); // 确保已加锁（临时表除外）
  assert(inited == INDEX); // 确保索引已初始化
  assert(!pushed_idx_cond || buf == table->record[0]);// 确保条件推送缓冲区正确

  // 生成列处理标记
  m_update_generated_read_fields= table->has_gcol(); // 标记是否需要更新生成列
  // 核心操作：带性能监控的索引扫描
  MYSQL_TABLE_IO_WAIT(PSI_TABLE_FETCH_ROW, active_index, result,
    { result= index_first(buf); })// 调用引擎具体实现
  if (!result && m_update_generated_read_fields)
  {
    // 生成列值更新
    result= update_generated_read_fields(buf, table, active_index);
    m_update_generated_read_fields= false;// 重置标记
  }
  DBUG_RETURN(result);// 返回结果
}

```
进入Innodb的具体实现：`index_first`
```cpp
// storage/innobase/handler/ha_innodb.cc
int
ha_innobase::index_first(
/*=====================*/
	uchar*	buf)	/*!< in/out: buffer for the row */
{
	DBUG_ENTER("index_first");

	ha_statistic_increment(&SSV::ha_read_first_count);
    // 设置搜索模式
	int	error = index_read(buf, NULL, 0, HA_READ_AFTER_KEY);

	/* MySQL does not seem to allow this to return HA_ERR_KEY_NOT_FOUND */
	if (error == HA_ERR_KEY_NOT_FOUND) {
		error = HA_ERR_END_OF_FILE;
	}

	DBUG_RETURN(error);
}

```
`index_read`是Innodb的索引扫描的核心实现。
```cpp
// storage/innobase/handler/ha_innodb.cc
int
ha_innobase::index_read()
{
  if (mode != PAGE_CUR_UNSUPP) {
    // 进入 InnoDB 并发控制
    innobase_srv_conc_enter_innodb(m_prebuilt);
    // 检查是否为普通表（非内建表）
    if (!dict_table_is_intrinsic(m_prebuilt->table)) {
        // 检查事务是否被中止
        if (TrxInInnoDB::is_aborted(m_prebuilt->trx)) {
            // 回滚事务并返回错误码
            innobase_rollback(ht, m_user_thd, false);
            DBUG_RETURN(convert_error_code_to_mysql(...));
        }
        // 标记是否为 INSERT ... SELECT 语句
        m_prebuilt->ins_sel_stmt = thd_is_ins_sel_stmt(m_user_thd);
        // 调用 MVCC 行搜索
        ret = row_search_mvcc(...);
    } else {
        // 内建表处理（无需 MVCC）
        ret = row_search_no_mvcc(...);
    }

    // 退出 InnoDB 并发控制
    innobase_srv_conc_exit_innodb(m_prebuilt);
}
}
```
这里可以看到`row_search_mvcc()`，基于MVCC保证事务隔离性。
```cpp
// storage/innobase/rwo/rowOsel.cc
dberr_t
row_search_mvcc(
	byte*		buf,
	page_cur_mode_t	mode,
	row_prebuilt_t*	prebuilt,
	ulint		match_mode,
	ulint		direction)
{



```
- `row_search_mvcc()`函数方法贼长，接近`2500`行代码。



## 3.2、read_record



## 3.3、evaluate_join_record
连接条件检查：即时评估
```cpp
rc = evaluate_join_record(join, qep_tab); // 评估连接条件 
```
- **条件检查**：
  - 检查`where`子句条件
  - 验证表之间的连接条件（如 `t1.id = t2.foreign_id`）
- **字段转换**：将存储引擎格式转换为MySQL内部格式
- **短路优化**：条件不满足时立即跳过内层表扫描。

```cpp
static enum_nested_loop_state
evaluate_join_record(JOIN *join, QEP_TAB *const qep_tab)
{
  Item *condition= qep_tab->condition();
  const plan_idx qep_tab_idx= qep_tab->idx();
  bool found= TRUE;
  
  if (condition)
  {
    found= MY_TEST(condition->val_int());
  }
  
  if (found)
  { 
    plan_idx return_tab= join->return_tab;
    if (found)
    {
      qep_tab->found_match= true;
      rc= (*qep_tab->next_select)(join, qep_tab+1, 0);
      if (qep_tab->do_firstmatch() &&
               QEP_AT(qep_tab, match_tab).found_match)
      {
        set_if_smaller(return_tab, qep_tab->firstmatch_return);
      }
      set_if_smaller(join->return_tab, return_tab);
    }
  }
}
```
在`sub_select`函数中，可以看到这样一行代码：`(*join_tab->read_first_record)(join_tab)`这个就是读取表A的第一行结果，可以看`join_tab`里面的信息有表A的名字。接下来就是很关键的一个函数：`evaluate_join_record`，这个函数主要做2件事：
- 将当前已经拿到的信息进行`where`条件计算，判断是否需要继续往下走；
- 递归`JOIN`；





实际执行示例，假设三表连接：A、B、C
```text
// 处理表A
sub_select(A) {
  while (读取A表记录) {
    evaluate(A记录) {
      // 递归调用表B的处理
      sub_select(B) {
        while (读取B表记录) {
          evaluate(B记录) {
            // 递归调用表C的处理
            sub_select(C) {
              while (读取C表记录) {
                evaluate(C记录) {
                  // 最内层：输出结果
                  end_send();
                }
              }
            }
          }
        }
      }
    }
  }
}
```
MySQL在执行嵌套循环连接时，核心流程就是逐行从存储引擎读取数据，然后进行连接条件检查。
1. 读取数据：逐行读取
```cpp
error = info->read_record(info);// 读取下一条记录 
```
- **存储引擎接口**：通过`handler`类的方法（如`ha_rnd_next`或`ha_index_next`）从存储引擎获取单行记数据。
- **物流读取**：每次调用可能导致磁盘I/O
- **记录格式**：读取的存储引擎的原始格式（如Innodb的紧凑格式）



## 3.1、READ_RECORD
`READ_RECORD`是一个核心数据结构，用于管理从存储引擎中读取记录的逻辑。是执行器与存储引擎交互的关键桥梁。

**结构**：
```cpp
struct READ_RECORD {
  TABLE *table;                  // 关联的表对象
  uint index;                    // 使用的索引号
  uchar *key;                    // 索引键值
  uint key_length;               // 索引键长度
  Setup_func setup_func;         // 初始化读取的函数（如 read_first_record）
  Read_func read_next_func;      // 迭代读取的函数（如 read_next）
  uchar *record;                 // 缓存读取的记录
  int status;                    // 读取状态（如 RR_OK、RR_END）
  // ... 其他成员
};
```

**关键函数**：

| 函数                  | 作用                                             |
|---------------------|------------------------------------------------|
| read_first_record() | 初始化读取操作，调用存储引擎的ha_index_init()或ha_rnd_init()   |
| read_next_record()  | 读取下一条记录，调用存储引擎的ha_index_next() 或 ha_rnd_next() |
| ha_index_init()     | 存储引擎接口，初始化索引读取                                 |
| ha_index_read()     | 存储引擎接口，读取索引匹配的第一条记录                            |
| ha_index_next()     | 存储引擎接口，读取下一条索引匹配的记录                            |



# 四、小结
数据读取的基本流程：
1. **初始化读取**：通过 `read_first_record` 初始化读取操作（如打开索引或全表扫描）
2. **逐条读取**：通过 `read_next` 逐行读取数据（或批量读取）。
3. **条件判断**：对每条记录执行过滤条件（如`where`子句）。
4. **循环处理**：直到读取完所有符合条件的记录。

```cpp
void sub_select(...) {
  READ_RECORD *info = &qep_tab->read_record;
  while (rc == NESTED_LOOP_OK) {
    if (in_first_read) {
      error = read_first_record(qep_tab); // 初始化读取
      in_first_read = false;
    } else {
      error = info->read_record(info);    // 逐条读取
    }
    rc = evaluate_join_record(...);       // 条件判断与结果处理
  }
}
```



参考文章：
https://juejin.cn/post/6844904114489393160
https://github.com/astrophor/astrophor.github.io/blob/master/_posts/2013-6-1-MySQL_select%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.md
