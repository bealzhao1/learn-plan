# Go 打包工具

## 3.1 Go Modules（官方标准）

```bash
# 初始化模块
go mod init github.com/user/project

# 添加依赖
go get github.com/gin-gonic/gin@latest

# 整理依赖
go mod tidy

# 下载依赖到本地缓存
go mod download

# 生成 vendor 目录
go mod vendor
```

**核心文件**：
- `go.mod`：声明模块路径和依赖版本
- `go.sum`：依赖的加密哈希校验

## 3.2 构建与交叉编译

```bash
# 标准构建
go build -o app ./cmd/server

# 交叉编译
GOOS=linux GOARCH=amd64 go build -o app-linux

# 减小体积
go build -ldflags="-s -w" -o app

# 静态编译（无 CGO 依赖）
CGO_ENABLED=0 go build -o app
```

## 3.3 常用工具链

| 工具 | 用途 |
|------|------|
| `go build` | 编译 |
| `go test` | 测试 |
| `go vet` | 静态分析 |
| `golangci-lint` | 综合 lint 工具 |
| `go generate` | 代码生成 |
| `go tool pprof` | 性能分析 |
| `go tool trace` | 追踪分析 |
| `goreleaser` | 跨平台发布 |
| `ko` | 容器化构建（无 Dockerfile） |
