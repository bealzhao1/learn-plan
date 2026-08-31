# Prometheus 与全链路监控

## 28.1 一个必须纠正的前提

「全链路监控」最大的误区是**以为 Prometheus 一个东西就能搞定**。事实上 Prometheus 只负责「指标（Metrics）」这一环，真正的全链路监控是「**指标 + 日志 + 链路追踪**」三位一体：

| 支柱 | 负责回答 | 代表工具 |
|------|----------|----------|
| Metrics（指标） | **哪里**出问题了 | Prometheus |
| Logging（日志） | **为什么**出问题 | Loki / ELK |
| Tracing（追踪） | **慢在哪一跳** | Jaeger / Tempo |

三者用同一个 `trace_id` 串起来，才能从「发现异常」走到「定位根因」。

## 28.2 Prometheus 架构：一个「拉模式」的时序数据库

Prometheus 最反直觉的一点：**它不是被动等数据，而是主动去「抓」**。每个目标暴露一个 `/metrics` 端点，Prometheus 周期性地去拉取。

```
                        ┌──────────────────┐
                        │  Alertmanager     │
                        │  告警去重/路由/静默 │
                        └────────▲─────────┘
                                 │ 触发告警规则
┌──────────┐   scrape  ┌────────┴─────────┐
│ 目标实例   │◄──────────│   Prometheus      │
│ /metrics  │  拉取     │   Server (TSDB)   │
└──────────┘           └────────▲─────────┘
┌──────────┐                    │ PromQL 查询
│ Exporter  │◄──────────────────┤
└──────────┘                    ▼
                         ┌──────────────┐
                         │  Grafana 可视化 │
                         └──────────────┘
```

### 数据模型：`指标名 + 多维标签`

```
http_request_duration_seconds{method="GET", path="/api/order", status="500"} = 0.83
```

靠标签（labels）实现多维查询——同样是 `http_request_duration_seconds`，按 `path`、`method`、`status` 任意切片聚合，这是它和传统「表结构」监控最本质的区别。

### 四种指标类型（埋点时就要选对）

| 类型 | 含义 | 典型场景 | 只能增/可增可减 |
|------|------|----------|----------------|
| **Counter** | 只增不减的计数 | 请求总数、错误总数、累计流量 | 只能增 |
| **Gauge** | 可增可减的瞬时值 | CPU、内存、当前连接数、队列长度 | 可增可减 |
| **Histogram** | 分布统计（分桶） | 请求延迟分布、P50/P95/P99 | 只增 |
| **Summary** | 分位数（客户端算） | 需要精确 P99 时 | 只增 |

> 延迟一定要用 **Histogram** 而不是 Gauge 记平均值——平均值会掩盖长尾问题。

### 拉模式 vs 推模式的取舍

- 拉模式好处：目标挂了 Prometheus 自己知道（拉不到就是告警）。
- 拉模式坏处：短生命周期任务来不及被拉，所以才有 **Pushgateway** 补充。

## 28.3 全链路监控三支柱 + trace_id 打通

```
                      ┌─────────────────────────────────┐
                      │           trace_id             │
                      │  (贯穿请求生命周期的唯一标识)      │
                      └───────┬──────────┬─────────────┘
                              │          │
            ┌─────────────────▼──┐  ┌────▼──────────────┐  ┌──────────────┐
            │   Metrics (聚合)    │  │  Logging (明细)    │  │ Tracing (链路) │
            │  发现"哪里"异常      │  │ 解释"为什么"异常    │  │ 定位"慢在哪跳" │
            └────────────────────┘  └───────────────────┘  └──────────────┘
```

一个完整的排查闭环：

1. 监控告警报「订单服务 P99 延迟飙升」（Metrics 发现异常）；
2. 用 Grafana 关联查询，拿到这段时间内高延迟的 `trace_id`（Metrics → Tracing 跳转）；
3. 在 Jaeger 里看这个 trace，发现慢在「调用支付服务」这一跳（Tracing 定位）；
4. 用 `trace_id` 搜 Loki 日志，看到支付服务报「连接池耗尽」（Logging 解释根因）。

## 28.4 Prometheus 做「全链路」的落地清单

用 **RED + USE** 两个方法论埋点：

**RED 方法（每个服务/接口都要有，看「服务质量」）**
- **R**ate：请求速率
- **E**rrors：错误率
- **D**uration：耗时分布（Histogram）

**USE 方法（每个资源都要有，看「资源瓶颈」）**
- **U**tilization：利用率（CPU、内存、磁盘）
- **S**aturation：饱和度（队列长度、连接数——往往比利用率更早暴露问题）
- **E**rrors：错误数

各层埋点布局：

| 层 | 埋什么 | 用什么 |
|----|--------|--------|
| 基础设施 | CPU/内存/磁盘/网络 | node_exporter |
| 容器 | 容器资源 | cAdvisor / kube-state-metrics |
| 中间件 | Redis/MySQL 连接数、慢查询、命中率 | redis_exporter / mysqld_exporter |
| Nginx 层 | 请求数、5xx、连接数 | nginx-prometheus-exporter |
| 网关层 | **每接口的 QPS、错误率、P99 延迟、限流命中数、熔断次数** | client_golang 埋点 |
| 业务层 | 业务指标（下单量、支付成功率） | 自研埋点 |

## 28.5 几个必须知道的关键点

1. **告警要分层**：P0（服务不可用，5xx 飙升）→ 立即电话/钉钉；P1（P99 延迟超阈值）→ 群通知；P2（容量水位）→ 看板观察。用 `for` 字段做**持续时长判断**（如「5 分钟内错误率 > 1%」），避免瞬时抖动误报。

2. **Prometheus 自身的瓶颈**：它是**单机时序库**，长期数据要外接 **Thanos / Cortex / VictoriaMetrics** 做长期存储和跨集群聚合（否则数据只能存本地磁盘，会撑爆）。

3. **标签的坑**：**标签是高基数陷阱**——不要把 `trace_id`、`user_id` 这种高基数值塞进 Prometheus 标签（会指数级膨胀内存）。trace_id 应该放在**日志和 Tracing** 里，Prometheus 只留低基数标签（path、method、status）。

4. **Prometheus 查不了「单次请求」**：它存的是聚合后的时间序列，**没有 trace_id 级别的数据**。所以「某个用户这次请求为什么慢」这类问题，Prometheus 答不了，必须靠 Tracing + Logging。这正是三支柱缺一不可的原因。

## 28.6 附：Prometheus 开源信息

| 项 | 内容 |
|----|------|
| 主仓库 | `github.com/prometheus/prometheus` |
| 开源协议 | Apache License 2.0 |
| 开发语言 | Go |
| 起源 | 2012 年 SoundCloud 内部孵化 |
| 托管方 | 2016 年加入 CNCF，继 Kubernetes 之后第二个「毕业」项目 |

`prometheus` 组织下还有一整套生态：`alertmanager`（告警）、`node_exporter`（机器指标）、`client_golang`（Go 埋点客户端）、`blackbox_exporter`（拨测）、`pushgateway`（短任务推送）。

## 28.7 面试高频题

1. Prometheus 为什么用「拉模式」而不是「推模式」？
2. Counter 和 Gauge 的区别？延迟为什么用 Histogram？
3. 什么是高基数标签？为什么 trace_id 不能放进 Prometheus 标签？
4. 全链路监控的三大支柱是什么？分别回答什么问题？
5. RED 和 USE 分别代表什么？各用于什么场景？
6. Prometheus 数据量大怎么办？（Thanos / Cortex / VictoriaMetrics）
7. 如何做告警去重和降噪？（Alertmanager 分组 + `for` 持续判断）
8. Prometheus 能查到「单次请求」的明细吗？为什么？
