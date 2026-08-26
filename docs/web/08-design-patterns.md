# 设计模式的优点

## 8.1 创建型模式

| 模式 | 优点 | Go 典型应用 |
|------|------|------------|
| **单例** | 全局唯一实例，节省资源 | `sync.Once` + 包级变量 |
| **工厂方法** | 解耦创建逻辑，扩展不修改 | `New()` 构造函数惯例 |
| **抽象工厂** | 创建产品族，保证兼容性 | 数据库 driver 注册 |
| **建造者** | 分步构建复杂对象，链式调用清晰 | `strings.Builder`、选项模式 |
| **原型** | 避免重复初始化，克隆高效 | `proto.Clone()` |

## 8.2 结构型模式

| 模式 | 优点 | Go 典型应用 |
|------|------|------------|
| **适配器** | 兼容不兼容接口，不改原代码 | `io.Reader` 适配各种输入 |
| **装饰器** | 动态增强功能，不修改原始逻辑 | `http.HandlerFunc` 中间件 |
| **代理** | 控制访问，延迟初始化，缓存 | RPC client stub |
| **组合** | 统一处理树形结构 | 文件系统、UI 组件 |
| **外观** | 简化复杂子系统接口 | SDK 封装 |

## 8.3 行为型模式

| 模式 | 优点 | Go 典型应用 |
|------|------|------------|
| **策略** | 算法可替换，符合开闭原则 | `sort.Interface` |
| **观察者** | 松耦合事件通知 | channel 广播 |
| **责任链** | 请求处理者可灵活组合 | Gin 中间件链 |
| **模板方法** | 固定骨架，子步骤可变 | `io.Copy` 模板 |
| **迭代器** | 统一遍历接口 | `range` / `iter` 包 |

## 8.4 Go 特色模式

```go
// Functional Options 模式
type Server struct {
    port    int
    timeout time.Duration
}
type Option func(*Server)

func WithPort(p int) Option     { return func(s *Server) { s.port = p } }
func WithTimeout(t time.Duration) Option { return func(s *Server) { s.timeout = t } }

func NewServer(opts ...Option) *Server {
    s := &Server{port: 8080, timeout: 30 * time.Second}
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// 使用
srv := NewServer(WithPort(9090), WithTimeout(60*time.Second))
```

## 8.5 设计模式核心价值

1. **可维护性** — 代码结构清晰，新人可快速理解
2. **可扩展性** — 新增功能不修改已有代码（开闭原则）
3. **可复用性** — 通用方案，减少重复造轮子
4. **松耦合** — 模块间依赖最小化
5. **可测试性** — 依赖注入、接口隔离使单测更容易
