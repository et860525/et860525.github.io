---
title: "Vue - Morning Things 專案(五) - Not Found 頁面"
date: 2023-06-21
draft: false
author: "Chen Yu Fan"
tags: ["Vue", "Frontend", "TypeScript", "Projects", "API"]
---

端午連假前的最後一篇，來建立一個 404 頁面。

<!--more-->

## NotFound Page

如果在網址列上面輸入沒有登記在 `router` 裡的網址，那就會被導向一個空白的頁面。

所以這邊要對 `./src/router/index.ts` 來做設定 (來源：[Vue Router - 404](https://router.vuejs.org/guide/essentials/dynamic-matching.html#catch-all-404-not-found-route))：

```ts
const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    {
      path: '/',
      name: 'home',
      component: HomeView,
    },
    {
      path: '/login',
      name: 'login',
      component: LoginView,
    },
    {
      path: '/:pathMatch(.*)*',
      name: 'NotFound',
      component: NotFound,
    },
  ],
});
```

`'/:pathMatch(.*)*'` 這代表**所有**的沒預先登入的 `path` 都會被導向 `NotFound`。

> ⚠️ 注意：`'/:pathMatch(.*)*'` 這一段一定要在最下面，如果在最上面就會發生所有 `path` 都會被導向 `NotFound`

---

## 結語

目前這個系列可能會先停一陣子，這兩個禮拜的面試我發現了還有很多東西要先補回來，往後可能會先以複習的方式把一些東西撿回來。