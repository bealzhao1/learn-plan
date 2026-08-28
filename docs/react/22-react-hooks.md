# React Hooks 深入

## 22.1 为什么要有 Hooks

类组件的问题：逻辑被 `componentDidMount` / `componentDidUpdate` / `componentWillUnmount` 生命周期割裂，状态逻辑难以复用（HOC 和 render props 导致嵌套地狱）。

Hooks 让**函数组件**拥有状态和副作用能力，且让**同一逻辑内聚在一起**、可复用。

## 22.2 Hook 的底层存储：链表

组件内的每个 Hook 按调用顺序挂在组件 Fiber 的 `memoizedState` 链表上：

```
fiber.memoizedState
    │
    ▼
[useState hook] → next → [useEffect hook] → next → [useState hook] → ...
   (state=0)              (effect 对象)            (state='a')
```

**这就是「Hook 规则」的由来**：

- **只能在顶层调用 Hook**：因为 React 靠「调用顺序」匹配 hook 与状态，放进条件/循环会让顺序错乱
- **只能在函数组件或自定义 Hook 里调用**：hook 需要挂在组件 Fiber 上

## 22.3 useState 的原理

```jsx
const [count, setCount] = useState(0);
```

- `useState(0)`：首次渲染把初始值存入 hook 节点；后续渲染直接读回
- **惰性初始化**：`useState(() => expensiveComputation())` 只在首次执行，避免每次渲染重算
- `setCount` 触发更新：值变了 → 标记更新 → 重新渲染
- **批处理（Batching）**：同一事件里的多次 `setState` 会合并成一次渲染（React 18 起对异步回调、Promise、setTimeout 也自动批处理）

```jsx
// 连续调用，只触发一次重渲染，count 只加 1（都基于旧的 0）
setCount(count + 1);
setCount(count + 1);
setCount(count + 1);

// 想连续累加，用函数式更新
setCount(c => c + 1);
setCount(c => c + 1);
setCount(c => c + 1);
```

## 22.4 useEffect 与 useLayoutEffect

```jsx
useEffect(() => {
  // 副作用逻辑
  return () => { /* 清理函数 */ };
}, [deps]);
```

| | useEffect | useLayoutEffect |
|---|---|---|
| 执行时机 | Commit 后**异步**执行（浏览器绘制之后） | DOM 变更后、**浏览器绘制之前**同步执行 |
| 会阻塞渲染吗 | 不阻塞 | 会阻塞 |
| 典型场景 | 数据请求、订阅、日志、手动改 DOM 建议用它 | 读取布局、同步改 DOM 避免闪烁 |

**依赖数组**：`deps` 变化才重新执行；空数组 `[]` 只在挂载/卸载执行一次。

## 22.5 useMemo 与 useCallback

```jsx
const memoizedValue = useMemo(() => expensive(a, b), [a, b]);
const memoizedFn   = useCallback(() => doSomething(a), [a]);
```

- `useCallback(fn, deps)` 本质是 `useMemo(() => fn, deps)`，缓存的是**函数引用**
- **用途**：配合 `React.memo` 避免子组件因「函数引用变化」而无效重渲染
- **误区**：不要到处滥用——缓存本身有开销，只有「真正昂贵的计算」或「引用稳定性影响子组件」时才值得用

```jsx
// 典型正确用法：child 用 React.memo 包裹，传稳定引用避免无谓重渲染
const handleClick = useCallback(() => setCount(c => c + 1), []);
<Child onClick={handleClick} />  // Child 用 React.memo
```

## 22.6 useRef

```jsx
const ref = useRef(0);
ref.current = 5; // 修改不会触发重新渲染
```

- 保存**可变值**，跨渲染保持同一引用，且改它不触发重渲染
- 常用于：DOM 引用、保存定时器 id、保存「上一次的值」、打破闭包陈旧值

## 22.7 闭包陷阱（Stale Closure）

函数组件每次渲染都会「重新执行」，本次渲染里捕获的是**本次渲染时的 props/state**：

```jsx
useEffect(() => {
  const timer = setInterval(() => {
    console.log(count); // 永远打印 0！
  }, 1000);
  return () => clearInterval(timer);
}, []); // 空依赖，闭包捕获的是首次渲染的 count=0
```

**解法**：

```jsx
// 方案 1：函数式更新，不依赖外部 count
setInterval(() => setCount(c => c + 1), 1000);

// 方案 2：用 ref 始终指向最新值
const countRef = useRef(count);
countRef.current = count;
setInterval(() => console.log(countRef.current), 1000);

// 方案 3：把依赖写进 deps，每次重建定时器
useEffect(() => { ... }, [count]);
```

## 22.8 其他常用 Hook

- **useReducer**：复杂状态逻辑用 reducer 表达，`dispatch` 引用稳定
- **useContext**：跨层级取共享值（见状态管理篇）
- **useImperativeHandle**：自定义暴露给父组件的 ref 实例
- **useDeferredValue / useTransition**：标记非紧急更新

## 22.9 自定义 Hook

以 `use` 开头、内部调用其他 Hook 的普通函数，用于**复用有状态逻辑**：

```jsx
function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => localStorage.getItem(key) ?? initial);
  useEffect(() => localStorage.setItem(key, value), [key, value]);
  return [value, setValue];
}
// 使用：const [name, setName] = useLocalStorage('name', '');
```

## 22.10 面试高频题

1. **为什么 Hook 不能放在 if 里？** React 靠调用顺序把 hook 与状态对应，条件调用会让顺序错乱、状态错位。

2. **useEffect 和 useLayoutEffect 区别？** 前者绘制后异步执行、不阻塞；后者绘制前同步执行、会阻塞，用于读布局避免闪烁。

3. **useCallback 和 useMemo 区别？** useCallback 缓存函数引用，useMemo 缓存计算结果；useCallback 是 useMemo 的特例。

4. **useEffect 的清理函数何时执行？** 依赖变化或组件卸载时，在下一次 effect 执行前调用（先清理旧的后执行新的）。

5. **为什么有时 state 打印的是旧值？** 闭包捕获了本次渲染的旧 state；用函数式更新或 ref 解决。
