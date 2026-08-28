# React 核心概念与渲染机制

## 21.1 为什么需要 React

传统命令式 DOM 操作（jQuery）的痛点：开发者手动管理「数据 → UI」的同步，状态分散在 DOM 里，复杂交互下极易失控。

React 的核心思想是**声明式 + 组件化**：你只描述「UI 应该长什么样」，React 负责把状态变化渲染成正确的 DOM。

```
命令式: 你告诉 DOM 每一步怎么做 (先找到节点，再改 class，再插子节点...)
声明式: 你告诉 React 目标状态是什么 (count=5)，React 自动算出 DOM 该变成什么样
```

## 21.2 JSX 的原理

JSX 不是模板字符串，也不是 HTML，它是 `React.createElement` 的**语法糖**，由 Babel/SWC 在编译期转换。

```jsx
// 你写的 JSX
const el = <div id="app" className="box">hello</div>;

// 编译后（React 17 之前）
const el = React.createElement('div', { id: 'app', className: 'box' }, 'hello');

// React 17+ 的自动运行时（jsx-runtime）
import { jsx as _jsx } from 'react/jsx-runtime';
const el = _jsx('div', { id: 'app', className: 'box', children: 'hello' });
```

要点：

- 大写的 `<MyComp/>` 是组件引用，小写的 `<div>` 是宿主元素
- `className` 而非 `class`（class 是 JS 保留字）
- 表达式用 `{}` 包裹，但 `{}` 里不能写语句（只能写表达式）

## 21.3 虚拟 DOM（Virtual DOM）

虚拟 DOM 是**内存中的 JS 对象树**，描述真实 DOM 结构：

```js
// 一个 React 元素本质就是这样一个对象
{
  type: 'div',
  props: { className: 'box', children: [{ type: 'span', props: { children: 'hi' } }] },
  key: null,
  ref: null,
  $$typeof: Symbol(react.element),
}
```

**为什么需要它？**

| 方案 | 问题 |
|------|------|
| 直接操作真实 DOM | 频繁读写真实 DOM 代价高（触发重排/重绘），且状态与视图易不同步 |
| 虚拟 DOM | 先在内存里 diff 出最小变更，再一次性批量更新真实 DOM |

> 注意：虚拟 DOM 不是为了「比直接操作 DOM 更快」，而是为了**开发体验（声明式）**和**可预测性**，同时把 DOM 操作收敛为 diff 后的最小 patch。真正快的场景（如高频动画）仍需手动优化。

## 21.4 Fiber 架构

React 15 用**递归的 Stack Reconciler**，一旦开始 diff 就无法中断，长任务会阻塞主线程导致掉帧。React 16 引入 **Fiber** 解决这个问题。

**Fiber 是什么**：既是一种**数据结构**（可中断的单元），也是一个**调度架构**。

Fiber 节点本质上是一个**链表节点**，把组件树展开成链表：

```js
// Fiber 节点关键字段（简化）
{
  tag: 2,                 // 节点类型 (FunctionComponent / HostComponent / ...)
  type: 'div',            // 对应的元素类型
  key: null,
  stateNode: domNode,     // 对应的真实 DOM
  return: parentFiber,    // 父节点
  child: firstChild,      // 第一个子节点
  sibling: nextSibling,   // 下一个兄弟节点
  alternate: oldFiber,    // 指向另一棵树上的对应节点（双缓存）
  flags: 0,               // 副作用标记（插入/更新/删除）
  lanes: 0,               // 优先级
}
```

**双缓存（Double Buffering）**：

```
current 树（屏幕上正在显示）     workInProgress 树（内存中构建）
        │                                  │
        └──────── alternate 互指 ──────────┘
```

- 每次更新在内存中构建一棵新的 `workInProgress` 树
- 构建完成后一次性切换指针（`current` 指向新树）
- 好处：渲染过程可中断、可丢弃、可复用旧节点

## 21.5 Reconciliation（协调 / Diff 算法）

协调就是「新旧两棵 Fiber 树求最小差异」的过程。为了把 O(n³) 的通用树 diff 降到 O(n)，React 采用三条**启发式策略**：

**1. 同层比较，不跨层移动**：只比较同一层级的节点，跨层则直接销毁重建。

**2. 类型不同直接替换**：`<div>` 变 `<span>`，整棵子树卸载重建，不再深入比较。

**3. 用 key 标识同层子节点**：列表 diff 靠 key 判断哪些节点可复用、可移动。

```jsx
// 没有 key：React 只能按索引比较，顺序变化会导致错误复用（状态错乱）
<ul>{items.map(i => <li>{i.text}</li>)}</ul>

// 有 key：React 能准确识别「移动 vs 新增 vs 删除」
<ul>{items.map(i => <li key={i.id}>{i.text}</li>)}</ul>
```

> **不要用数组下标 index 当 key**：当列表发生插入/删除/排序时，index 会让 React 错把「复用」当成「更新」，造成输入框内容串位等 bug。

## 21.6 渲染流程：Render 与 Commit

React 把一次更新分为两个阶段：

```
┌─────────────────────────────────────────────────────┐
│ Render 阶段（可中断、可恢复）                         │
│  计算状态变化 → 生成新的 workInProgress Fiber 树      │
│  标记副作用 flags（Placement / Update / Deletion）    │
│  由 Scheduler 按优先级分片执行（时间切片）             │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│ Commit 阶段（不可中断、同步执行）                      │
│  before mutation → mutation → layout                  │
│  把 flags 对应的变更一次性应用到真实 DOM              │
│  执行 useEffect / useLayoutEffect 等副作用             │
└─────────────────────────────────────────────────────┘
```

**Render 阶段可中断**是 Fiber 的核心价值：Scheduler 给每个工作单元分配 5ms 左右的时间片，时间用完就把执行权交还浏览器（处理输入/动画），保证不卡顿。这也是为什么 Render 阶段**不能有副作用**。

## 21.7 调度与优先级（Lane 模型）

并发更新按紧急程度分配不同优先级的 lane：

```
高优先级 lane（用户输入、点击、拖拽）── 可打断低优先级
低优先级 lane（网络数据、大数据渲染）── 可被抢占
```

- 高优先级更新可以**打断**正在进行的低优先级渲染（低优先级渲染被丢弃后重启）
- `startTransition` 可显式把更新标记为「非紧急过渡更新」
- `useDeferredValue` 让非紧急的值延迟更新

## 21.8 面试高频题

1. **虚拟 DOM 一定比真实 DOM 快吗？** 不一定。它胜在开发体验与可预测性，diff + patch 本身有开销；高频精细操作仍需直接操作 DOM。

2. **Fiber 解决了什么问题？** 让渲染可中断、可调度，避免长任务阻塞主线程；通过时间切片支持并发渲染。

3. **为什么列表要 key，key 用 index 有什么问题？** key 是同层 diff 的身份标识；index 在增删排序时会导致节点错位复用、状态串位。

4. **React 一次更新里 Render 和 Commit 能被打断吗？** Render 可中断，Commit 不可中断（否则 DOM 会处于中间态）。

5. **`$$typeof` 字段是干嘛的？** 用来防 XSS 注入——服务端渲染时恶意 JSON 里塞不进带 `Symbol` 的元素，React 借此识别「这到底是不是合法的 React 元素」。
