# Gin Web 框架设计原理

## 7.1 架构概览

```
HTTP Request
    │
    ▼
┌──────────────┐
│   Engine     │  (核心引擎，实现 http.Handler 接口)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  RouterGroup │  (路由分组，支持中间件)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Radix Tree  │  (前缀树路由匹配)
└──────┬───────┘
       │
       ▼
┌──────────────────────┐
│  Middleware Chain     │  (中间件链 → HandlersChain)
│  [Logger][Auth][...]  │
└──────┬───────────────┘
       │
       ▼
┌──────────────┐
│   Handler    │  (业务处理函数)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Context     │  (请求上下文，贯穿整个请求生命周期)
└──────────────┘
```

## 7.2 核心设计

**1. Radix Tree（基数树 / 压缩前缀树）**

```
路由注册: GET /api/users/:id
         GET /api/users/:id/posts
         GET /api/products

树结构:
            /api/
           /     \
      users/    products [handler]
        |
      :id [handler]
        |
      /posts [handler]
```

- 比 hash map 更节省内存（共享前缀）
- 支持路径参数 `:param` 和通配符 `*path`
- 每个 HTTP 方法独立一棵树

**2. Context 池化 (sync.Pool)**

```go
// Engine 内部
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    c := engine.pool.Get().(*Context) // 从池中获取
    c.writermem.reset(w)
    c.Request = req
    c.reset()
    
    engine.handleHTTPRequest(c)
    
    engine.pool.Put(c) // 归还到池中
}
```

- 避免每次请求 `new(Context)`，减少 GC 压力
- 性能提升关键之一

**3. 中间件链式调用**

```go
type HandlerFunc func(*Context)
type HandlersChain []HandlerFunc

// Context.Next() 实现洋葱模型
func (c *Context) Next() {
    c.index++
    for c.index < int8(len(c.handlers)) {
        c.handlers[c.index](c)
        c.index++
    }
}

// Context.Abort() 终止链
func (c *Context) Abort() {
    c.index = abortIndex // 63
}
```

**4. 路由分组与中间件继承**

```go
r := gin.New()
r.Use(Logger(), Recovery()) // 全局中间件

api := r.Group("/api")
api.Use(AuthMiddleware()) // 组级中间件
{
    api.GET("/users", listUsers)    // 继承全局 + 组中间件
    api.POST("/users", createUser)
}
```

## 7.3 性能优化点

| 优化 | 说明 |
|------|------|
| sync.Pool | Context 复用，减少 GC |
| Radix Tree | O(k) 路由查找（k=路径长度）|
| 预分配 slice | HandlersChain 预分配容量 |
| 零拷贝 | 路径参数直接引用请求路径字符串 |
| 避免反射 | 绑定参数用 tag 解析而非大量反射 |
