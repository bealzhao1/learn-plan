# GMP 调度模型（深度版）

> 基于 Go 1.21+ runtime/runtime2.go、proc.go 源码解析

## 1. 为什么需要 GMP？

### 1.1 演进历史

```
Go 1.0:  GM 模型（无 P）
   └── 问题：G 切换需要全局锁；线程 M 与 CPU 强耦合
Go 1.1:  引入 P（Processor）
   └── 每个 P 拥有本地队列，显著降低锁竞争
Go 1.14: 异步抢占（基于 SIGURG 信号）
   └── 解决 for{} 无法被调度的问题
Go 1.17: P 数量从 GOMAXPROCS 改为按需初始化
   └── 降低 idle 集群内存占用
```

### 1.2 GM 模型的问题

```go
// GM 模型下的全局队列（伪代码）
global_runq.lock.Lock()
g := global_runq.head
global_runq.head = g.next
global_runq.lock.Unlock()
// 任何 M 想取 G 都得抢一把全局锁 → 性能瓶颈
```

## 2. GMP 三元组精解

### 2.1 G（Goroutine）

```go
// runtime/runtime2.go
type g struct {
    stack       stack       // 当前 goroutine 栈
    stackguard0 uintptr     // 栈保护边界（抢占信号检测点）
    m           *m          // 当前关联的 M
    sched       gobuf       // 调度上下文（PC/SP/寄存器）
    atomicstatus uint32     // 状态原子变量
    goid        int64       // goroutine ID
    gopc        uintptr     // 创建该 g 的 PC
    startpc     uintptr     // goroutine 函数入口
    waitsince   int64       // 被阻塞的时间
    waitreason  waitReason  // 阻塞原因
    // ... timer、channel、定时器链表等
}
```

**G 的关键状态机**：

```
        ┌──── goid 分配 ────┐
        ▼                   │
   Gidle ──► Grunnable ◄────┤
                │           │
                ▼           │
         Gexecuting ──► Gosuspended (等待资源)
                │           │
                │     ┌─────┘
                ▼     ▼
            Gdead (栈释放，归还 pool)
                │
              重新入 gfree 队列复用
```

| 状态 | 含义 |
|------|------|
| `_Gidle` | 刚分配未初始化 |
| `_Grunnable` | 在运行队列上，等待被调度 |
| `_Grunning` | 正在某个 M 上执行 |
| `_Gsyscall` | 正在执行系统调用 |
| `_Gwaiting` | 阻塞等待（channel、网络、锁） |
| `_Gdead` | 已结束，待回收复用 |
| `_Gcopystack` | 栈正在拷贝（动态扩容时） |

### 2.2 M（Machine）

```go
type m struct {
    g0          *g          // 特殊 g，用于调度器自身栈
    curg        *g          // 当前正在运行的 g
    p           *p          // 当前关联的 P
    nextp       *p          // 即将切换到的 P
    park        note        // 唤醒通道（park/unpark）
    alllink     *m          // allm 链表
    schedlink   muintptr    // M 链表节点
    spinning    bool        // 是否处于自旋状态
    laddr       uintptr     // 信号处理函数地址
    sigmask     sigset      // 信号屏蔽
    // ... tls、cgo 句柄
}
```

**M 的关键概念**：
- **M0**：程序启动时的主线程（编号 0），对应主 OS 线程
- **g0**：每个 M 都有一个特殊的 g0，栈是 M 自己的栈（线程栈）
  - g0 不在运行队列上，只用于执行调度逻辑
  - 用户 g 通过 m.g0.sched 切换回调度器

**M 的数量**：
- 默认上限 10000（`sched.maxmcount`）
- 不受 GOMAXPROCS 控制，按需创建
- 进入 idle 后等待 5 分钟（`GCPOLL`）自动回收

### 2.3 P（Processor）

```go
type p struct {
    id          int32       // P 的编号（0..GOMAXPROCS-1）
    status      uint32      // _Pidle/_Prunning/_Psyscall/_Pgcstop/_Pdead
    m           *m          // 关联的 M
    mcache      *mcache     // 小对象分配缓存（关键！）
    pcache      pageCache   // 页分配缓存
    runq        [256]guintptr // 本地运行队列（数组，无锁 CAS）
    runqtail    uint32
    runqhead    uint32
    runnext     guintptr   // 下一个要执行的 G（优先级提升）
    // ... 计时器、抢占标记、GC 状态等
}
```

**P 的关键作用**：
- **本地队列（LRQ）**：256 槽位的环形队列，**无锁访问**
- **mcache**：每个 P 一个，goroutine 小对象分配几乎无锁
- **runnext**：下一个要执行的 G，新创建的 G 直接放这里（高优先级）
- **gFree 队列**：回收的 G 复用，减少栈分配开销

**P 的数量 = `GOMAXPROCS`**（默认 = CPU 核数）
- P 在 Go 1.17 之前：启动时一次性创建
- Go 1.17 之后：按需从 idle p 池分配，节省内存

## 3. 调度器核心数据结构

### 3.1 全局调度器 schedt

```go
type schedt struct {
    lock        mutex        // 全局调度锁
    gowait      uint32       // 等待中的 G 数
    midle        muintptr    // idle M 链表
    nmidle       int32       // idle M 数量
    nmidlelocked int32
    maxmcount    int32       // M 上限
    pidle        puintptr    // idle P 链表
    npidle       uint32      // idle P 数量
    runqsize     int32       // 全局 G 队列大小
    runq         gQueue      // 全局 GRQ
    gflock       mutex
    gfreeStack   gList       // 大栈 g 回收队列
    gfreeNoStack gList       // 无栈 g 回收队列
    // ... GC、sysmon、timer 字段
}
```

**全局队列（GRQ）**：所有 P 的本地队列满后溢出的 G；sysmon 也会将长时间未调度的 G 放入。

### 3.2 调度关系全景

```
┌───────────────────────────────────────────────────────┐
│                  runtime.schedt                       │
│   runq (GRQ)    pidle    midle    gfree              │
└────────┬────────────┬────────────────┬───────────────┘
         │            │                │
         ▼            ▼                ▼
    ┌─────────────────────────────────────────┐
    │              Processor P[]               │
    │   P0.runq  P1.runq  P2.runq  ...        │
    │   P0.mcache P1.mcache P2.mcache ...      │
    └────────┬──────────────┬──────────────────┘
             │              │
             ▼              ▼
    ┌─────────────────────────────────────────┐
    │               Thread M[]                 │
    │     M0(g0)   M1   M2   M3 ...          │
    └─────────────────────────────────────────┘
```

## 4. 调度核心循环 `runtime.schedule()`

```go
// runtime/proc.go (精简)
func schedule() {
    // 1. 每调度 61 次，从 GRQ 取一次（防止饿死）
    if gp == nil {
        if _g_.m.p.ptr().schedtick%61 == 0 {
            gp = globrunqget(_g_.m.p.ptr(), 1)
        }
    }

    // 2. 从 P 的本地队列取
    if gp == nil {
        gp = runqget(_g_.m.p.ptr())
    }

    // 3. 还没有？从其他 P 偷、全局队列、网络轮询器
    if gp == nil {
        gp = findrunnable()  // ★ 最耗时的函数
    }

    // 4. 执行 G
    execute(gp)
}

func findrunnable() (gp *g) {
    // 1. 本地 P 队列
    // 2. 全局 GRQ（每次最多偷 128 个）
    // 3. 从所有 P 偷（work stealing）
    // 4. 网络轮询器 (netpoll)
    // 5. 还没找到？自旋等
}
```

## 5. 调度策略详解

### 5.1 本地队列优先（Local Run Queue）

```go
func runqget(pp *p) *g {
    // 优先拿 runnext（最新创建的 G）
    if gp := pp.runnext; gp != nil {
        pp.runnext = 0
        return gp
    }
    // CAS 弹出队首
    for {
        h := atomic.Load(&pp.runqhead)
        t := pp.runqtail
        if t == h { return nil }
        gp := pp.runq[(h+1)%256]
        if atomic.CAS(&pp.runqhead, h, h+1) {
            return gp
        }
    }
}
```

### 5.2 Work Stealing（工作窃取）

```go
// 找到其他非空 P 的队列，从尾部偷一半
func runqsteal(pp, p2 *p) *g {
    for i := 0; i < len(pp.runq); i++ {
        h := atomic.Load(&p2.runqhead)
        t := atomic.LoadAcq(&p2.runqtail)
        n := t - h
        n = n - n/2  // 偷一半
        if gp := p2.runq[(h+n)%256]; atomic.CAS(...) {
            // 复制 n 个 G 到 pp.runq 尾部
        }
    }
}
```

**为什么从尾部偷？**
- 队尾是较老的 G，分配概率更均匀
- 队首通常是高频热点 G（保留在原队列命中率更高）

### 5.3 Hand Off（移交机制）

```
M 即将阻塞 (syscall/block)：
   1. 解绑 P 与 M
   2. 把 P 放到 _Psyscall 状态
   3. P 进入 pidle 链表，等待新 M 接管
   4. sysmon 监控 10ms 内未恢复 → 创建新 M 接管 P
   5. sysmon 监控前恢复 → P 直接复用，避免切换

回收 M（sysmon）：
   1. syscall M 在等待新 G 时，2 次 schedule 后仍未运行 G
   2. M 归还 P，标记 g0 进入 park
   3. 空闲 5 分钟后被 GC 释放
```

### 5.4 Spinning Thread（自旋线程）

**目的**：消除 P 与 M 绑定的延迟

```go
// 自旋条件：P 有任务但无 idle M
func resetspinning() {
    // 当前 M 加入 sched.midle 链表
    // 如果还有需要执行的工作 → 唤醒一个自旋线程
    if sched.nmspinning.Load() == 0 && sched.pidle != nil {
        wakeM()
    }
}
```

**自旋的代价**：占用 CPU 但不执行用户代码
**收益**：避免 M 创建/销毁延迟（~1000 个时钟周期）

**约束**：
- 至少保留 1 个自旋 M
- 全局自旋上限 = GOMAXPROCS

## 6. 抢占式调度

### 6.1 Go 1.13 之前：协作式抢占

```
调度点（function call） → 插入抢占检查 → 检查 sched.gcwaiting 等标志
```

**问题**：纯 `for {}` 死循环永远不调度！

### 6.2 Go 1.14+：基于信号的异步抢占

```go
// 源码位置：runtime/signal_unix.go
func preemptM(mp *m) {
    // 1. 设置 mp.gsignal 标志
    mp.preemptGen.Add(1)

    // 2. 向 M 关联的 OS 线程发送 SIGURG
    signalM(mp, sigPreempt)
}

// 信号处理函数
func sighandler(sig uint32, info *siginfo, ctxt unsafe.Pointer, gp *g) {
    // 检查是否是抢占信号
    if sig == sigPreempt && gp.m.laddr == getcallerpc() {
        // 保存现场并切回调度器
        asyncPreempt()
    }
}
```

**SIGURG 的妙用**：默认 handler 是 Go runtime 独占的，不会与其他信号冲突。

### 6.3 抢占检查点

| 检查点 | 触发 |
|--------|------|
| 函数调用（栈增长检查） | 协作式抢占（兼容旧行为） |
| `preemptStop` | GC STW |
| `preemptGcStop` | GC STW |
| `preemptPark` | 长时间运行的 G |
| `preemptSyscall` | 长时间 syscall |

## 7. sysmon 系统监控线程

```go
func sysmon() {
    // 不属于任何 P，不可被调度器调度走
    // 每 10us ~ 10ms 唤醒一次
    for {
        // 1. 抢占检查（>10ms 未让出 → 强制抢占）
        retake(now)

        // 2. 网络轮询
        lastpoll := sched.lastpoll
        if now()-lastpoll >= 10*1000*1000 {
            sched.lastpoll = now
            netpoll(0)   // ★ 唤醒 G 以接收数据
        }

        // 3. 回收 syscall 时间过长的 P
        // 4. timer 任务处理
        // 5. 时间调节
    }
}
```

**sysmon 三大职责**：
1. **retake**：让长时间运行/阻塞的 P 让出资源
2. **netpoll**：将就绪的网络 G 放入运行队列
3. **reclaim**：回收闲置 5 分钟以上的 M

## 8. Netpoller（网络轮询）

```
                ┌─────────────┐
   用户 G       │  netpoll()  │
   waitnet      │ (epoll/kqueue/iocp)│
     │          └──────┬──────┘
     ▼                 │ 唤醒就绪 fd
     P.runq  ◄─────────┘
     │
     ▼
   goroutine 继续执行 Read/Write
```

- Linux：epoll
- macOS：kqueue
- Windows：IOCP

**调度器触发时机**：
1. `findrunnable` 在所有其他方法都没有 G 时调用
2. sysmon 每 10ms 调用一次
3. GC Mark 阶段辅助调用

## 9. 调度生命周期示例

### 9.1 一个 G 的诞生到结束

```
1. go func(){}        →  newproc1() → gfree 队列取空 g → 初始化栈 → 放入 P.runnext

2. schedule() 循环    →  看到 runnext → execute()

3. _g_.m.curg = gp    →  gogo(&gp.sched) → 切到用户栈

4. 用户代码执行       →  可能进入：
                        - syscall（_Gsyscall）
                        - channel 阻塞（_Gwaiting）
                        - GC 抢占

5. gosched()/gopark() →  保存上下文到 gp.sched → 切回 g0 → schedule()

6. gp 完成            →  goexit() → 状态置 _Gdead → 栈归还 gfree
```

### 9.2 channel 通信的调度流程

```go
ch <- v  // 发送

// 1. 找到等待的 receiver → 直接传递（不阻塞）
// 2. 没有 receiver → 当前 g 状态置 _Gwaiting
// 3. sudog 加入 ch.sendq
// 4. gopark 释放 P，让其他 g 执行
// 5. 被接收时 → goready 恢复 _Grunnable，加入 P.runq
```

## 10. 调优与诊断工具

### 10.1 GODEBUG 参数

| 参数 | 含义 |
|------|------|
| `GODEBUG=gctrace=1` | 打印 GC 信息 |
| `GODEBUG=schedtrace=1000` | 每 1s 打印调度器状态 |
| `GODEBUG=scheddetail=1` | 详细调度器信息 |

```bash
SCHED 0ms: gomaxprocs=8 idleprocs=7 threads=5 spinningthreads=1 idlethreads=2 runqueue=0...
```

### 10.2 runtime metrics

```go
// runtime/metrics 包
metrics.Read(metrics.All) // 标准化的指标读取

// 关键指标
"sched/goroutines:goroutines"  // 当前 G 总数
"sched/latencies:sched-pauses" // 调度停顿延迟
"sched/gomaxprocs:threads"     // 线程数
```

### 10.3 pprof

```go
import _ "net/http/pprof"
go func() { http.ListenAndServe(":6060", nil) }()

// 查看 goroutine 调度
// go tool pprof http://localhost:6060/debug/pprof/goroutine
// go tool trace  http://localhost:6060/debug/pprof/trace
```

### 10.4 调优建议

| 场景 | 建议 |
|------|------|
| CPU 密集 | GOMAXPROCS = CPU 核数（默认） |
| 高并发网络 | GOMAXPROCS 适当调大（但不宜超过核数 2 倍） |
| 大量 goroutine 睡眠 | 调大 GOMAXPROCS 提升并行调度 |
| CPU 飙升 | 检查 sysmon 日志，可能存在忙等 G |
| 内存占用过高 | 检查 gfree 队列，可能是 goroutine 泄漏 |

## 11. 面试高频深度问题

| 问题 | 核心答案 |
|------|----------|
| 为什么引入 P？ | 解决 GM 模型下 G 与全局队列锁竞争严重的问题，P 作为本地队列减少锁 |
| Work stealing 偷一半的设计 | 平衡负载，队尾偷保持原队列热数据命中率 |
| sysmon 是 goroutine 吗？ | 不是，是单独的 OS 线程 m0.g0，不属于任何 P，不可被调度 |
| 为什么需要 g0？ | 调度器自身不能运行在用户 goroutine 栈上，否则切换开销巨大 |
| 抢占信号为什么用 SIGURG？ | 默认未被应用占用，Go 可独占使用；不会与用户信号冲突 |
| goroutine 切换开销多少？ | ~200ns（含调度器执行 + 上下文切换） |
| GOMAXPROCS 调大一定好吗？ | 不一定，超出 CPU 核数会增加调度器开销、降低 CPU 缓存命中率 |
| Netpoller 线程是谁？ | 由 runtime 在内部启动的 OS 线程，单独负责 epoll/kqueue |
| M 上限是多少？ | 10000（`sched.maxmcount`），M 上限与 P 数无关 |
| goroutine 泄漏怎么排查？ | pprof goroutine profile、runtime.NumGoroutine 监控、channel close 检测 |