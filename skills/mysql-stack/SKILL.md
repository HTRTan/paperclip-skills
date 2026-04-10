---
name: mysql-stack
description: 提供 MySQL 数据库架构验证、迁移决策审查、性能影响评估及安全合规检查。包含迁移前检查清单、SQL 规范、备份策略、数据一致性验证及常见反模式识别。
---

# MySQL Stack 技能

本技能帮助您在数据库设计、迁移或重大变更时，系统地验证架构合理性、评估迁移风险、确保数据完整性，并遵循最佳实践。适用于开发阶段的设计评审、上线前的迁移审核，以及日常运维中的决策支持。

## 何时使用

- 设计新表或修改现有表结构时，需要验证范式、索引、数据类型
- 计划进行数据库迁移（schema 变更、数据迁移、版本升级）
- 需要评估迁移对性能、锁、复制延迟的影响
- 审查 SQL 查询是否高效、安全（防止 SQL 注入、避免全表扫描）
- 制定备份与恢复策略，验证数据一致性
- 检查数据库配置、权限、加密等安全合规项
- 排查迁移过程中可能的数据丢失或类型不匹配问题

## 核心概念速查

| 概念 | 说明 | 关键检查点 |
|------|------|-----------|
| 范式 | 减少冗余与异常 | 3NF 通常足够，适当反范式用于性能 |
| 索引 | 加速查询，减慢写入 | 覆盖索引、最左前缀、索引选择性 |
| 外键约束 | 保证引用完整性 | 注意锁和级联操作的影响 |
| 事务隔离级别 | 控制并发行为 | RR（默认）或 RC，注意幻读、间隙锁 |
| 迁移策略 | 变更不中断服务 | pt-online-schema-change, gh-ost |
| 备份类型 | 逻辑备份（mysqldump） vs 物理备份（XtraBackup） | RPO/RTO 要求 |
| 复制延迟 | 主从同步滞后 | 监控 Seconds_Behind_Master |
| 数据一致性 | 迁移前后数据相同 | 校验和、行数对比、抽样验证 |

## 迁移决策检查清单

在开始任何数据库迁移前，请逐项确认以下内容：

### 1. 变更类型与影响评估

- [ ] **是否必须立即执行？** 能否安排在业务低峰期？
- [ ] **DDL 类型**：添加列（允许 NULL，无默认值最快）| 删除列（需要重写表）| 修改数据类型（可能重写）| 添加索引（在线？MySQL 5.6+ 支持）| 重命名表（需更新应用代码）
- [ ] **表大小**：< 1GB 可在线 DDL；> 10GB 建议使用 pt-osc/gh-ost
- [ ] **锁评估**：DDL 是否会阻塞读写？使用 `ALTER TABLE ... ALGORITHM=INPLACE, LOCK=NONE` 检查可行性
- [ ] **复制影响**：在主库执行后，从库会如何重放？大表 DDL 可能导致复制延迟

### 2. 数据验证准备

- [ ] **备份就绪**：已执行最新备份（物理或逻辑），并验证可恢复
- [ ] **回滚计划**：如何撤销变更？保留旧版本表或脚本
- [ ] **数据一致性检查**：迁移前后运行校验查询（行数、SUM/CRC32 关键列）
- [ ] **测试环境验证**：在类生产环境（数据量相似）先执行一次完整流程
- [ ] **通知与审批**：变更窗口、影响范围、回滚负责人

### 3. 应用程序兼容性

- [ ] 应用代码是否引用被改动的列/表名？
- [ ] 数据类型变更是否导致隐式转换或精度丢失（如 INT 改为 BIGINT，DATETIME 精度）？
- [ ] 新增 NOT NULL 列是否有默认值？旧行如何填充？
- [ ] 删除索引后，查询性能是否下降？
- [ ] 是否有数据库连接池或 ORM 缓存需要刷新？

## MySQL 迁移模式与工具对比

| 工具/方法 | 适用场景 | 优点 | 缺点 |
|-----------|----------|------|------|
| 原生 ONLINE DDL | MySQL 5.6+，添加列/索引，修改默认值 | 无需额外工具，语法简单 | 大表仍可能短暂锁表，不支持删除列等重写操作 |
| pt-online-schema-change | 大表（>10GB），避免阻塞写入 | 触发器方式，几乎无锁，可暂停 | 需要额外权限，触发器影响性能 |
| gh-ost | 大表，要求无触发器，更安全 | 无触发器，基于 binlog 回放，可测试 | 配置稍复杂，需要 binlog 格式 ROW |
| 手动影子表 | 完全控制迁移过程 | 灵活，适用于复杂转换 | 代码量大，容易出错 |

### 示例：使用 pt-online-schema-change 安全添加索引

```bash
pt-online-schema-change \
  --alter "ADD INDEX idx_user_status (user_id, status)" \
  D=yourdb,t=orders \
  --host=127.0.0.1 \
  --user=admin \
  --password=*** \
  --chunk-size=1000 \
  --critical-load="Threads_running=200" \
  --max-load="Threads_running=50" \
  --execute
```

## SQL 规范与审查清单

### 查询性能检查项

- [ ] **是否使用了索引？** 使用 `EXPLAIN` 分析，关注 `type`（ALL 代表全表扫描）、`key`、`rows`、`Extra`（Using filesort / Using temporary 应避免）
- [ ] **SELECT \*** 是否必要？仅选取所需列
- [ ] **WHERE 条件中是否对索引列使用函数或运算？** 例如 `WHERE DATE(created_at) = '2024-01-01'` 无法使用索引，应改为 `created_at BETWEEN ...`
- [ ] **隐式类型转换**：例如 `WHERE id = '123'`（id 为 INT）会导致全表扫描
- [ ] **分页优化**：大 offset 使用延迟关联或游标方式
- [ ] **避免在循环中执行 SQL**：改为批量操作或 JOIN

### 安全性检查

- [ ] **防止 SQL 注入**：使用参数化查询（Prepared Statements），禁止拼接用户输入
- [ ] **权限最小化**：应用账号仅授予必要权限（SELECT, INSERT, UPDATE, DELETE），无 DDL 或 FILE 权限
- [ ] **敏感数据加密**：密码、身份证等字段使用 AES_ENCRYPT 或应用层加密
- [ ] **日志脱敏**：慢查询日志、错误日志不输出敏感信息

### 数据完整性检查

- [ ] **外键约束**：级联更新/删除是否符合业务逻辑？是否导致死锁？
- [ ] **唯一约束**：防止重复数据，需考虑复合唯一索引
- [ ] **NOT NULL 与默认值**：应用插入时是否提供所有 NOT NULL 列？
- [ ] **检查约束**（MySQL 8.0.16+）：使用 `CHECK` 保证字段值范围

## 迁移前后验证脚本模板

### 迁移前基线检查

```sql
-- 1. 记录表行数、数据大小
SELECT 
    table_name,
    table_rows,
    ROUND(data_length / 1024 / 1024, 2) AS data_mb,
    ROUND(index_length / 1024 / 1024, 2) AS index_mb
FROM information_schema.tables
WHERE table_schema = 'yourdb' AND table_name = 'target_table';

-- 2. 关键列的数据分布（用于迁移后对比）
SELECT 
    COUNT(*) AS total,
    SUM(IF(status = 'active', 1, 0)) AS active_count,
    SUM(IF(status = 'inactive', 1, 0)) AS inactive_count,
    MIN(created_at) AS min_created,
    MAX(created_at) AS max_created
FROM target_table;

-- 3. 外键依赖检查
SELECT 
    CONSTRAINT_NAME,
    TABLE_NAME,
    COLUMN_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE REFERENCED_TABLE_NAME = 'target_table' AND TABLE_SCHEMA = 'yourdb';

-- 4. 存在重复数据（若计划添加唯一索引）
SELECT user_id, order_id, COUNT(*)
FROM target_table
GROUP BY user_id, order_id
HAVING COUNT(*) > 1;
```

### 迁移后验证

```sql
-- 1. 对比行数（应与迁移前一致）
SELECT COUNT(*) FROM target_table;

-- 2. 关键列校验和（使用 CRC32 或 MD5 聚合）
SELECT 
    COUNT(*) AS total,
    SUM(IF(status = 'active', 1, 0)) AS active_count,
    SUM(IF(status = 'inactive', 1, 0)) AS inactive_count,
    MIN(created_at) AS min_created,
    MAX(created_at) AS max_created
FROM target_table;

-- 3. 检查新增列默认值是否正确填充（对于 NOT NULL DEFAULT ...）
SELECT COUNT(*) FROM target_table WHERE new_column IS NULL;  -- 应为 0

-- 4. 测试关键查询性能（对比 EXPLAIN 输出）
EXPLAIN SELECT * FROM target_table WHERE user_id = 123 AND status = 'active';
```

### 数据一致性校验（全表对比，适合小表）

```bash
#!/bin/bash
# 使用 mysqldump 对比迁移前后两个数据库实例的表数据

mysqldump -h source_host -u user -p'pass' yourdb target_table --no-create-info --skip-extended-insert > /tmp/old_data.sql
mysqldump -h target_host -u user -p'pass' yourdb target_table --no-create-info --skip-extended-insert > /tmp/new_data.sql

diff /tmp/old_data.sql /tmp/new_data.sql
if [ $? -eq 0 ]; then
    echo "数据完全一致"
else
    echo "数据存在差异，请手动检查"
fi
```

## 备份与回滚策略

### 快速回滚方案

1. **闪回查询（需要 binlog 格式为 ROW）**：使用 `mysqlbinlog` 提取反向 SQL
2. **重命名旧表保留**：
   ```sql
   -- 迁移前
   RENAME TABLE orders TO orders_old, orders_new TO orders;
   -- 如果迁移失败，反向重命名
   ```
3. **逻辑备份恢复**：
   ```bash
   mysql -h target -u user -p yourdb < backup.sql
   ```
4. **物理备份恢复**（最快）：使用 XtraBackup 还原整个数据目录

### 备份验证脚本

```bash
#!/bin/bash
# 每日验证备份可恢复性

BACKUP_FILE="/backup/mysql/$(date +%Y%m%d)_yourdb.sql.gz"
TEST_DB="test_restore_$(date +%Y%m%d%H%M%S)"

gunzip -c $BACKUP_FILE | mysql -e "CREATE DATABASE $TEST_DB; USE $TEST_DB; source /dev/stdin"

# 简单检查是否有表
TABLE_COUNT=$(mysql -N -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='$TEST_DB'")
if [ $TABLE_COUNT -gt 0 ]; then
    echo "备份验证成功，共 $TABLE_COUNT 张表"
    mysql -e "DROP DATABASE $TEST_DB"
else
    echo "备份验证失败"
    exit 1
fi
```

## 常见迁移决策场景与建议

| 场景 | 推荐方案 | 注意事项 |
|------|----------|----------|
| 添加列（允许 NULL） | `ALTER TABLE ADD COLUMN` (ALGORITHM=INPLACE) | 几乎无影响，可在线执行 |
| 添加列（NOT NULL 无默认值） | 先添加允许 NULL，再分批更新值，最后 MODIFY NOT NULL | 避免大表锁 |
| 修改列数据类型（INT→BIGINT） | `ALTER TABLE MODIFY`，需重建表 | 大表使用 pt-osc |
| 删除索引 | `ALTER TABLE DROP INDEX` (INPLACE) | 立即生效，但需确认查询性能 |
| 重命名表 | 原子操作，但需更新所有引用 | 应用需配合停机或新老表双写 |
| 字符集转换（latin1→utf8mb4） | 使用 pt-osc，并注意索引长度限制 | 可能增加存储空间 |
| 合并分区 | 先创建新表，INSERT SELECT，然后重命名 | 停机时间长，建议在线工具 |
| 数据清理（删除过期行） | 分批删除，避免大事务 | 控制每批 1000 行，加 LIMIT |

## 性能基线对比模板

迁移前与迁移后，对比以下指标：

- **慢查询数量**（同一时间段）
- **QPS/TPS**（使用 `SHOW GLOBAL STATUS` 或监控系统）
- **复制延迟**（`SHOW SLAVE STATUS\G` 中的 `Seconds_Behind_Master`）
- **InnoDB 锁等待**（`SHOW ENGINE INNODB STATUS`）
- **表磁盘占用**（`SELECT data_length, index_length ...`）

### 示例：压力测试对比（使用 sysbench）

```bash
# 安装 sysbench，准备数据
sysbench oltp_read_write --mysql-host=localhost --mysql-db=testdb --tables=10 --table-size=100000 prepare

# 迁移前测试
sysbench oltp_read_write --threads=16 --time=60 run > before.txt

# 执行迁移

# 迁移后测试
sysbench oltp_read_write --threads=16 --time=60 run > after.txt

# 对比 before.txt 和 after.txt 中的 transactions per second
```

## 安全与合规检查项

- [ ] **数据传输加密**：客户端连接使用 SSL/TLS（`require_secure_transport=ON`）
- [ ] **静态数据加密**：启用 TDE 或文件系统加密
- [ ] **审计日志**：启用 audit_log 插件记录 DDL 和敏感操作
- [ ] **密码策略**：validate_password 插件强制复杂度
- [ ] **定期权限回收**：移除不再使用的用户或过度授权
- [ ] **备份加密**：使用 `openssl enc` 加密备份文件

## 故障排查速查表

| 问题 | 诊断命令 | 解决方案 |
|------|----------|----------|
| 迁移期间锁等待 | `SHOW PROCESSLIST;` 查看 State | 使用在线工具或低峰期，KILL 阻塞查询 |
| 复制延迟飙升 | `SHOW SLAVE STATUS\G` 看 `Seconds_Behind_Master` | 调整 slave_parallel_workers，或使用 pt-slave-restart 跳过临时错误 |
| 磁盘空间不足 | `df -h`，检查 binlog 和 ibdata1 | 清理旧 binlog（`PURGE BINARY LOGS`），或添加磁盘 |
| 外键约束失败 | `SHOW ENGINE INNODB STATUS` 查看最新外键错误 | 检查数据完整性，暂时禁用外键检查 `SET foreign_key_checks=0`（谨慎） |
| 应用连接失败 | 检查 `max_connections` 和 `threads_connected` | 增加 max_connections，或排查连接泄漏 |
| 数据类型转换错误 | 查看 MySQL 错误日志 | 使用 CAST 或修改应用逻辑，确保数据可转换 |

## 扩展资源

| 主题 | 链接 |
|------|------|
| MySQL 在线 DDL 操作 | [https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html) |
| pt-online-schema-change 文档 | [https://www.percona.com/doc/percona-toolkit/LATEST/pt-online-schema-change.html](https://www.percona.com/doc/percona-toolkit/LATEST/pt-online-schema-change.html) |
| gh-ost 项目 | [https://github.com/github/gh-ost](https://github.com/github/gh-ost) |
| MySQL 备份与恢复 | [https://dev.mysql.com/doc/refman/8.0/en/backup-and-recovery.html](https://dev.mysql.com/doc/refman/8.0/en/backup-and-recovery.html) |
| 性能调优最佳实践 | [https://dev.mysql.com/doc/refman/8.0/en/optimization.html](https://dev.mysql.com/doc/refman/8.0/en/optimization.html) |
