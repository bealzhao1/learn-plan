# Redis 场景分析与使用

## 12.1 缓存策略

| 策略 | 流程 | 适用场景 |
|------|------|---------|
| Cache Aside | 读: 先缓存→未命中→查DB→写缓存 | 通用，一致性要求适中 |
| Read Through | 缓存层透明代理读取 | 缓存框架封装 |
| Write Through | 同步写缓存+DB | 一致性要求高 |
| Write Behind | 异步批量写DB | 写密集，允许短暂不一致 |

## 12.2 经典场景

**1. 分布式锁**
```redis
-- 加锁（原子操作）
SET lock_key unique_value NX PX 30000

-- 解锁（Lua 保证原子性）
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```
> 生产环境推荐 Redisson（看门狗续期）或 RedLock

**2. 缓存穿透/击穿/雪崩**

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| **穿透** | 查询不存在的数据 | 布隆过滤器 / 缓存空值 |
| **击穿** | 热点 key 过期瞬间大量请求 | 互斥锁重建 / 永不过期+异步更新 |
| **雪崩** | 大量 key 同时过期 | 过期时间加随机值 / 多级缓存 |

**3. 排行榜**
```redis
ZADD leaderboard 1000 "player:1"
ZADD leaderboard 2000 "player:2"
ZREVRANGE leaderboard 0 9 WITHSCORES   -- Top 10
ZREVRANK leaderboard "player:1"         -- 排名
ZINCRBY leaderboard 100 "player:1"      -- 加分
```

**4. 限流器（滑动窗口）**
```redis
-- 滑动窗口限流
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
local count = redis.call('ZCARD', key)
if count < limit then
    redis.call('ZADD', key, now, now .. math.random())
    redis.call('EXPIRE', key, window / 1000)
    return 1
end
return 0
```

**5. 延迟队列**
```redis
-- 生产者：添加延迟任务
ZADD delay_queue <执行时间戳> <任务JSON>

-- 消费者：轮询到期任务
ZRANGEBYSCORE delay_queue 0 <当前时间戳> LIMIT 0 10
ZREM delay_queue <任务>
```

## 12.3 Redis 高可用架构

| 架构 | 节点数 | 特点 |
|------|--------|------|
| 主从复制 | 1主N从 | 读写分离，手动故障转移 |
| Sentinel | 3+ 哨兵 | 自动故障转移，不支持分片 |
| Cluster | 6+ (3主3从) | 16384 slot 分片，自动迁移 |
