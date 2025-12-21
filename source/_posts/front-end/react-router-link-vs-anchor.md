---
title: React 路由跳转：从 a 标签到编程式导航
date: 2025-12-21 11:29:42
tags: [react, react-router]
categories: [前端]
---

# React 路由跳转：从 a 标签到编程式导航

在 React 应用开发中，正确处理页面跳转是确保应用性能和用户体验的关键。本文将对比传统的 `<a>` 标签与 React Router 提供的导航方式。

### 1. 为什么不推荐使用 <a> 标签？

标准的 `<a>` 标签通过 `href` 属性进行跳转时，会触发浏览器的**全页刷新**。
* **状态丢失**：应用内存中的所有 State（如登录信息、表单输入）都会被重置。
* **加载冗余**：浏览器会重新请求所有资源（JS, CSS），失去了单页应用（SPA）的性能优势。

### 2. 声明式导航：Link 组件

对于普通的导航链接，应当使用 `react-router-dom` 提供的 `<Link>` 组件。它通过拦截点击事件并使用 History API 来更新 URL，从而实现**无刷新跳转**。

```jsx
import { Link } from 'react-router-dom';

// 基础用法
<Link to="/reset-password">Forgot password?</Link>
```

结合 MUI (Material UI) 使用： 为了保持 MUI 的样式，可以通过 component 属性进行组合：

```jsx
import { Link as RouterLink } from 'react-router-dom';
import { Link } from '@mui/material';

<Link component={RouterLink} to="/reset-password">
  Forgot password?
</Link>
```

### 3. 编程式导航：useNavigate

在某些场景下，我们需要在执行完特定逻辑（如：表单提交、权限验证）后再进行跳转，这时就需要使用 useNavigate 钩子。

```jsx
import { useNavigate } from 'react-router-dom';

function LoginPage() {
  const navigate = useNavigate();

  const handleLogin = async () => {
    // 假设这里有一段异步登录逻辑
    const success = await loginApi();
    
    if (success) {
      // 逻辑完成后跳转
      navigate('/dashboard');
    }
  };

  return <button onClick={handleLogin}>Login</button>;
}
```

## 总结

| 使用场景 | 推荐实现方式 |
| --- | --- |
| **普通的文字链接** | `<Link to="...">` |
| **需要 UI 框架样式 (如 MUI)** | `<MuiLink component={RouterLink} to="..." />` |
| **逻辑处理（如提交表单）后跳转** | `const navigate = useNavigate(); navigate(...)` |
| **跳转到外部网站（非本应用页面）** | `<a href="...">` |

通过合理使用这些工具，可以确保 React 应用在提供丝滑交互的同时，维持应用状态的连续性