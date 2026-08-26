# Go 并发编程

## 5.1 并发原语

```go
// 1. sync.Mutex — 互斥锁
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()

// 2. sync.RWMutex — 读写锁
var rwmu sync.RWMutex
rwmu.RLock()   // 多读并发
rwmu.RUnlock()

// 3. sync.WaitGroup — 等待一组 goroutine
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    // work
}()
wg.Wait()

// 4. sync.Once — 只执行一次
var once sync.Once
once.Do(func() {
    // 初始化
})

// 5. sync.Pool — 对象池，减少 GC 压力
pool := sync.Pool{
    New: func() interface{} { return new(bytes.Buffer) },
}
buf := pool.Get().(*bytes.Buffer)
defer pool.Put(buf)

// 6. sync.Map — 并发安全的 map
var m sync.Map
m.Store("key", "value")
val, ok := m.Load("key")

// 7. context — 超时/取消传播
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
```

## 5.2 并发模式

**Fan-out / Fan-in**
```go
func fanOut(input <-chan int, workers int) []<-chan int {
    channels := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        channels[i] = worker(input)
    }
    return channels
}

func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

**Pipeline（管道）**
```go
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}
```

## 5.3 常见陷阱

| 陷阱 | 说明 | 解决方案 |
|------|------|---------|
| 数据竞争 | 多 goroutine 无锁读写共享变量 | `go run -race`、mutex、channel |
| goroutine 泄漏 | channel 无人消费导致阻塞 | context 取消、done channel |
| 闭包捕获循环变量 | `for i := range` 中 i 被共享 | 传参或局部变量 |
| sync 类型拷贝 | Mutex/WaitGroup 不可拷贝 | 传指针 |
