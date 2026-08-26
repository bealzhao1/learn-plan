# Go 语言核心

## 1.1 语言特性

| 特性 | 说明 |
|------|------|
| 静态类型 | 编译期类型检查，减少运行时错误 |
| 垃圾回收 | 三色标记法 + 混合写屏障，STW 时间极短 |
| 原生并发 | goroutine + channel 是一等公民 |
| 接口隐式实现 | duck typing 风格，解耦更彻底 |
| 值语义 | struct 默认值拷贝，配合指针可控制分配 |
| 快速编译 | 秒级编译大型项目 |

## 1.2 核心数据结构

```go
// slice 底层结构
type slice struct {
    array unsafe.Pointer // 指向底层数组
    len   int            // 当前长度
    cap   int            // 容量
}

// map 底层结构 (简化)
type hmap struct {
    count     int            // 元素个数
    B         uint8          // buckets 数量 = 2^B
    buckets   unsafe.Pointer // 桶数组
    oldbuckets unsafe.Pointer // 扩容时旧桶
}

// string 底层结构
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```

## 1.3 内存管理

- **逃逸分析**：编译器决定变量分配在栈还是堆（`go build -gcflags="-m"`）
- **内存对齐**：struct 字段按对齐规则排列，影响内存占用
- **GC 策略**：三色标记 + 混合写屏障，目标 STW < 1ms
