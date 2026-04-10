---
name: mysql-optimizer
description: 提供 MySQL 查询优化、SQL 审查、索引设计及执行计划分析的专业知识。包含慢查询定位、EXPLAIN 解读、索引优化策略、统计信息管理及反模式识别。
---

# MySQL Optimizer 技能

本技能帮助您系统化地分析和优化 MySQL 查询性能，从慢查询定位到执行计划解读，从索引设计到 SQL 重写，提供可操作的优化建议和最佳实践。

## 何时使用

- 用户报告查询响应缓慢，需要定位和解决
- 审查新上线的 SQL 语句，防止性能问题
- 设计或调整索引以加速特定查询
- 分析 `EXPLAIN` 输出并理解其含义
- 处理大表分页、排序、分组等常见性能瓶颈
- 优化 JOIN、子查询、聚合查询
- 排查锁等待、死锁或临时表磁盘溢出问题
- 调整 MySQL 配置参数以提升查询性能

## 核心概念速查

| 概念 | 说明 | 关键关注点 |
|------|------|-----------|
| 执行计划 | MySQL 如何执行查询 | `EXPLAIN` 输出中的 `type`, `key`, `rows`, `Extra` |
| 索引类型 | B-Tree（主流）、Hash、全文、空间 | 最左前缀、索引选择性、覆盖索引 |
| 访问类型 | 从全表扫描到索引查找 | `ALL` < `index` < `range` < `ref` < `eq_ref` < `const` |
| 临时表 | 排序或分组产生的中间结果 | `Using temporary` 需优化 |
| 文件排序 | 无法使用索引时的排序 | `Using filesort` 需索引优化 |
| 索引条件下推 | 索引过滤减少回表 | MySQL 5.6+ 特性 |
| 查询缓存 | 已废弃（MySQL 8.0） | 忽略 |
| 优化器提示 | 影响执行计划选择 | `FORCE INDEX`, `IGNORE INDEX`, `STRAIGHT_JOIN` |

## 慢查询定位与分析流程

### 1. 开启并配置慢查询日志

```sql
-- 查看当前设置
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';
SHOW VARIABLES LIKE 'log_queries_not_using_indexes';

-- 动态开启（重启失效）
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.5;           -- 记录超过 0.5 秒的查询
SET GLOBAL log_queries_not_using_indexes = ON;
SET GLOBAL min_examined_row_limit = 100;     -- 忽略扫描行数少的查询
```

### 2. 分析慢查询日志

```bash
# 使用 mysqldumpslow 工具汇总
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# 使用 pt-query-digest（Percona Toolkit）更详细分析
pt-query-digest /var/log/mysql/slow.log --limit 20
```

### 3. 实时监控当前运行的查询

```sql
-- 查看当前所有连接和正在执行的查询
SHOW PROCESSLIST;
-- 或者更详细的信息（MySQL 5.7+）
SELECT * FROM performance_schema.threads WHERE PROCESSLIST_STATE IS NOT NULL\G

-- 查看当前执行时间最长的查询
SELECT id, user, host, db, command, time, state, info
FROM information_schema.PROCESSLIST
WHERE command != 'Sleep'
ORDER BY time DESC;
```

## EXPLAIN 深度解读

### 输出列详解

| 列名 | 关键值 | 说明 |
|------|--------|------|
| `id` | 数字 | 查询中 SELECT 的序号，越大越先执行 |
| `select_type` | SIMPLE, PRIMARY, SUBQUERY, DERIVED, UNION | 查询类型 |
| `table` | 表名或别名 | 当前行访问的表 |
| `type` | **system > const > eq_ref > ref > range > index > ALL** | 访问类型，从好到差 |
| `possible_keys` | 可用索引 | 优化器考虑使用的索引 |
| `key` | 实际使用的索引 | NULL 表示未使用 |
| `key_len` | 字节数 | 索引使用的长度（前缀索引或复合索引部分） |
| `ref` | 列或常量 | 与索引列比较的对象 |
| `rows` | 估算行数 | 需要扫描的行数，越小越好 |
| `filtered` | 百分比 | 满足 WHERE 条件的行占比（越大越好） |
| `Extra` | Using index, Using where, Using temporary, Using filesort | 额外信息，红色警报项 |

### 典型危险信号与对策

| Extra 信息 | 含义 | 优化方向 |
|------------|------|----------|
| `Using filesort` | 需要额外排序 | 为 `ORDER BY` 列建立索引，或利用现有索引顺序 |
| `Using temporary` | 使用临时表（内存或磁盘） | 优化 `GROUP BY` 或 `DISTINCT`，使用索引 |
| `Using where` + 全表扫描 | 无法使用索引过滤 | 添加合适的索引 |
| `Using join buffer` | 连接无法使用索引 | 为被驱动表的连接列加索引 |
| `Impossible WHERE` | WHERE 条件恒假 | 检查逻辑错误 |
| `No matching row in const table` | 连接空值 | 数据问题或查询逻辑 |

### EXPLAIN 扩展工具

```sql
-- 显示更详细的分区、过滤等信息
EXPLAIN FORMAT=JSON SELECT ...;

-- 显示实际执行过程中使用的索引和行数（MySQL 8.0.18+）
EXPLAIN ANALYZE SELECT ...;

-- 查看优化器追踪（详细决策过程）
SET optimizer_trace="enabled=on";
SELECT ...;
SELECT * FROM information_schema.OPTIMIZER_TRACE\G
```

## 索引优化核心策略

### 索引设计黄金法则

1. **最左前缀原则**：复合索引 `(a, b, c)` 支持 `a`, `a,b`, `a,b,c` 的查找，不支持 `b` 或 `c` 单独。
2. **高选择性优先**：索引列的区分度越高越好（`COUNT(DISTINCT col)/COUNT(*)` 接近 1）。
3. **覆盖索引**：查询所需列全部在索引中，避免回表（`Extra: Using index`）。
4. **索引下推**：将 WHERE 条件下推到索引层过滤（MySQL 5.6+ 自动）。
5. **避免冗余索引**：`(a,b)` 已覆盖 `(a)`，后者冗余。
6. **考虑排序需求**：`ORDER BY` 的列方向应与索引顺序一致（ASC/DESC）。

### 常见索引失效场景

| 查询写法 | 失效原因 | 修正示例 |
|----------|----------|----------|
| `WHERE col + 1 = 10` | 对索引列使用函数或运算 | `WHERE col = 9` |
| `WHERE DATE(col) = '2024-01-01'` | 函数包裹索引列 | `WHERE col BETWEEN '2024-01-01' AND '2024-01-02'` |
| `WHERE col LIKE '%abc'` | 前导通配符 | 避免以 `%` 开头 |
| `WHERE col1 = 1 OR col2 = 2` | OR 两边不全是同一索引列 | 使用 UNION 或分别索引 |
| `WHERE col IS NULL` | 部分索引不支持 NULL（取决于存储引擎） | 尽量避免 NULL，使用默认值 |
| `WHERE col != 5` | 不等条件一般无法使用索引 | 重新设计查询 |
| 隐式类型转换 | `WHERE int_col = '123'` 可；`WHERE char_col = 123` 失效 | 保持类型一致 |

### 索引创建与管理

```sql
-- 查看表现有索引
SHOW INDEX FROM table_name;

-- 创建索引
CREATE INDEX idx_name ON table_name (col1, col2);
CREATE UNIQUE INDEX idx_unique ON table_name (col);
CREATE FULLTEXT INDEX idx_fulltext ON table_name (content);

-- 删除冗余索引
DROP INDEX idx_redundant ON table_name;

-- 强制使用索引（不推荐，用于紧急情况）
SELECT * FROM table_name FORCE INDEX (idx_name) WHERE ...;
```

## SQL 重写优化模板

### 分页优化（大 offset）

```sql
-- 低效写法：OFFSET 会扫描所有前 N 行
SELECT * FROM orders ORDER BY id LIMIT 100000, 20;

-- 优化1：记住上一页最后一条记录的 ID（游标分页）
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 20;

-- 优化2：延迟关联（先查 ID，再回表）
SELECT o.* 
FROM orders o
JOIN (SELECT id FROM orders ORDER BY id LIMIT 100000, 20) tmp ON o.id = tmp.id;
```

### 避免 SELECT *

```sql
-- 坏习惯
SELECT * FROM users WHERE email = 'user@example.com';

-- 优化：只取需要的列
SELECT id, name, email FROM users WHERE email = 'user@example.com';
-- 可建立覆盖索引 (email, id, name)
```

### 子查询优化为 JOIN

```sql
-- 低效：相关子查询
SELECT name 
FROM products 
WHERE category_id IN (SELECT id FROM categories WHERE active = 1);

-- 优化：JOIN 形式
SELECT p.name 
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE c.active = 1;
-- 确保 c.id 和 p.category_id 有索引
```

### 聚合查询优化

```sql
-- 低效：先 JOIN 后 GROUP BY，大量临时表
SELECT u.id, COUNT(o.id) 
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;

-- 优化：使用子查询预聚合（减少 JOIN 行数）
SELECT u.id, COALESCE(oc.cnt, 0) 
FROM users u
LEFT JOIN (
    SELECT user_id, COUNT(*) cnt FROM orders GROUP BY user_id
) oc ON u.id = oc.user_id;
```

### 批量操作代替循环

```sql
-- 坏：应用层循环逐条更新
for id in ids:
    UPDATE products SET stock = stock - 1 WHERE id = id;

-- 好：单条 UPDATE + IN（或使用临时表）
UPDATE products SET stock = stock - 1 WHERE id IN (1,2,3);
```

## 统计信息与优化器

### 更新统计信息

```sql
-- 重新分析表，更新索引分布统计
ANALYZE TABLE table_name;

-- 查看表统计信息
SHOW TABLE STATUS LIKE 'table_name'\G
SELECT * FROM mysql.innodb_table_stats WHERE table_name = 'table_name';
```

### 影响优化器选择的因素

- 表统计信息（过时会导致错误执行计划）
- 索引选择性
- `eq_range_index_dive_limit` 参数
- 系统变量 `optimizer_switch`（如 `index_merge`, `mrr` 等）
- 查询中的提示（`STRAIGHT_JOIN`, `SQL_NO_CACHE` 等）

## 性能调优配置参数

| 参数 | 说明 | 推荐起始值 |
|------|------|-----------|
| `innodb_buffer_pool_size` | InnoDB 缓存数据和索引 | 系统内存的 70-80% |
| `innodb_log_file_size` | Redo 日志大小 | 1-2 GB |
| `tmp_table_size` / `max_heap_table_size` | 内存临时表上限 | 64M - 256M |
| `sort_buffer_size` | 排序缓冲区 | 2M - 4M（避免过大） |
| `join_buffer_size` | 连接缓冲区 | 2M - 4M |
| `query_cache_type` | 查询缓存（8.0 已移除） | OFF |
| `innodb_flush_log_at_trx_commit` | 持久性 vs 性能 | 1（强一致）或 2（高性能） |

## 实战优化案例

### 案例1：订单列表慢查询

**原查询**：
```sql
SELECT * FROM orders 
WHERE user_id = 123 
ORDER BY created_at DESC 
LIMIT 20;
```

**问题**：`(user_id, created_at)` 无复合索引，先通过 user_id 索引找到所有订单，再排序（Using filesort）。

**优化**：创建复合索引 `idx_user_created (user_id, created_at)`，查询变为 `Using index condition`，且 ORDER BY 使用索引顺序，消除 filesort。

### 案例2：报表统计 GROUP BY 慢

**原查询**：
```sql
SELECT DATE(created_at) AS day, COUNT(*), SUM(amount)
FROM payments
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY DATE(created_at);
```

**问题**：`DATE(created_at)` 函数导致无法使用 `created_at` 索引。

**优化**：
```sql
SELECT DATE(created_at) AS day, COUNT(*), SUM(amount)
FROM payments
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'
GROUP BY day;   -- MySQL 允许使用别名，但实际仍按表达式分组
```
更好的方案是增加生成列 `created_date DATE AS (DATE(created_at)) STORED` 并加索引。

### 案例3：分页深度翻页

**原查询**：
```sql
SELECT * FROM logs ORDER BY id LIMIT 500000, 20;
```
扫描 50 万行后取 20 行。

**优化**：使用延迟关联或记住上一页最大 ID。

## 监控与诊断工具

```sql
-- 查看当前查询的 profile（已废弃，改用 Performance Schema）
SET profiling = 1;
SELECT ...;
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;

-- 使用 Performance Schema 分析等待事件
SELECT * FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;

-- 查看索引使用情况
SELECT * FROM sys.schema_index_statistics WHERE table_schema = 'yourdb';

-- 查看未使用的索引
SELECT * FROM sys.schema_unused_indexes;
```

## 反模式识别清单

| 反模式 | 说明 | 正确做法 |
|--------|------|----------|
| 外键使用字符串 | 性能差 | 使用整数代理键 |
| 过度索引 | 写入性能下降 | 定期审查并删除冗余索引 |
| 未使用覆盖索引 | 频繁回表 | 调整索引包含所需列 |
| 隐式类型转换 | 索引失效 | 统一字段和应用类型 |
| SELECT * 大量列 | 浪费 I/O | 只取必要列 |
| LIMIT 巨大 offset | 低效 | 游标或延迟关联 |
| OR 跨不同列 | 无法使用索引 | 分别查询 UNION |
| 依赖查询缓存 | 已废弃 | 使用应用层缓存或代理 |

## 快速决策树

```
查询慢？
├─ 慢查询日志中有记录？
│  ├─ 是 → 获取具体 SQL → EXPLAIN 分析
│  │  ├─ type = ALL → 加索引
│  │  ├─ Extra 含 Using temporary/filesort → 优化索引或 SQL
│  │  ├─ rows 很大 → 评估索引选择性或改写查询
│  │  └─ 其他 → 考虑配置参数或硬件
│  └─ 否 → 可能是锁等待或资源瓶颈
│     ├─ SHOW PROCESSLIST 查看状态
│     └─ 检查 InnoDB 锁信息
└─ 索引已存在但仍慢？
   ├─ 统计信息过时 → ANALYZE TABLE
   ├─ 优化器选错索引 → 使用 FORCE INDEX 或调整参数
   └─ 数据分布问题 → 考虑分区或架构调整
```

## 扩展资源

| 主题 | 链接 |
|------|------|
| MySQL 官方优化手册 | [https://dev.mysql.com/doc/refman/8.0/en/optimization.html](https://dev.mysql.com/doc/refman/8.0/en/optimization.html) |
| EXPLAIN 输出格式 | [https://dev.mysql.com/doc/refman/8.0/en/explain-output.html](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html) |
| 索引最佳实践 | [https://use-the-index-luke.com/](https://use-the-index-luke.com/) |
| Percona Toolkit | [https://www.percona.com/software/database-tools/percona-toolkit](https://www.percona.com/software/database-tools/percona-toolkit) |
| 高性能 MySQL（书籍） | 第 5 版，O'Reilly |