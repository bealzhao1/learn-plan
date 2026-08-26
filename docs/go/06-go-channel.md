# Channel 深入解析

## 6.1 底层结构

```go
type hchan struct {
    qcount   uint           // 当前队列中的元素数
    dataqsiz uint           // 环形缓冲区大小
    buf      unsafe.Pointer // 环形缓冲区指针
    elemsize uint16         // 元素大小
    closed   uint32         // 是否已关闭
    sendx    uint           // 发送索引
    recvx    uint           // 接收索引
    recvq    waitq          // 等待接收的 goroutine 队列
    sendq    waitq          // 等待发送的 goroutine 队列
    lock     mutex          // 互斥锁
}
```

## 6.2 工作机制

**无缓冲 Channel（同步）**
```
Sender goroutine          Receiver goroutine
      │                          │
      ├── ch <- value ──────────►├── value = <-ch
      │   (阻塞等待接收者)         │   (阻塞等待发送者)
      │                          │
      └── 直接拷贝到接收者栈 ──────┘
```

**有缓冲 Channel（异步）**
```
   ┌───┬───┬───┬───┬───┐
   │ A │ B │ C │   │   │  环形缓冲区 (cap=5, len=3)
   └───┴───┴───┴───┴───┘
     ↑               ↑
   recvx           sendx
```

## 6.3 使用模式

```go
// 1. 信号通知
done := make(chan struct{})
go func() {
    // work...
    close(done) // 广播完成信号
}()
<-done

// 2. 超时控制
select {
case result := <-ch:
    process(result)
case <-time.After(3 * time.Second):
    fmt.Println("timeout")
}

// 3. 限流器 (令牌桶)
limiter := make(chan struct{}, 10) // 最多10个并发
for req := range requests {
    limiter <- struct{}{} // 获取令牌
    go func(r Request) {
        defer func() { <-limiter }() // 释放令牌
        handle(r)
    }(req)
}

// 4. 生产者-消费者
func producer(ch chan<- int) {
    for i := 0; ; i++ {
        ch <- i
    }
}
func consumer(ch <-chan int) {
    for v := range ch {
        process(v)
    }
}

// 5. select 多路复用
select {
case msg := <-msgCh:
    handle(msg)
case err := <-errCh:
    handleErr(err)
case <-ctx.Done():
    return ctx.Err()
}
```

## 6.4 Channel 公理

| 操作 | nil channel | closed channel | 正常 channel |
|------|-------------|---------------|-------------|
| 发送 | 永久阻塞 | **panic** | 阻塞或成功 |
| 接收 | 永久阻塞 | 返回零值 | 阻塞或成功 |
| 关闭 | **panic** | **panic** | 正常关闭 |
