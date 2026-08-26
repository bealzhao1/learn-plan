# Kafka 场景分析与使用

## 13.1 核心架构

```
Producer ──→ Broker Cluster ──→ Consumer Group
                │
         ┌──────┼──────┐
         ▼      ▼      ▼
      Topic-A  Topic-B Topic-C
      ┌─┬─┬─┐
      │P0│P1│P2│  (分区 Partition)
      └─┴─┴─┘
       │
       ▼
    Segment (日志分段文件)
    ├── .log   (消息数据)
    ├── .index (稀疏偏移量索引)
    └── .timeindex (时间索引)
```

## 13.2 核心特性

| 特性 | 说明 |
|------|------|
| **高吞吐** | 顺序写磁盘 + 零拷贝 (sendfile) + 批量压缩 |
| **持久化** | 消息持久化到磁盘，可配置保留策略 |
| **分区有序** | 单分区内消息严格有序 |
| **Consumer Group** | 分区在组内负载均衡，组间广播 |
| **Exactly Once** | 幂等 Producer + 事务 |
| **ISR 副本机制** | In-Sync Replicas 保证数据不丢 |

## 13.3 典型使用场景

| 场景 | 说明 | 配置要点 |
|------|------|---------|
| 日志收集 | ELK/EFK 的传输层 | 高吞吐，retention 7天 |
| 事件驱动 | 微服务解耦 | 保证顺序，幂等消费 |
| 流处理 | Kafka Streams / Flink | Exactly Once 语义 |
| CDC | 数据库变更捕获 | Debezium + Kafka Connect |
| 指标监控 | 时序数据管道 | 高吞吐，短保留 |
| 削峰填谷 | 应对流量突峰 | 大分区数，消费者弹性伸缩 |

## 13.4 生产最佳实践

```properties
# Producer 关键配置
acks=all                    # 所有 ISR 确认
retries=Integer.MAX_VALUE   # 无限重试
max.in.flight.requests.per.connection=5  # 配合幂等
enable.idempotence=true     # 幂等生产者
compression.type=lz4        # 压缩

# Consumer 关键配置
enable.auto.commit=false    # 手动提交 offset
max.poll.records=500        # 批量拉取
session.timeout.ms=45000    # 心跳超时
```

## 13.5 消息可靠性保证

```
Producer                  Broker                  Consumer
   │                        │                        │
   ├── acks=all ───────────►│                        │
   │                        ├── ISR 全部写入          │
   │◄── ACK ───────────────┤                        │
   │                        │                        │
   │                        ├── 分区副本同步           │
   │                        │                        │
   │                        │───────────────────────►│
   │                        │                 手动commit offset
   │                        │◄───────────────────────┤
```
