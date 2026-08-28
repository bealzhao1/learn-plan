# React 路由与数据请求

## 24.1 React Router

React Router 是事实标准的路由库（v6+ 为主流）。核心是**声明式路由 + 嵌套 + 数据无关**。

```jsx
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/users" element={<Users />}>
          <Route path=":id" element={<UserDetail />} />  {/* 嵌套路由 */}
        </Route>
        <Route path="*" element={<NotFound />} />        {/* 兜底 */}
      </Routes>
    </BrowserRouter>
  );
}
```

**两种路由模式**：

| 模式 | 地址形式 | 说明 |
|------|----------|------|
| BrowserRouter | `/users/1` | 走 History API，需要服务端支持 fallback 到 index.html |
| HashRouter | `/#/users/1` | 走 location.hash，无需服务端配置，刷新不 404 |

**常用 Hook**：

```jsx
const params = useParams();       // 取路径参数 :id
const [searchParams] = useSearchParams(); // 取查询参数 ?a=1
const navigate = useNavigate();   // 编程式跳转 navigate('/users')
const location = useLocation();   // 当前 location 对象
```

## 24.2 数据请求基础

**fetch / axios 模式**：

```jsx
useEffect(() => {
  let ignore = false; // 竞态处理：组件卸载后不 setState
  fetch(`/api/users/${id}`)
    .then(r => r.json())
    .then(data => { if (!ignore) setUser(data); });
  return () => { ignore = true; };
}, [id]);
```

手写请求的常见坑：

- **竞态（race condition）**：快速切换 id，慢请求可能覆盖快请求 → 需 ignore 标记或 AbortController
- **重复请求**：组件反复挂载/卸载触发重复拉取
- **缓存缺失**：每次进入页面都重新请求，无缓存、无去重
- **状态管理繁琐**：isLoading/isError/data 手动维护

## 24.3 TanStack Query（React Query）

TanStack Query 用**声明式**方式把上述问题一揽子解决：

```jsx
import { useQuery, useMutation, QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient();

function UserList() {
  const { data, isLoading, isError, error } = useQuery({
    queryKey: ['users'],                                  // 缓存身份
    queryFn: () => fetch('/api/users').then(r => r.json()),
    staleTime: 30_000,                                    // 30s 内新鲜，不重新请求
  });

  if (isLoading) return <div>loading...</div>;
  if (isError) return <div>{error.message}</div>;
  return data.map(u => <div key={u.id}>{u.name}</div>);
}
```

**核心概念**：

| 概念 | 说明 |
|------|------|
| queryKey | 缓存 key，数组形式；变化则重新请求，相同则命中缓存 |
| staleTime | 数据多久视为「新鲜」，新鲜期内不重新请求 |
| cacheTime | 数据在缓存里保留多久（无组件使用后） |
| refetchOnWindowFocus | 窗口聚焦时是否后台刷新（默认 true） |

**Mutation（写操作）+ 乐观更新**：

```jsx
const mutation = useMutation({
  mutationFn: newTodo => fetch('/api/todos', { method: 'POST', body: JSON.stringify(newTodo) }),
  onMutate: async newTodo => {          // 乐观更新：先改 UI
    await queryClient.cancelQueries(['todos']);
    const prev = queryClient.getQueryData(['todos']);
    queryClient.setQueryData(['todos'], old => [...old, newTodo]);
    return { prev };
  },
  onError: (_err, _v, ctx) => queryClient.setQueryData(['todos'], ctx.prev), // 失败回滚
  onSettled: () => queryClient.invalidateQueries(['todos']),                  // 最终重取
});
```

## 24.4 SWR 与 TanStack Query 对比

| | SWR | TanStack Query |
|---|---|---|
| 作者/维护 | Vercel | TanStack 社区 |
| 核心思想 | stale-while-revalidate | 更完整的缓存/失效模型 |
| 能力 | 简洁、聚焦数据获取 | 功能更全（分页、无限滚动、乐观更新） |
| 适用 | 轻量场景 | 复杂数据场景 |

## 24.5 路由级数据获取最佳实践

1. **用 queryKey 把路由参数带进去**：`queryKey: ['user', id]`，id 变化自动重新请求，天然解决竞态
2. **预取（prefetch）**：hover 链接时 `queryClient.prefetchQuery`，点击即达
3. **路由懒加载**：`React.lazy(() => import('./Page'))` + `<Suspense>` 分包
4. **服务端状态交给 Query，客户端状态交给 useState/Zustand**，边界清晰

## 24.6 面试高频题

1. **BrowserRouter 和 HashRouter 区别？** 前者走 History API、URL 干净但需服务端 fallback；后者走 hash、无需配置但 URL 带 `#`。

2. **如何避免请求竞态？** 用 ignore 标记 / AbortController / 或直接用 Query 的 queryKey 驱动。

3. **staleTime 和 cacheTime 区别？** staleTime 决定「多久内不重新请求」，cacheTime 决定「无组件使用时数据在内存里保留多久」。

4. **乐观更新是什么？为什么需要？** 先乐观改 UI 再发请求，失败回滚；用于提升交互响应速度。

5. **React Router v6 相比 v5 的差异？** 引入 `<Routes>`/`<Route element>`、支持嵌套路由、移除 `<Switch>`、用 `useNavigate` 替代 `useHistory`。
