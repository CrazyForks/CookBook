# MySQL 事务与锁机制

## 1. 事务基础

```
事务 ACID 特性:

┌─────────────────────────────────────────────────────────────┐
│                    ACID 特性                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  A (Atomicity) 原子性:                                     │
│  ├── 事务是不可分割的工作单位                              │
│  ├── 要么全部成功，要么全部失败                            │
│  └── 实现: Undo Log                                        │
│                                                             │
│  C (Consistency) 一致性:                                   │
│  ├── 事务前后数据库保持一致状态                            │
│  └── 实现: 由其他三个特性共同保证                          │
│                                                             │
│  I (Isolation) 隔离性:                                     │
│  ├── 并发事务之间互不干扰                                  │
│  └── 实现: MVCC + 锁                                       │
│                                                             │
│  D (Durability) 持久性:                                    │
│  ├── 事务提交后数据永久保存                                │
│  └── 实现: Redo Log                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 事务隔离级别

```
隔离级别:

┌──────────────┬──────────────┬──────────────┬──────────────┐
│ 隔离级别     │ 脏读         │ 不可重复读   │ 幻读         │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ READ UNCOMMITTED │ ✓      │ ✓            │ ✓            │
│ READ COMMITTED   │ ✗      │ ✓            │ ✓            │
│ REPEATABLE READ  │ ✗      │ ✗            │ ✓ (InnoDB ✗) │
│ SERIALIZABLE     │ ✗      │ ✗            │ ✗            │
└──────────────┴──────────────┴──────────────┴──────────────┘

问题说明:
- 脏读: 读取到未提交的数据
- 不可重复读: 同一事务内两次读取结果不同（其他事务修改）
- 幻读: 同一事务内两次查询结果行数不同（其他事务插入/删除）
```

```sql
-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 设置隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- MySQL 默认: REPEATABLE READ
-- 生产推荐: READ COMMITTED (大多数场景)
```

## 3. MVCC 多版本并发控制

```
MVCC 原理:

┌─────────────────────────────────────────────────────────────┐
│                    MVCC 实现机制                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  隐藏字段（每行数据）:                                     │
│  ├── DB_TRX_ID: 最后修改事务 ID                            │
│  ├── DB_ROLL_PTR: 回滚指针，指向 Undo Log                  │
│  └── DB_ROW_ID: 自增行 ID（无主键时使用）                  │
│                                                             │
│  Undo Log 版本链:                                          │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                │
│  │ 当前版本 │←──│ 版本 2  │←──│ 版本 1  │                │
│  │ TRX_ID=3│    │ TRX_ID=2│    │ TRX_ID=1│                │
│  └─────────┘    └─────────┘    └─────────┘                │
│                                                             │
│  Read View（读视图）:                                      │
│  ├── m_ids: 活跃事务 ID 列表                               │
│  ├── min_trx_id: 最小活跃事务 ID                           │
│  ├── max_trx_id: 下一个分配的事务 ID                       │
│  └── creator_trx_id: 创建 Read View 的事务 ID             │
│                                                             │
│  可见性判断:                                                │
│  1. 版本 TRX_ID < min_trx_id → 可见                       │
│  2. 版本 TRX_ID > max_trx_id → 不可见                     │
│  3. 版本 TRX_ID 在 min 和 max 之间:                       │
│     - 在 m_ids 中 → 不可见（未提交）                      │
│     - 不在 m_ids 中 → 可见（已提交）                      │
│                                                             │
│  不同隔离级别的 Read View:                                  │
│  - READ COMMITTED: 每次 SELECT 创建新的 Read View          │
│  - REPEATABLE READ: 事务开始时创建 Read View               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 4. 锁机制

```
InnoDB 锁类型:

┌─────────────────────────────────────────────────────────────┐
│                    InnoDB 锁                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  按粒度分类:                                                │
│  ├── 表锁: 锁定整张表                                      │
│  ├── 行锁: 锁定一行或一个范围                              │
│  └── 页锁: 锁定一页（InnoDB 不支持）                       │
│                                                             │
│  行锁类型:                                                  │
│  ├── Record Lock: 记录锁，锁定单行                         │
│  ├── Gap Lock: 间隙锁，锁定范围（不含记录）                │
│  ├── Next-Key Lock: 临键锁，记录锁 + 间隙锁               │
│  └── Insert Intention Lock: 插入意向锁                     │
│                                                             │
│  按模式分类:                                                │
│  ├── S Lock (Shared Lock): 共享锁，读锁                    │
│  ├── X Lock (Exclusive Lock): 排他锁，写锁                 │
│  ├── IS Lock (Intention Shared): 意向共享锁                │
│  └── IX Lock (Intention Exclusive): 意向排他锁             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- 手动加锁
-- 共享锁
SELECT * FROM users WHERE id = 1 LOCK IN SHARE MODE;

-- 排他锁
SELECT * FROM users WHERE id = 1 FOR UPDATE;

-- 间隙锁（锁定范围）
SELECT * FROM users WHERE age > 18 AND age < 30 FOR UPDATE;
-- 锁定 (18, 30) 范围内的所有记录和间隙
```

## 5. 死锁

```
死锁产生条件:

┌─────────────────────────────────────────────────────────────┐
│                    死锁四要素                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 互斥条件: 资源独占                                      │
│  2. 请求与保持: 持有并等待其他资源                          │
│  3. 不可剥夺: 已获取的资源不能被强制剥夺                    │
│  4. 循环等待: 形成资源等待环                                │
│                                                             │
│  死锁示例:                                                  │
│  事务 A: SELECT * FROM t WHERE id = 1 FOR UPDATE;           │
│  事务 B: SELECT * FROM t WHERE id = 2 FOR UPDATE;           │
│  事务 A: SELECT * FROM t WHERE id = 2 FOR UPDATE; (等待 B)  │
│  事务 B: SELECT * FROM t WHERE id = 1 FOR UPDATE; (等待 A)  │
│                                                             │
│  InnoDB 死锁检测:                                           │
│  - innodb_deadlock_detect = ON (默认开启)                   │
│  - 检测到死锁后回滚代价小的事务                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- 查看死锁信息
SHOW ENGINE INNODB STATUS;

-- 死锁日志分析
-- LATEST DETECTED DEADLOCK 部分

-- 避免死锁:
-- 1. 按固定顺序访问表和行
-- 2. 保持事务简短
-- 3. 使用合适的索引减少锁范围
-- 4. 降低隔离级别
```

## 6. 锁等待与超时

```sql
-- 查看锁等待
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- 查看当前锁
SELECT * FROM information_schema.INNODB_LOCKS;

-- 查看当前事务
SELECT * FROM information_schema.INNODB_TRX;

-- 设置锁等待超时
SET innodb_lock_wait_timeout = 50;  -- 50秒

-- 死锁检测开关
SET innodb_deadlock_detect = ON;
```

## 7. 乐观锁与悲观锁

```sql
-- 悲观锁: 先加锁，再操作
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- 乐观锁: 使用版本号
-- 1. 读取版本号
SELECT balance, version FROM accounts WHERE id = 1;
-- 假设 version = 1, balance = 1000

-- 2. 更新时检查版本号
UPDATE accounts 
SET balance = balance - 100, version = version + 1 
WHERE id = 1 AND version = 1;
-- 返回 affected_rows: 1 表示成功，0 表示被其他事务修改

-- 应用层重试逻辑
-- 如果 affected_rows = 0，则重试
```

## 8. 事务最佳实践

```
事务最佳实践 Checklist:

□ 事务设计:
  - 事务尽量短小
  - 避免事务中包含 RPC 调用
  - 避免大事务（锁定过多行）
  - 合理选择隔离级别

□ 锁优化:
  - 使用索引减少锁范围
  - 按固定顺序访问资源
  - 避免间隙锁（使用唯一索引）
  - 读操作使用快照读

□ 死锁预防:
  - 统一加锁顺序
  - 设置合理的锁超时
  - 使用乐观锁（并发高场景）
  - 定期监控死锁日志

□ 监控告警:
  - 监控锁等待时间
  - 监控死锁发生频率
  - 监控长事务
  - 监控锁冲突
```
