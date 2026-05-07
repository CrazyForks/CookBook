# MySQL 索引原理与优化

## 1. 索引概述

```
为什么需要索引？

┌─────────────────────────────────────────────────────────────┐
│                    索引的作用                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  无索引: 全表扫描（Full Table Scan）                       │
│  SELECT * FROM users WHERE email = 'test@example.com';      │
│  → 扫描 100 万行，逐行比较                                 │
│  → 耗时: 1000ms+                                           │
│                                                             │
│  有索引: 索引查找（Index Lookup）                           │
│  → 通过 B+ 树快速定位                                      │
│  → 耗时: 1ms                                               │
│                                                             │
│  索引类型:                                                  │
│  ├── B+ 树索引: 最常用，支持范围查询                       │
│  ├── 哈希索引: 等值查询快，不支持范围                       │
│  ├── 全文索引: 文本搜索                                    │
│  └── 空间索引: 地理位置查询                                │
│                                                             │
│  索引分类:                                                  │
│  ├── 主键索引: PRIMARY KEY                                 │
│  ├── 唯一索引: UNIQUE                                      │
│  ├── 普通索引: INDEX                                       │
│  ├── 联合索引: 多列组合索引                                │
│  └── 覆盖索引: 查询列都在索引中                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. B+ 树原理

```
B+ 树结构:

┌─────────────────────────────────────────────────────────────┐
│                    B+ 树特点                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                    [30|60]         ← 非叶子节点（索引）      │
│                   /    |    \                               │
│                  /     |     \                              │
│         [10|20]    [40|50]    [70|80]  ← 非叶子节点         │
│        /  |  \    /  |  \    /  |  \                       │
│       ↓   ↓   ↓  ↓   ↓   ↓  ↓   ↓   ↓                     │
│      [10][20][30][40][50][60][70][80][90] ← 叶子节点（数据）│
│       ←───→←───→←───→←───→←───→←───→                      │
│       双向链表连接所有叶子节点                              │
│                                                             │
│  B+ 树优势:                                                │
│  ├── 所有数据都在叶子节点，查询稳定                        │
│  ├── 叶子节点双向链表，支持范围查询                        │
│  ├── 非叶子节点只存索引，能存更多 key                      │
│  └── 树高度低（3-4层可存千万级数据）                       │
│                                                             │
│  估算:                                                     │
│  - 假设每个节点 16KB，每个 key 8B，指针 6B                 │
│  - 每个节点可存: 16KB / 14B ≈ 1170 个 key                  │
│  - 3 层 B+ 树: 1170 × 1170 × 16 ≈ 2000 万行               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 3. 索引使用原则

### 3.1 最左前缀原则

```sql
-- 联合索引: INDEX idx_name_age_email (name, age, email)

-- ✓ 能使用索引
SELECT * FROM users WHERE name = '张三';
SELECT * FROM users WHERE name = '张三' AND age = 25;
SELECT * FROM users WHERE name = '张三' AND age = 25 AND email = 'test@ex.com';
SELECT * FROM users WHERE name = '张三' AND email = 'test@ex.com';  -- 只用到 name

-- ✗ 不能使用索引
SELECT * FROM users WHERE age = 25;
SELECT * FROM users WHERE email = 'test@ex.com';
SELECT * FROM users WHERE age = 25 AND email = 'test@ex.com';

-- 最左前缀原则:
-- 联合索引从最左边列开始匹配，遇到范围查询停止匹配
-- INDEX(a, b, c) 支持: a / a,b / a,b,c
-- INDEX(a, b, c) 不支持: b / c / b,c
```

### 3.2 索引失效场景

```sql
-- 1. 对索引列使用函数
-- ✗ 索引失效
SELECT * FROM users WHERE YEAR(create_time) = 2024;
-- ✓ 改写
SELECT * FROM users WHERE create_time >= '2024-01-01' 
  AND create_time < '2025-01-01';

-- 2. 对索引列进行运算
-- ✗ 索引失效
SELECT * FROM users WHERE age + 1 = 25;
-- ✓ 改写
SELECT * FROM users WHERE age = 24;

-- 3. 隐式类型转换
-- ✗ phone 是 varchar，传入 int，索引失效
SELECT * FROM users WHERE phone = 13800138000;
-- ✓ 改写
SELECT * FROM users WHERE phone = '13800138000';

-- 4. LIKE 左模糊
-- ✗ 索引失效
SELECT * FROM users WHERE name LIKE '%三';
-- ✓ 右模糊可以使用索引
SELECT * FROM users WHERE name LIKE '张%';

-- 5. OR 条件
-- ✗ 如果 OR 两侧有非索引列，索引失效
SELECT * FROM users WHERE name = '张三' OR age = 25;
-- ✓ 改写为 UNION
SELECT * FROM users WHERE name = '张三'
UNION
SELECT * FROM users WHERE age = 25;

-- 6. NOT IN / NOT EXISTS / !=
-- ✗ 可能索引失效
SELECT * FROM users WHERE status != 1;
SELECT * FROM users WHERE status NOT IN (1, 2);

-- 7. IS NULL / IS NOT NULL
-- ✗ 可能索引失效（取决于数据分布）
SELECT * FROM users WHERE name IS NOT NULL;
```

## 4. 索引设计原则

```
索引设计 Checklist:

□ 适合创建索引:
  - WHERE 条件频繁使用的列
  - JOIN 连接条件的列
  - ORDER BY 排序的列
  - GROUP BY 分组的列
  - 高选择性的列（重复值少）

□ 避免创建索引:
  - 数据量小的表
  - 频繁更新的列
  - 低选择性的列（如性别）
  - 不在查询条件中的列

□ 联合索引设计:
  - 区分度高的列放前面
  - 频繁查询的列放前面
  - 排序字段放最后

□ 索引数量控制:
  - 单表索引不超过 5-6 个
  - 单个索引字段不超过 5 个
  - 定期清理无用索引
```

## 5. 执行计划分析

```sql
-- EXPLAIN 查看执行计划
EXPLAIN SELECT * FROM users WHERE name = '张三';

-- 关键字段说明
-- id: 查询序号
-- select_type: 查询类型（SIMPLE/PRIMARY/SUBQUERY）
-- table: 表名
-- type: 访问类型
-- possible_keys: 可能使用的索引
-- key: 实际使用的索引
-- key_len: 索引长度
-- rows: 预估扫描行数
-- Extra: 额外信息
```

```
type 访问类型（从好到差）:

┌──────────────┬──────────────────────────────────────────────┐
│ 类型         │ 说明                                          │
├──────────────┼──────────────────────────────────────────────┤
│ system       │ 表只有一行记录                                │
│ const        │ 主键或唯一索引等值查询                        │
│ eq_ref       │ 主键或唯一索引关联查询                        │
│ ref          │ 普通索引等值查询                              │
│ range        │ 索引范围查询                                  │
│ index        │ 全索引扫描                                    │
│ ALL          │ 全表扫描（需要优化）                          │
└──────────────┴──────────────────────────────────────────────┘

Extra 常见值:
- Using index: 覆盖索引，性能好
- Using where: 在存储引擎层后进行过滤
- Using temporary: 使用临时表
- Using filesort: 额外排序（需要优化）
```

## 6. 慢 SQL 优化

### 6.1 开启慢查询日志

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过1秒记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未使用索引的查询
```

### 6.2 优化案例

```sql
-- 案例1: 分页查询优化
-- ✗ 慢查询
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;
-- ✓ 优化: 使用子查询或游标
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;

-- 案例2: COUNT 优化
-- ✗ 全表统计
SELECT COUNT(*) FROM orders WHERE status = 1;
-- ✓ 使用索引统计
SELECT COUNT(*) FROM orders USE INDEX(idx_status) WHERE status = 1;

-- 案例3: JOIN 优化
-- ✗ 笛卡尔积
SELECT * FROM orders o, users u WHERE o.user_id = u.id;
-- ✓ 显式 JOIN
SELECT * FROM orders o INNER JOIN users u ON o.user_id = u.id;
-- ✓ 确保 JOIN 列有索引

-- 案例4: IN 子查询优化
-- ✗ 子查询
SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE status = 1);
-- ✓ 改写为 JOIN
SELECT o.* FROM orders o INNER JOIN users u ON o.user_id = u.id WHERE u.status = 1;

-- 案例5: 索引合并
-- UNION 代替 OR
SELECT * FROM users WHERE name = '张三' OR email = 'test@ex.com';
-- 改写为
SELECT * FROM users WHERE name = '张三'
UNION
SELECT * FROM users WHERE email = 'test@ex.com';
```

## 7. 覆盖索引

```sql
-- 覆盖索引: 查询列都在索引中，无需回表

-- 联合索引: INDEX idx_name_age (name, age)

-- ✓ 覆盖索引（Extra: Using index）
SELECT name, age FROM users WHERE name = '张三';

-- ✗ 需要回表（Extra: 无 Using index）
SELECT * FROM users WHERE name = '张三';

-- 优化: 将查询列加入索引
ALTER TABLE users ADD INDEX idx_name_age_email (name, age, email);
```

## 8. 索引监控

```sql
-- 查看索引使用情况
SELECT * FROM sys.schema_unused_indexes;
SELECT * FROM sys.schema_redundant_indexes;

-- 查看表的索引
SHOW INDEX FROM users;

-- 查看索引大小
SELECT 
    database_name,
    table_name,
    index_name,
    stat_value * @@innodb_page_size AS index_size_bytes
FROM mysql.innodb_index_stats
WHERE stat_name = 'size';
```

## 9. 生产环境索引策略

```
索引优化 Checklist:

□ 新建索引前:
  - 使用 EXPLAIN 分析查询
  - 评估索引选择性
  - 考虑联合索引设计
  - 测试索引效果

□ 索引维护:
  - 定期分析表统计信息
  - 监控未使用索引
  - 清理冗余索引
  - 重建碎片化索引

□ 查询优化:
  - 避免 SELECT *
  - 使用覆盖索引
  - 合理使用 LIMIT
  - 避免索引失效写法

□ 监控告警:
  - 慢查询日志分析
  - 执行计划监控
  - 索引使用率统计
```
