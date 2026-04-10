# DBA Lead Tools

## Paperclip API
Same endpoints as CTO — inbox, checkout, checkin, comment, update status.

## Skills
| Skill | When to use |
|-------|--------------|
| `paperclip` | Coordination, ticket management |
| `mysql-optimizer` | `EXPLAIN` analysis, indexing strategies, slow query log, N+1 elimination, batch operations |
| `mybatis-schema` | Mapper XML review, resultMap, type aliases, dynamic SQL, annotation vs XML trade-offs |
| `data-remediation` | Data quality validation, anomaly detection, bulk fixes with controlled transactions |
| `mysql-stack` | Database conventions (decimal for cents, ISO dates, UUIDs, engine=InnoDB, charset=utf8mb4) |

## CLI Tools

| Tool | Purpose |
|------|---------|
| `mysql -h host -u user -p -e "..."` | Run ad‑hoc queries for analysis |
| `mysql -h host -u user -p database < script.sql` | Execute SQL files |
| `pt-query-digest slow.log` | Analyze slow query log (Percona Toolkit) |
| `mysqldump --single-transaction --routines --triggers` | Create logical backup |
| `mysqlbinlog --start-datetime ... binlog.0000xx` | Point‑in‑time recovery inspection |
| `SHOW MIGRATIONS` (if using Flyway) | Check migration status |
| `flyway migrate` / `liquibase update` (review only) | Generate or review migration scripts |
| `git diff migrations/` | Review migration file changes in PRs |
| `EXPLAIN FORMAT=JSON SELECT ...` | Detailed execution plan |
| `SHOW INDEX FROM table;` | List indexes |

## Tools You Do NOT Use

| Tool | Operated by |
|------|--------------|
| `flyway migrate` / `liquibase update` on production | DevOps Lead executes |
| `mysqlbinlog` restore / `SET GLOBAL` changes | DevOps Lead executes (you advise) |
| Application code (Java/MyBatis mappers) | App Dev Lead |
| Schema changes via Hibernate/JPA `ddl-auto` | App Dev Lead (dev only, forbidden in prod) |

## Useful Queries for Auditing

```sql
-- List all tables with row counts and engine
SELECT TABLE_NAME, TABLE_ROWS, ENGINE, CREATE_TIME 
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY TABLE_ROWS DESC;

-- List all indexes (non-PK)
SELECT TABLE_NAME, INDEX_NAME, COLUMN_NAME, SEQ_IN_INDEX, NON_UNIQUE 
FROM information_schema.STATISTICS 
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX;

-- Check for tables missing created_at / updated_at
SELECT TABLE_NAME 
FROM information_schema.COLUMNS 
WHERE TABLE_SCHEMA = DATABASE()
GROUP BY TABLE_NAME
HAVING SUM(COLUMN_NAME = 'created_at') = 0
   OR SUM(COLUMN_NAME = 'updated_at') = 0;

-- Find tables without primary key (dangerous for replication/binlog)
SELECT TABLE_NAME 
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME NOT IN (
    SELECT TABLE_NAME 
    FROM information_schema.KEY_COLUMN_USAGE 
    WHERE CONSTRAINT_NAME = 'PRIMARY' AND TABLE_SCHEMA = DATABASE()
  );

-- Top 10 largest tables by data length
SELECT TABLE_NAME, ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb 
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = DATABASE()
ORDER BY (DATA_LENGTH + INDEX_LENGTH) DESC 
LIMIT 10;
```

## Memory (PARA)
Use `para-memory-files` skill. Daily notes: `$AGENT_HOME/memory/YYYY-MM-DD.md`.

## Key Files to Track
- `migrations/*.sql` — all migration scripts (Flyway/Liquibase format)
- `src/main/resources/mapper/*.xml` — MyBatis mapper XML files
- `src/main/resources/mybatis-config.xml` — MyBatis configuration
- `application-{env}.properties` — datasource URLs, connection pool settings
- `db/migration/` — versioned SQL scripts

## References
- `$AGENT_HOME/HEARTBEAT.md` — execution checklist
- `$AGENT_HOME/SOUL.md` — persona and voice
