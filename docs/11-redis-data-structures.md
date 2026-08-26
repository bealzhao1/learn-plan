# Redis 数据结构

## 11.1 底层数据结构

| 底层结构 | 说明 |
|----------|------|
| **SDS** (Simple Dynamic String) | 动态字符串，O(1) 获取长度，二进制安全 |
| **ziplist** (压缩列表) | 紧凑的连续内存，小数据省空间 |
| **listpack** | ziplist 升级版（Redis 7.0），解决级联更新问题 |
| **quicklist** | ziplist/listpack 组成的双向链表 |
| **hashtable** | 渐进式 rehash 的哈希表 |
| **skiplist** (跳表) | 有序结构，O(logN) 查找，实现简单 |
| **intset** | 整数集合，有序紧凑 |

## 11.2 数据类型与编码

| 类型 | 底层编码 | 使用场景 |
|------|---------|---------|
| **String** | int / embstr / raw | 缓存、计数器、分布式锁 |
| **List** | quicklist | 消息队列、时间线 |
| **Hash** | listpack / hashtable | 对象存储、购物车 |
| **Set** | intset / hashtable | 标签、共同好友、去重 |
| **ZSet** | listpack / skiplist+hashtable | 排行榜、延迟队列 |
| **Stream** | Radix Tree + listpack | 消息队列（带消费组） |
| **Bitmap** | String (位操作) | 签到、布隆过滤器 |
| **HyperLogLog** | 稀疏/稠密编码 | 基数统计（UV） |
| **GEO** | ZSet (geohash 编码) | 附近的人 |

## 11.3 跳表 (ZSet 核心)

```
Level 4:  HEAD ──────────────────────────────────────→ 50 ──→ NULL
Level 3:  HEAD ──────────── 20 ──────────────────────→ 50 ──→ NULL
Level 2:  HEAD ──→ 10 ──→ 20 ──────────── 40 ──────→ 50 ──→ NULL
Level 1:  HEAD ──→ 10 ──→ 20 ──→ 30 ──→ 40 ──→ 45 → 50 ──→ NULL
```

- 多层索引，平均 O(logN) 时间复杂度
- 比平衡树实现简单，范围查询高效
- 每个节点随机层数（概率 p=0.25）
