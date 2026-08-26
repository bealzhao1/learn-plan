# Kafka vs Pulsar 对比

## 14.1 架构对比

| 维度 | Kafka | Pulsar |
|------|-------|--------|
| **架构** | Broker = 计算+存储一体 | Broker(计算) + BookKeeper(存储) 分离 |
| **分区模型** | Partition 绑定 Broker | Topic → Segment 分散到 Bookie |
| **扩容** | 需 Rebalance 分区 | 无需迁移数据，加 Broker/Bookie 即可 |
| **多租户** | 不原生支持 | 原生 Tenant / Namespace |
| **消息模型** | Pull（拉） | Push + Pull 混合 |
| **消费模式** | Consumer Group | Exclusive / Shared / Failover / Key_Shared |
| **延迟消息** | 不原生支持 | 原生支持 |
| **消息确认** | Offset 提交 | 单条 ACK / 累积 ACK |
| **协议支持** | Kafka Protocol | Kafka/AMQP/MQTT 多协议适配 |
| **Geo 复制** | MirrorMaker（需额外部署） | 原生跨地域复制 |

## 14.2 选型建议

```
选 Kafka 当：
├── 团队已有 Kafka 经验和运维能力
├── 主要是大数据 / 流处理场景 (Flink/Spark/KSQL)
├── 生态工具依赖重（Debezium, Kafka Connect, Schema Registry）
├── 日志收集、指标管道等高吞吐简单场景
└── 不需要复杂消息路由

选 Pulsar 当：
├── 需要计算存储分离（云原生弹性伸缩）
├── 多租户隔离需求（SaaS 平台）
├── 需要原生延迟消息 / 死信队列
├── 需要灵活的消费模式（Shared 多消费者并发）
├── 跨地域复制 / 灾备要求高
└── 同时需要流处理 + 传统消息队列
```

## 14.3 性能对比

| 指标 | Kafka | Pulsar |
|------|-------|--------|
| 吞吐量（MB/s） | 极高（顺序 I/O 优势） | 高（BookKeeper 写入开销略大）|
| 延迟（P99） | 低（批量优化） | 极低（立即持久化+推送）|
| 尾延迟 | 可能抖动（GC/Rebalance） | 更稳定（存储分离）|
| 分区数扩展 | 数千分区后性能下降 | 百万级 Topic 无压力 |
| 存储效率 | 高（顺序写 log） | 较高（BookKeeper 分段存储）|
