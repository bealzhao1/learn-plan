# React SSR 与 Next.js

## 26.1 为什么需要 SSR

纯客户端渲染（CSR）的问题：

```
CSR 流程：浏览器下载空 HTML → 下载并执行 JS → JS 请求数据 → 渲染页面
问题：首屏白屏久、SEO 抓不到内容（爬虫看不到 JS 渲染后的 DOM）
```

服务端渲染（SSR）在服务端就把首屏 HTML 渲染好返回，浏览器「秒见」内容，SEO 友好。

## 26.2 渲染模式对比

| 模式 | 渲染位置 | 首屏 | SEO | 适用 |
|------|----------|------|-----|------|
| CSR | 浏览器 | 慢（白屏） | 差 | 后台系统、登录后应用 |
| SSR | 服务端（每次请求） | 快 | 好 | 实时内容、个性化页 |
| SSG | 构建时 | 最快 | 最好 | 内容固定（博客、文档） |
| ISR | 构建时 + 定时/按需再生 | 快 | 好 | 更新不频繁的内容 |

```
SSG（静态生成）  ←→  ISR（增量再生成）  ←→  SSR（按需渲染）  ←→  CSR（客户端渲染）
  构建时生成         有更新时再生成          每次请求渲染          浏览器里渲染
```

## 26.3 Next.js 核心

Next.js 是目前 React 生态最主流的框架（App Router 为 v13+ 新范式）。

**目录约定（App Router）**：

```
app/
├── layout.tsx        # 根布局（共享导航/头部）
├── page.tsx          # 首页（对应 /）
├── blog/
│   ├── page.tsx      # /blog
│   └── [slug]/
│       └── page.tsx  # /blog/:slug 动态路由
└── api/
    └── hello/route.ts # 后端 API /api/hello
```

**服务端组件 vs 客户端组件**：

| | Server Component（默认） | Client Component（"use client"） |
|---|---|---|
| 运行位置 | 服务端 | 浏览器 |
| 能否用 Hook/事件 | 否 | 是 |
| 能否访问服务端资源 | 是（数据库、文件） | 否 |
| 是否打包进 JS | 否（更小） | 是 |
| 典型用途 | 数据获取、SEO 内容 | 交互、表单、状态 |

```jsx
// 服务端组件：默认，可直接 async 取数据，代码不进浏览器包
export default async function Page() {
  const posts = await db.post.findMany();
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}

// 客户端组件：需要交互时加 "use client"
"use client";
import { useState } from 'react';
export default function Counter() {
  const [n, setN] = useState(0);
  return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```

**数据获取**：服务端组件里直接 `await fetch(...)`，Next.js 会做请求去重与缓存（可用 `export const revalidate = 60` 做 ISR）。

## 26.4 水合（Hydration）

SSR 把 HTML 送到浏览器后，React 还需要在这份静态 HTML 上「接管」——绑定事件、恢复状态，这个过程叫 **hydration**。

```
服务端：渲染 HTML（含内容，但无交互）
   │ 下载到浏览器
   ▼
浏览器：hydration —— React 遍历 HTML，把事件监听/状态"挂"上去
   │ 完成前页面可看但不可交互（点击无效）
   ▼
页面可交互
```

**常见问题**：

- **水合不匹配（hydration mismatch）**：服务端和客户端渲染结果不一致（如用了 `Date.now()`、`Math.random()`）→ React 警告甚至报错
- **解决**：把不确定的值放 `useEffect` 里再渲染，或用 `suppressHydrationWarning`

## 26.5 RSC（React Server Components）

RSC 是 Next.js App Router 的底层能力，核心收益：

1. **减小 JS 体积**：服务端组件不打包进浏览器
2. **服务端直接访问后端资源**：数据获取在服务端，减少请求往返、不暴露密钥
3. **流式渲染（Streaming）**：配合 `<Suspense>` 边渲染边发送，首屏更快

```jsx
import { Suspense } from 'react';
export default function Page() {
  return (
    <Suspense fallback={<Skeleton />}>   {/* 慢的部分先显示骨架屏 */}
      <SlowComponent />
    </Suspense>
  );
}
```

## 26.6 面试高频题

1. **CSR 和 SSR 的区别与取舍？** CSR 首屏慢、SEO 差但交互好；SSR 首屏快、SEO 好但服务端压力大、交互需 hydration。

2. **SSG、SSR、ISR 的区别？** 构建时/请求时/构建后按需再生成，对应「静态/动态/准静态」三种内容策略。

3. **什么是水合？水合不匹配怎么来的？** 服务端 HTML 被 React 接管绑定事件的过程；SSR 与 CSR 渲染结果不同（随机值/时间）会导致不匹配。

4. **服务端组件和客户端组件的分工？** 服务端组件负责数据获取与静态内容，客户端组件负责交互；「够用服务端优先」。

5. **RSC 解决了什么问题？** 减小客户端 JS、服务端直连数据源、支持流式渲染，改善首屏与数据获取效率。
