# MySQL 性能调优与故障排查

## 1. 性能监控

```
MySQL 监控指标:

┌─────────────────────────────────────────────────────────────┐
│                    关键监控指标                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  连接指标:                                                  │
│  ├── Threads_connected: 当前连接数                         │
│  ├── Threads_running: 活跃连接数                           │
│  ├── Max_used_connections: 历史最大连接数                   │
│  └── Connections: 总连接次数                               │
│                                                             │
│  查询指标:                                                  │
│  ├── Questions: 查询总数                                   │
│  ├── Com_select: SELECT 次数                               │
│  ├── Com_insert: INSERT 次数                               │
│  ├── Com_update: UPDATE 次数                               │
│  └── Slow_queries: 慢查询次数                              │
│                                                             │
│  InnoDB 指标:                                               │
│  ├── Innodb_buffer_pool_read_requests: 缓冲池读请求         │
│  ├── Innodb_buffer_pool_reads: 磁盘读次数                   │
│  ├── Innodb_row_lock_waits: 行锁等待次数                    │
│  └── Innodb_row_lock_time: 行锁等待时间                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- 查看全局状态
SHOW GLOBAL STATUS;

-- 查看关键指标
SHOW GLOBAL STATUS LIKE 'Threads%';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- 查看变量配置
SHOW GLOBAL VARIABLES LIKE 'max_connections';
SHOW GLOBAL VARIABLES LIKE 'innodb_buffer_pool_size';
```

## 2. 缓冲池优化

```
InnoDB Buffer Pool:

┌─────────────────────────────────────────────────────────────┐
│                    Buffer Pool 架构                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Buffer Pool                            │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │              数据页 (Data Pages)             │   │   │
│  │  │  ┌────┬────┬────┬────┬────┬────┬────┬────┐  │   │   │
│  │  │  │ P0 │ P1 │ P2 │ P3 │ P4 │ P5 │ P6 │ P7 │  │   │   │
│  │  │  └────┴────┴────┴────┴────┴────┴────┴────┘  │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │              索引页 (Index Pages)            │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │              自适应哈希索引 (AHI)            │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  缓冲池命中率 = 1 - (Innodb_buffer_pool_reads /            │
│                       Innodb_buffer_pool_read_requests)     │
│  目标: > 99%                                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- 查看缓冲池状态
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- 计算缓冲池命中率
SELECT 
    (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100 
    AS buffer_pool_hit_rate
FROM (
    SELECT 
        VARIABLE_VALUE AS Innodb_buffer_pool_reads
    FROM performance_schema.global_status 
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
) a, (
    SELECT 
        VARIABLE_VALUE AS Innodb_buffer_pool_read_requests
    FROM performance_schema.global_status 
    WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
) b;

-- 设置缓冲池大小（建议物理内存的 70-80%）
SET GLOBAL innodb_buffer_pool_size = 8589934592;  -- 8GB
```

## 3. 查询缓存

```sql
-- 查询缓存已废弃（MySQL 8.0 移除）
-- MySQL 5.7: query_cache_type = OFF (默认)

-- 替代方案: 使用 Redis 缓存热点查询
```

## 4. 连接池优化

```sql
-- 最大连接数
SET GLOBAL max_connections = 1000;

-- 查看连接使用情况
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';

-- 连接超时
SET GLOBAL wait_timeout = 600;          -- 非交互连接超时 10分钟
SET GLOBAL interactive_timeout = 1800;  -- 交互连接超时 30分钟

-- 连接数建议:
-- 小型应用: 100-200
-- 中型应用: 500-1000
-- 大型应用: 1000-2000
```

## 5. 日志优化

```sql
-- Redo Log 配置
SET GLOBAL innodb_log_file_size = 1073741824;     -- 1GB
SET GLOBAL innodb_log_buffer_size = 67108864;     -- 64MB

-- Binlog 配置
SET GLOBAL binlog_expire_logs_seconds = 604800;   -- 7天
SET GLOBAL max_binlog_size = 104857600;           -- 100MB

-- 慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = ON;

-- 通用查询日志（生产环境关闭）
SET GLOBAL general_log = OFF;
```

## 6. 表优化

```sql
-- 分析表
ANALYZE TABLE users;

-- 优化表（整理碎片）
OPTIMIZE TABLE users;

-- 检查表
CHECK TABLE users;

-- 重建表（InnoDB）
ALTER TABLE users ENGINE=InnoDB;

-- 查看表大小
SELECT 
    table_name,
    table_rows,
    ROUND(data_length / 1024 / 1024, 2) AS data_mb,
    ROUND(index_length / 1024 / 1024, 2) AS index_mb,
    ROUND((data_length + index_length) / 1024 / 1024, 2) AS total_mb
FROM information_schema.tables
WHERE table_schema = 'your_database'
ORDER BY total_mb DESC;
```

## 7. 慢 SQL 分析

```bash
# 使用 mysqldumpslow 分析慢查询日志
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# 参数说明:
# -s t: 按总时间排序
# -s c: 按次数排序
# -t 10: 显示前10条

# 使用 pt-query-digest (Percona Toolkit)
pt-query-digest /var/log/mysql/slow.log > report.txt
```

```sql
-- 使用 performance_schema 分析
SELECT * FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC
LIMIT 10;

-- 查看正在执行的查询
SHOW PROCESSLIST;

-- 杀死慢查询
KILL <process_id>;
```

## 8. 故障排查

### 8.1 连接问题

```sql
-- 问题: Too many connections
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';

-- 解决:
SET GLOBAL max_connections = 2000;
-- 或修改 my.cnf 永久生效

-- 问题: 连接超时
SHOW VARIABLES LIKE 'connect_timeout';
SHOW VARIABLES LIKE 'wait_timeout';

-- 解决:
SET GLOBAL connect_timeout = 10;
SET GLOBAL wait_timeout = 600;
```

### 8.2 锁问题

```sql
-- 查看锁等待
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- 查看当前锁
SELECT * FROM sys.innodb_lock_waits;

-- 杀死阻塞进程
KILL <blocking_pid>;

-- 问题: 长事务
SELECT * FROM information_schema.INNODB_TRX
WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 60;
```

### 8.3 内存问题

```sql
-- 查看内存使用
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- 内存分配
SELECT 
    SUBSTRING_INDEX(event_name, '/', 2) AS code_area,
    FORMAT_BYTES(CURRENT_NUMBER_OF_BYTES_USED) AS current_used
FROM performance_schema.memory_summary_global_by_event_name
ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC
LIMIT 10;
```

### 8.4 磁盘 IO 问题

```sql
-- 查看 IO 状态
SHOW STATUS LIKE 'Innodb_data%';
SHOW STATUS LIKE 'Innodb_os%';

-- 查看表空间
SELECT 
    tablespace_name,
    file_name,
    FORMAT_BYTES(total_extents * extent_size) AS size
FROM information_schema.FILES
WHERE file_type = 'TABLESPACE';
```

## 9. 生产环境配置模板

```ini
# my.cnf 生产环境配置

[mysqld]
# 基础配置
server-id = 1
port = 3306
datadir = /var/lib/mysql
socket = /var/lib/mysql/mysql.sock

# 字符集
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# 连接配置
max_connections = 1000
max_connect_errors = 100
wait_timeout = 600
interactive_timeout = 1800

# InnoDB 配置
innodb_buffer_pool_size = 8G          # 物理内存的70-80%
innodb_buffer_pool_instances = 8      # CPU核心数
innodb_log_file_size = 1G
innodb_log_buffer_size = 64M
innodb_flush_log_at_trx_commit = 1    # 1: 最安全
innodb_flush_method = O_DIRECT
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# 日志配置
log_error = /var/log/mysql/error.log
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = ON

# Binlog 配置
log_bin = /var/log/mysql/mysql-bin
binlog_format = ROW
binlog_expire_logs_seconds = 604800
max_binlog_size = 100M
sync_binlog = 1

# 临时表
tmp_table_size = 64M
max_heap_table_size = 64M

# 排序和连接
sort_buffer_size = 4M
join_buffer_size = 4M
read_buffer_size = 2M
read_rnd_buffer_size = 8M
```

## 10. 调优检查清单

```
MySQL 调优 Checklist:

□ 内存配置:
  - innodb_buffer_pool_size 合理设置
  - 连接内存参数调整
  - 临时表大小配置

□ 查询优化:
  - 开启慢查询日志
  - 定期分析慢 SQL
  - 使用 EXPLAIN 分析执行计划
  - 优化索引设计

□ 锁优化:
  - 监控锁等待
  - 分析死锁日志
  - 优化事务设计
  - 使用合适的隔离级别

□ 日志管理:
  - 合理配置 Binlog
  - 定期清理过期日志
  - 监控错误日志

□ 监控告警:
  - 连接数监控
  - 查询性能监控
  - 锁等待监控
  - 磁盘空间监控
```
