# React 状态管理

## 23.1 状态分类

```
┌────────────────────────────────────────────────────────────┐
│ 本地状态 (useState)          组件内部私有，如开关、表单输入    │
│ 共享/全局状态 (Context/Redux/Zustand)  跨组件共享，如用户信息  │
│ 服务端状态 (TanStack Query)  来自 API，需缓存/失效/重试         │
│ 路由状态 (URL)              反应在地址栏，可分享、可回退         │
└────────────────────────────────────────────────────────────┘
```

**核心原则**：先想清楚「这状态属于哪一类」，再选工具。不要什么都塞进全局 store。

## 23.2 Context API

```jsx
const ThemeCtx = createContext('light');

function App() {
  return (
    <ThemeCtx.Provider value="dark">
      <Toolbar />
    </ThemeCtx.Provider>
  );
}

function Toolbar() {
  const theme = useContext(ThemeCtx); // 直接取到 'dark'
  return <div>{theme}</div>;
}
```

**优点**：无第三方依赖、适合低频变更的全局值（主题、语言、用户信息）。

**局限**：

- **性能**：Provider 的 value 一变，所有消费它的组件都会重渲染（无法局部跳过）
- **不适合高频变更状态**：如每秒变化的输入值，会造成大面积重渲染

```jsx
// 优化：value 用 useMemo 缓存，避免无关重渲染时对象引用变化
const value = useMemo(() => ({ theme, user }), [theme, user]);
```

## 23.3 Redux

Redux 的哲学：**单一数据源 + 单向数据流**。

```
Action (描述发生了什么) → Dispatch → Reducer (纯函数，算新状态) → Store (更新) → View (重渲染)
```

```js
// Redux Toolkit（现代推荐写法）
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    incremented: s => { s.value += 1; }, // immer 让你"可变"写法，内部不可变
  },
});
```

**优点**：状态可预测、可追溯（DevTools 时间旅行）、生态成熟、适合大型团队协作。

**缺点**：样板代码多、心智负担重；小项目「杀鸡用牛刀」。

## 23.4 Zustand（轻量状态）

Zustand 是当前很流行的轻量方案，核心是**无 Provider、基于 hook 的细粒度订阅**：

```js
import { create } from 'zustand';

const useStore = create(set => ({
  count: 0,
  inc: () => set(s => ({ count: s.count + 1 })),
}));

function Counter() {
  const count = useStore(s => s.count); // 只订阅 count，其它变化不触发重渲染
  const inc = useStore(s => s.inc);
  return <button onClick={inc}>{count}</button>;
}
```

**优点**：API 极简、按需订阅（selector 精确到字段）、无 Provider 嵌套、支持中间件（persist、immer）。

## 23.5 方案对比与选型

| 方案 | 适用场景 | 学习成本 | 性能 |
|------|----------|----------|------|
| useState | 组件本地状态 | 低 | 高 |
| Context | 低频全局值（主题/用户） | 低 | 中（易整树重渲染） |
| Zustand | 中小项目全局状态 | 低 | 高（细粒度订阅） |
| Redux Toolkit | 大型项目、强可追溯 | 中高 | 高 |
| Recoil/Jotai | 原子化、细粒度派生 | 中 | 高 |

**选型建议**：

- 简单 → useState + Context 就够
- 中大型、要爽 → Zustand
- 大团队、要规范与 DevTools → Redux Toolkit

## 23.6 服务端状态：交给 TanStack Query

服务端状态（Server State）和客户端状态本质不同——它有**异步、缓存、失效、重试、去重**等特性，用 Redux/Zustand 手写缓存极易出错。

TanStack Query 把这些封装好：

```jsx
const { data, isLoading, isError, refetch } = useQuery({
  queryKey: ['todos'],           // 缓存 key
  queryFn: () => fetch('/api/todos').then(r => r.json()),
  staleTime: 5000,               // 5 秒内视为新鲜，不重新请求
});
```

要点（详细见路由与数据请求篇）：

- 用 `queryKey` 做缓存身份，自动去重、缓存、后台刷新
- 内置 loading/error/refetch 状态机，告别手写 `isLoading` 布尔值
- 支持乐观更新、分页、无限滚动、预取

## 23.7 面试高频题

1. **Context 的性能问题是什么？** Provider value 变化会使所有消费者重渲染，且难以按字段跳过，不适合高频状态。

2. **Redux 三大原则？** 单一数据源、状态只读（用 action 描述变化）、用纯函数 reducer 修改。

3. **Zustand 相比 Redux 的优势？** 无 Provider、样板少、selector 细粒度订阅避免无效重渲染。

4. **客户端状态和服务端状态为什么要分开管？** 服务端状态有缓存/失效/异步/重试等特性，专用工具（Query）更合适。

5. **为什么 Redux 要「不可变更新」？** 不可变引用才能做浅比较判断是否变化，也是时间旅行调试的基础。
