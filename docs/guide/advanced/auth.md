---
title: 权限管理
order: 9
---

对于一个 Web 应用，权限管理是经常会涉及的需求之一，通常包含以下几种常见的权限管理类型：

- 页面权限：当用户访问某个没有权限的页面时跳转到无权限页面；
- 操作权限：页面中的某些按钮或组件针对无权限的用户直接隐藏；
- 接口权限：当用户通过操作调用没有权限的接口时跳转到无权限页面。

## 初始化权限数据

大多数情况下权限管理通常需要从服务端获取权限数据，然后在前端通过权限对比以此控制页面、操作等等权限行为。在 icejs 框架中约定通过 `getInitialData` 从服务端异步获取初始化的权限数据，并且约定最终返回格式为 `{auth: {[key: string]: boolean }}` 的形式。

```tsx
import { runApp, request, IAppConfig } from 'ice';

const appConfig: IAppConfig = {
  app: {
    getInitialData: async () => {
      // 模拟服务端返回的数据
      const data = await request('/api/auth');
      const { role, starPermission, followPermission } = data;

      // 约定权限必须返回一个 auth 对象
      // 返回的每个值对应一条权限
      return {
        auth: {
          admin: role === 'admin',
          guest: role === 'guest',
          starRepo: starPermission,
          followRepo: followPermission,
        },
      };
    },
  },
  auth: {
    // 可选的，设置无权限时的展示组件，默认为 null
    NoAuthFallback: <div>没有权限...</div>,
    // 或者传递一个函数组件
    // NoAuthFallback: () => <div>没有权限..</div>
  },
};

runApp(appConfig);
```

## 页面权限

页面权限通常也称之为路由权限，如需对某些页面进行权限控制只需在页面组件的 `pageConfig` 中配置准入权限即可。

```diff
// src/routes.ts
import Home from '@/pages/Home';

export default [
  {
    path: '/home',
    component: Home,
+    // icejs 1.x 仅支持将 pageConfig 配置在对应的页面组件上，请参考「页面组件」章节
+    pageConfig: {
+      auth: ['admin'],
+    },
  },
];

export default routerConfig;
```

## 操作权限

在某些场景下，如某个组件中要根据角色判断是否有操作权限，我们可以通过 `useAuth` Hooks 在组件中获取权限数据，同时也可以更新初始的权限数据。

### 获取权限数据

```tsx
import React from 'react';
import { useAuth } from 'ice';

function Foo() {
  const [auth] = useAuth();
  return (
    <>
      当前用户权限数据：
      <code>{JSON.stringify(auth)}</code>
    </>
  );
}
```

### 设置权限数据

```tsx
import React from 'react';
import { useAuth } from 'ice';

function Foo() {
  const [auth, setAuth] = useAuth();

  // 更新权限，与默认的 auth 数据进行合并
  function updateAuth() {
    setAuth({ starRepo: false, followRepo: false });
  }

  return (
    <>
      当前用户角色：
      <code>{JSON.stringify(auth)}</code>
      <button type="button" onClick={updateAuth}>
        更新权限
      </button>
    </>
  );
}
```

### 自定义权限组件

对于操作类权限，通常我们可以自定义封装权限组件，以便更细粒度的控制权限和复用。

```ts
import React from 'react';
import { useAuth } from 'ice';
import NoAuth from '@/components/NoAuth';

function Auth({ children, authKey, fallback }) {
  const [auth] = useAuth();
  // 判断是否有权限
  const hasAuth = auth[authKey];

  // 有权限时直接渲染内容
  if (hasAuth) {
    return children;
  } else {
    // 无权限时显示指定 UI
    return fallback || NoAuth;
  }
}

export default Auth;
```

使用如下：

```tsx
function Foo() {
  return (
    <Auth authKey={'starRepo'}>
      <Button type="button">Star</Button>
    </Auth>
  );
}
```

## 接口鉴权

请参考文档 [数据请求](/guide/basic/request.md)，业务上封装统一的请求方法，与服务端约定接口协议，前端根据状态码判断无权限、未登录等状态，然后跳转到指定页面。

## 版本升级

### 不再依赖状态管理方案

icejs 2.0 开始，auth 直接基于 Context 实现，不再依赖状态管理方案，开发者使用方式保持一致。

### 插件内置

icejs 1.11.2 开始框架默认内置权限插件，开发者无需安装。如果 ice.js 版本低于 1.11.2，需要自行安装 `build-plugin-ice-auth` 依赖，并在 `build.json` 中引入该插件。