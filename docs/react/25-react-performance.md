# React 性能优化

## 25.1 先定位，再优化

优化的第一原则是**测量**——不要凭感觉优化。用 React DevTools 的 **Profiler** 和浏览器 Performance 面板定位「哪个组件重渲染最多、哪次渲染最慢」。

```
优化流程：测量 → 定位瓶颈 → 针对性优化 → 再测量验证
```

## 25.2 重渲染的触发与避免

**组件重渲染的三类原因**：

1. 自身 state 变化
2. 父组件重渲染（子组件默认跟着重渲染）
3. 消费的 Context value 变化

**避免手段**：

```jsx
// 1. React.memo：props 浅比较不变则跳过渲染
const Child = React.memo(function Child({ count }) {
  return <div>{count}</div>;
});

// 2. 传稳定引用，配合 memo 才有效
const onClick = useCallback(() => {}, []);
<Child count={count} onClick={onClick} />

// 3. 状态下沉：把频繁变化的状态放到最小作用域组件
```

**核心心智**：`React.memo` 必须配合 `useMemo`/`useCallback` 的「稳定引用」才有意义，否则父组件每次渲染生成新对象/新函数，memo 形同虚设。

## 25.3 列表渲染

```jsx
// 用稳定且唯一的 key（不要用 index）
{list.map(item => <Row key={item.id} item={item} />)}

// 长列表用虚拟列表，只渲染可视区域
import { FixedSizeList } from 'react-window';
<FixedSizeList height={600} width="100%" itemCount={10000} itemSize={35}>
  {Row}
</FixedSizeList>
```

- **虚拟列表**：万级数据只渲染视口内几十条，大幅降低 DOM 节点数
- **分页 / 无限滚动**：配合 TanStack Query 的 `useInfiniteQuery`

## 25.4 代码分割与懒加载

```jsx
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./Dashboard')); // 单独打包，按需加载

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Dashboard />
    </Suspense>
  );
}
```

- `React.lazy` + 动态 `import()` 实现**路由级/组件级分包**
- 结合路由懒加载，首屏只加载当前页所需代码
- Webpack/Vite 会为动态 import 自动拆 chunk

## 25.5 性能陷阱清单

| 陷阱 | 后果 | 解法 |
|------|------|------|
| 内联对象/函数传给 memo 组件 | memo 失效，每次都重渲染 | useMemo/useCallback |
| 列表用 index 当 key | 增删时错位、状态串位 | 稳定唯一 key |
| Context 塞高频变化值 | 全消费树频繁重渲染 | 拆分 context 或换方案 |
| 组件内部定义子组件 | 每次渲染子组件身份变化导致 remount | 子组件提到外部定义 |
| 滥用 useMemo/useCallback | 缓存开销反而拖慢 | 只在必要时用 |
| 未懒加载大页面 | 首屏包体过大 | React.lazy 分包 |

## 25.6 首屏性能（与 React 相关部分）

- **包体**：tree-shaking、按需引入（如 `lodash-es`）、代码分割
- **渲染**：SSR/SSG（详见 SSR 与 Next.js 篇）、减少首屏同步请求
- **图片**：懒加载、响应式 `srcset`、现代格式（WebP/AVIF）
- **字体**：`font-display: swap` 避免阻塞

## 25.7 面试高频题

1. **React.memo 一定能提升性能吗？** 不一定。浅比较本身有开销，props 频繁变化时反而更慢；需配合稳定引用且组件渲染昂贵才有效。

2. **useMemo 和 useCallback 是不是越多越好？** 不是。缓存有内存和比较开销，滥用会负优化，先测量再决定。

3. **长列表怎么优化？** 虚拟列表（只渲染可视区）+ 稳定 key + 分页/无限滚动。

4. **如何减少首屏体积？** 路由懒加载、代码分割、tree-shaking、按需引入、图片懒加载。

5. **React.lazy 的原理？** 配合动态 import，返回一个在 resolve 后渲染目标组件的「懒组件」，加载期间用 Suspense fallback。
