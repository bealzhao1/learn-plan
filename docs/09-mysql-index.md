# MySQL 索引

## 9.1 索引数据结构

**B+ 树（InnoDB 默认）**

```
                    [10 | 20 | 30]              ← 非叶子节点（只存键+指针）
                   /    |     |    \
        [1,3,5,8] [11,15,18] [21,25,28] [31,35,40]  ← 叶子节点（存数据，双向链表）
            ↔           ↔          ↔          ↔
```

**特点**：
- 非叶子节点只存索引键，扇出度大（一个 16KB 页可存上千键）
- 叶子节点存实际数据/主键，通过双向链表相连
- 树高通常 2~4 层 → 亿级数据仅需 3~4 次 I/O

## 9.2 索引类型

| 类型 | 说明 | 适用场景 |
|------|------|---------|
| 主键索引（聚簇） | 数据按主键物理排列 | 每表仅一个 |
| 唯一索引 | 值不可重复 | 去重约束 |
| 普通索引 | 最基本的索引 | 加速查询 |
| 联合索引 | 多列组合 | 多条件查询 |
| 前缀索引 | 只索引列的前 N 个字符 | 长字符串 |
| 覆盖索引 | 查询列全在索引中 | 避免回表 |
| 全文索引 | 文本搜索 | LIKE '%word%' 替代 |

## 9.3 索引核心原则

**最左前缀原则**
```sql
-- 联合索引 (a, b, c)
WHERE a = 1 AND b = 2 AND c = 3  -- ✅ 完全命中
WHERE a = 1 AND b = 2            -- ✅ 命中 (a,b)
WHERE a = 1                      -- ✅ 命中 (a)
WHERE b = 2 AND c = 3            -- ❌ 无法使用索引（缺少最左列）
WHERE a = 1 AND c = 3            -- ⚠️ 只命中 (a)，c 无法用索引
```

**索引下推 (ICP, Index Condition Pushdown)**
- MySQL 5.6+ 优化
- 在存储引擎层就过滤不满足条件的记录
- 减少回表次数

**回表与覆盖索引**
```sql
-- 需要回表（二级索引找到主键 → 再查聚簇索引）
SELECT * FROM users WHERE name = 'Tom';

-- 覆盖索引（不需要回表）
SELECT id, name FROM users WHERE name = 'Tom';
-- 前提：(name) 索引包含 id（主键自动附加）
```

## 9.4 索引优化实战

```sql
-- 1. EXPLAIN 分析
EXPLAIN SELECT * FROM orders WHERE user_id = 100 AND status = 'paid';

-- 关注字段：type(ALL=全扫/ref=索引), key(使用的索引), rows(扫描行数), Extra

-- 2. 索引失效场景
WHERE YEAR(created_at) = 2024      -- ❌ 函数操作导致失效
WHERE name LIKE '%test%'           -- ❌ 左模糊匹配失效
WHERE age != 20                    -- ❌ 不等于难以使用索引
WHERE age + 1 > 10                 -- ❌ 表达式计算失效
WHERE varchar_col = 123            -- ❌ 隐式类型转换失效

-- 3. 优化建议
-- 小表驱动大表
SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE vip = 1);
-- 优于
SELECT * FROM orders o JOIN users u ON o.user_id = u.id WHERE u.vip = 1;
```
