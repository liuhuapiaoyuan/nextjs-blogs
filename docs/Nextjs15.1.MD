# 详细解析Next.js 15.1 的更新特性

本文不直接翻译blog中的更新日志，而是重点讲解一下本次更新带来的新特性。

📚 内容概览：
- 🚀 React 19 稳定版带来的新特性
  - 🎣 新的 Hooks：useActionState, useFormStatus, use, useTransition, useOptimistic
  - 🎨 新的 style 标签功能
- ⏭️ after() API 的稳定版发布
- 🔒 新的实验性 API：forbidden 和 unauthorized
- 💾 关于 use cache 的深入探讨

让我们一起深入了解这些激动人心的新特性吧！👇

## 引入 React 19 稳定版带来的新特性

本文挑选一些重点更新介绍：

### 1. 新的hook：useActionState

这是本轮最大的改造之一。React 19 引入了 Actions 概念，用于自动管理数据提交流程。useActionState 是基于 Actions 的新钩子，用于处理常见用例。

配套的 `<form>` 标签也支持新的属性 `action`，可以用来代替 `onSubmit`：

```typescript
<form action={actionFunction}>
```

这个改造配合 Next.js 的 Server Actions 可以极大地简化前后端的数据提交模式。它自动处理了挂起状态、错误处理，甚至支持乐观更新。

### 2. 新的hook：useFormStatus

useFormStatus 允许我们读取父 `<form>` 的状态，就像 `<form>` 是一个 Context 提供者一样。这使得我们可以轻松地根据表单的提交状态来更新 UI，比如显示加载指示器或禁用提交按钮。

### 3. 新的hook：use

use 是一个用于在渲染中读取资源的新 API。它允许我们在组件中直接使用 Promise，React 会在 Promise 解析之前自动进入挂起状态：

```javascript
function Profile() {
  const user = use(fetchUser());
  return <h1>{user.name}</h1>;
}
```

这大大简化了异步数据获取的处理。

### 4. 新的hook：useTransition

useTransition 钩子允许我们在不阻塞 UI 的情况下更新状态。它特别适用于处理较大的状态更新，可以提高应用的响应性：

```javascript
const [isPending, startTransition] = useTransition();

startTransition(() => {
  // 进行可能耗时的状态更新
});
```

### 5. 新的hook：useOptimistic

useOptimistic 是一个用于在 React 中实现乐观更新的新 API。它允许我们在不阻塞 UI 的情况下更新状态，可以提高应用的响应性。

```javascript
const [optimisticState, addOptimistic] = useOptimistic(
  state, // state: 初始值以及在没有待处理操作时返回的值。 
  (currentState, newValue) => {
    // 更新函数可以把新的的数据更新到state
    // 当异步操作接受后，state会被回滚
    // 特别适合比如聊天，发布文章，可以先发布一个sending状态的消息
  }
);
```

### 6. 新的标签：style

React 19 为内置的 `<style>` 标签增加了新的功能。现在我们可以更灵活地控制样式的应用：

```jsx
<style precedence="lower" media="(max-width: 600px)">
  {`
    .responsive-layout {
      flex-direction: column;
    }
  `}
</style>
```

这个新特性让我们能够更好地管理样式优先级，支持媒体查询，甚至可以指定备用样式表。它为 React 应用中的样式管理提供了更强大的工具。


## after() API 在 Next.js 15 首个 RC 版本中首次引入，现在已经稳定。

- after() 提供了一种在响应完成流式传输到用户之后执行任务（如日志记录、分析和系统同步）的方法，而不会阻塞主要响应。
- 支持在服务器操作（Server Actions）和路由处理程序（Route Handlers）中使用 cookies() 和 headers() 等运行时 API。

```javascript
import { after } from 'next/server';
import { log } from '@/app/utils';
 
export default function Layout({ children }) {
  // Secondary task
  after(() => {
    log();
   });
  
   // Primary task
   return <>{children}</>;
}
```

## 引入  `forbidden`和`unauthorized` 实验版

作为一个经常使用 Next.js 的开发者，我对这个新特性感到非常兴奋。
这两个 API 的引入，让我们在处理授权和认证问题时有了更多的灵活性。它们的工作方式类似于我们熟悉的 `notFound()`，但专门用于处理 403 和 401 错误。

让我来分享一下我的初步使用体验：

1. 首先，我们需要在 `next.config.ts` 中启用这个实验性功能：

```typescript
const nextConfig: NextConfig = {
  experimental: {
    authInterrupts: true,
  },
};
```

2. 然后，我们就可以在各种场景中使用这些新 API 了。比如，在一个需要管理员权限的页面中：

```typescript
import { checkAdminStatus } from '@/lib/auth';
import { forbidden } from 'next/navigation';

export default async function AdminDashboard() {
  const isAdmin = await checkAdminStatus();

  if (!isAdmin) {
    forbidden();
  }

  return <h1>欢迎来到管理员仪表板</h1>;
}
```

3. 最棒的是，我们可以自定义错误页面，使其更符合我们的应用风格：

```typescript
// app/forbidden.tsx
export default function Forbidden() {
  return (
    <div className="error-container">
      <h2>抱歉，您没有权限访问此页面</h2>
      <p>如果您认为这是一个错误，请联系管理员。</p>
      <Link href="/">返回首页</Link>
    </div>
  );
}
```

总的来说，这个新特性大大简化了我们处理授权和认证的流程。不过，由于它还处于实验阶段，我建议在非关键项目中先试用一下，看看是否符合你的需求。
 
## 关于`Cache`的爱恨情仇

> 不得不说，`use cache` 的引入非常有意思。虽然它还没有进入正式版，但我认为这个特性值得我们提前了解和关注。

让我们来聊聊 `use cache` 带来的一些变化：

1. 更精细的缓存控制
   以前，Next.js 的缓存机制对我们来说就像一个黑盒，有时候真让人摸不着头脑。现在，`use cache` 给了我们更多的主动权。我们可以在页面级别、组件级别，甚至是函数级别来控制缓存。这种灵活性真的很棒！

2. 动态参数与缓存的和谐共处
   一个令人惊喜的特性是，`use cache` 允许我们将非可序列化的参数传递给可缓存的函数或组件，而不会影响缓存。这在处理复杂数据结构时特别有用。

3. 默认缓存时间
   Next.js 默认设置了 15 分钟的重新验证周期，这个时间设置挺合理的。不过，如果需要的话，我们也可以根据具体需求进行调整。

4. 缓存标签
   `cacheTag` 和 `revalidateTag` 的引入让缓存管理变得更加灵活。我们可以给缓存打上标签，然后通过这些标签来精确控制哪些缓存需要重新验证。这对于管理大型应用的缓存策略来说，简直是福音。

虽然 `use cache` 还在实验阶段，但我已经迫不及待想在下一个项目中尝试了。它有助于解决了我们长期以来对 Next.js 缓存管理的困扰，让我们能够更好地掌控应用的性能和数据新鲜度。

不过，作为一个谨慎的开发者，我建议大家在非关键项目中先试水，充分测试这些新特性。毕竟，它们还在实验阶段，可能会有一些意想不到的问题。

总的来说，`use cache` 的引入标志着 Next.js 在缓存管理方面迈出了一大步。它让我们离构建更高效、更可控的 Web 应用又近了一步。
