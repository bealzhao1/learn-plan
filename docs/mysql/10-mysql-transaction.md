# MySQL 事务

## 10.1 ACID 特性

| 特性 | 含义 | 实现机制 |
|------|------|---------|
| **A** (Atomicity) | 全做或全不做 | undo log（回滚日志） |
| **C** (Consistency) | 数据满足约束 | 由 A+I+D 共同保证 |
| **I** (Isolation) | 并发事务互不干扰 | MVCC + 锁 |
| **D** (Durability) | 提交后永久生效 | redo log + 双写缓冲 |

## 10.2 隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 | 实现 |
|------|------|-----------|------|------|
| READ UNCOMMITTED | ✅ | ✅ | ✅ | 无 MVCC |
| READ COMMITTED | ❌ | ✅ | ✅ | 语句级快照 |
| **REPEATABLE READ** | ❌ | ❌ | ⚠️ | 事务级快照 + Gap Lock |
| SERIALIZABLE | ❌ | ❌ | ❌ | 全部加锁 |

> InnoDB 默认 RR 级别，通过 **Next-Key Lock (行锁+间隙锁)** 解决大部分幻读问题

## 10.3 MVCC (多版本并发控制)

```
每行数据隐藏列：
┌─────────┬──────────────┬──────────────┬─────────┐
│  数据    │ DB_TRX_ID    │ DB_ROLL_PTR  │ DB_ROW_ID│
│         │ (最后修改事务)│ (指向undo log)│ (隐藏主键)│
└─────────┴──────────────┴──────────────┴─────────┘

Read View（快照读时生成）：
- m_ids:       当前活跃事务 ID 列表
- min_trx_id:  最小活跃事务 ID
- max_trx_id:  下一个将分配的事务 ID
- creator_trx_id: 创建 Read View 的事务 ID

可见性判断：
- trx_id < min_trx_id  → 可见（事务已提交）
- trx_id >= max_trx_id → 不可见（事务在快照后开始）
- min_trx_id <= trx_id < max_trx_id:
    - 在 m_ids 中 → 不可见（事务未提交）
    - 不在 m_ids 中 → 可见（事务已提交）
```

## 10.4 锁机制

| 锁类型 | 粒度 | 说明 |
|--------|------|------|
| 行锁 (Record Lock) | 行 | 锁定索引记录 |
| 间隙锁 (Gap Lock) | 间隙 | 锁定索引记录之间的间隙，防止幻读 |
| 临键锁 (Next-Key Lock) | 行+间隙 | Record Lock + Gap Lock |
| 意向锁 (IS/IX) | 表 | 表明事务意图，加速锁兼容性判断 |
| 自增锁 (AUTO-INC) | 表 | 自增列的分配 |

## 10.5 死锁处理

```sql
-- 查看死锁信息
SHOW ENGINE INNODB STATUS;

-- 预防策略
-- 1. 事务尽量短小
-- 2. 按固定顺序访问表和行
-- 3. 合理使用索引，减少锁范围
-- 4. 降低隔离级别（RC 无 Gap Lock）
-- 5. InnoDB 超时自动回滚：innodb_lock_wait_timeout = 50s
```
