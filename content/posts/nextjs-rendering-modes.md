---
title: "Next.js 核心渲染模式解析：SSG、ISR、SSR、CSR"
date: 2025-10-20
draft: false
tags: ["nextjs", "ssr", "ssg", "programming"]
categories: ["tech"]
description: "深入理解 Next.js 的四种渲染模式：静态生成(SSG)、增量静态再生成(ISR)、服务端渲染(SSR)和客户端渲染(CSR)，以及它们各自的适用场景。"
---

在 Next.js 开发中，选择合适的渲染模式是提升应用性能的关键。本文详细解析 Next.js 的四种核心渲染模式。

## 什么是渲染模式？

渲染模式决定了**页面何时生成 HTML**以及**由谁来生成**。不同的模式在性能、实时性和 SEO 方面各有优劣。

<!--more-->

## 1. SSG - Static Site Generation（静态生成）

**核心原理**：构建时生成 HTML，之后直接复用。

```typescript
// pages/index.tsx (Pages Router)
export async function getStaticProps() {
  const res = await fetch('https://api.example.com/posts');
  const posts = await res.json();

  return {
    props: { posts },
    // 不需要 revalidate，就是纯静态
  };
}

// App Router
export async function generateStaticParams() {
  const posts = await fetchPosts();
  return posts.map((post) => ({ slug: post.slug }));
}
```

**特点**：
- 构建阶段生成所有页面
- 用户访问时直接返回静态文件，速度最快
- 数据在构建时固定，不支持实时更新

**适用场景**：
- 博客、文档网站
- 产品介绍页
- 不需要频繁更新的内容

---

## 2. ISR - Incremental Static Regeneration（增量静态再生成）

**核心原理**：页面首次访问时生成 HTML，之后在后台定时重新生成。

```typescript
// Pages Router
export async function getStaticProps() {
  const res = await fetch('https://api.example.com/posts');
  const posts = await res.json();

  return {
    props: { posts },
    revalidate: 60, // 60秒后重新生成
  };
}

// App Router
export const revalidate = 60;

export async function Page() {
  const posts = await fetchPosts();
  return <BlogList posts={posts} />;
}
```

**特点**：
- 页面大部分时间是静态的
- 数据可以定期更新，不需要每次请求都计算
- 支持"缓存在后台更新"的机制

**适用场景**：
- 电商商品列表
- 新闻文章列表
- 需要平衡性能和数据新鲜度的场景

**工作流程**：
```
用户首次请求 → 生成HTML → 缓存
60秒后再次请求 → 后台重新生成 → 更新缓存
```

---

## 3. SSR - Server-Side Rendering（服务端渲染）

**核心原理**：每次请求时实时在服务器生成 HTML。

```typescript
// Pages Router
export async function getServerSideProps() {
  const res = await fetch('https://api.example.com/user/profile', {
    // SSR 每次都请求，确保数据实时
    cache: 'no-store',
  });
  const user = await res.json();

  return {
    props: { user },
  };
}

// App Router
export const dynamic = 'force-dynamic';

export async function Page() {
  const user = await fetchUser();
  return <UserProfile user={user} />;
}
```

**特点**：
- 每次请求都需要计算
- 数据始终保持最新
- 服务器负载相对较高

**适用场景**：
- 需要实时数据的页面（如股价、库存）
- 用户专属的个性化内容
- 需要 SEO 且数据频繁变化的页面

---

## 4. CSR - Client-Side Rendering（客户端渲染）

**核心原理**：服务器只返回空 HTML，浏览器用 JavaScript 请求数据并渲染。

```typescript
// App Router - 默认就是 CSR
export default function Page() {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/posts')
      .then((res) => res.json())
      .then((data) => {
        setPosts(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <Loading />;
  return <BlogList posts={posts} />;
}
```

**特点**：
- 首屏渲染依赖 JavaScript
- SEO 不友好（需要额外配置）
- 减轻服务器压力

**适用场景**：
- 后台管理面板
- 需要用户交互后才知道显示什么的页面
- 社交媒体信息流

---

## 模式对比

| 模式 | 速度 | 实时性 | SEO | 服务器负载 |
|------|------|--------|-----|-----------|
| SSG | 最快 | 构建时固定 | 友好 | 低 |
| ISR | 快 | 定时更新 | 友好 | 中 |
| SSR | 较慢 | 实时 | 友好 | 高 |
| CSR | 依赖网络 | 实时 | 不友好 | 低 |

---

## 如何选择？

1. **数据不经常变化** → SSG
2. **数据变化不频繁，但需要定期更新** → ISR
3. **需要实时数据或个性化内容** → SSR
4. **纯交互式页面，用户登录后才能看到** → CSR

## 实际应用示例

假设一个博客系统：

```typescript
// 首页 - 使用 ISR，每分钟更新
export const revalidate = 60;
async function HomePage() {
  const posts = await fetchPosts();
  return <BlogList posts={posts} />;
}

// 文章详情页 - 使用 SSG，构建时生成所有
export async function generateStaticParams() {
  const posts = await fetchPosts();
  return posts.map((post) => ({ slug: post.slug }));
}

// 用户个人页 - 使用 SSR，每次实时获取
export const dynamic = 'force-dynamic';
async function UserPage() {
  const user = await fetchCurrentUser();
  return <UserProfile user={user} />;
}

// 管理后台 - CSR
export default function AdminPage() {
  // 纯客户端渲染
}
```

---

## 总结

Next.js 的四种渲染模式各有特点，没有绝对的优劣。根据业务的实际需求选择合适的渲染模式，才能在性能、实时性和开发成本之间找到最佳平衡点。

现代前端开发的一个重要趋势是**尽可能使用静态生成**，因为静态内容的性能优势非常明显。只有在确实需要实时数据时才考虑 SSR 或 CSR。


