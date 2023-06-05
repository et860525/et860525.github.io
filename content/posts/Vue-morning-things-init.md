---
title: "Vue - Morning Things 專案(一) - 初始化"
date: 2023-06-05
draft: false
author: "Chen Yu Fan"
tags: ["Vue", "Frontend", "TypeScript", "Projects", "API"]
---

終於要往前端發展了，在後端打滾的時候常常也會接觸到一些前端的技術。在框架的部分，我一開始是先學 `Svelte`，聽前端的朋友說其實框架都差不多，所以我就來學學看 [Vue](https://vuejs.org/) 了。

基本上我學習前端的目的就是為了要讓串接的 API 呈現的漂亮一點，所以這個系列我會試著用我那可笑的美術細胞🥲，好好的建構一個專案。

<!--more-->

## 專案動機

這個專案的動機是：**早上只要打開這個網頁就可以知道一天的大小事**。

該專案包含：

- 顯示氣象 (API 呼叫)
- 顯示自定義新聞 (RSS 儲存)
- 登入機制

## 初始化專案

根據 [Vue - Quick Start](https://vuejs.org/guide/quick-start.html) 來建立一個 Vue 專案。

```bash
pnpm init vue@latest
```

可以輸入專案名稱，輸入完後，會有選項可以選擇，只有兩個設定我按 Yes：

- `Add TypeScript?`：Yes
- `Add Vue Router for Single Page Application development?`：Yes

安裝專案所需的套件：

```bash
cd <your-project-name>
pnpm install
pnpm dev
```

這樣就完成專案的設定了。

> 如果是不是運行在本機上，如：虛擬機，請使用 `pnpm dev --host`  來啟動伺服器，否則會連不上

## TailwindCSS 設定

根據 [Install Tailwind CSS with Vue 3 and Vite](https://v2.tailwindcss.com/docs/guides/vue-3-vite) 來進行初始化設定。

安裝 Tailwind 和相關套件：

```bash
pnpm install -D tailwindcss@latest postcss@latest autoprefixer@latest
```

建立設定檔 (輸入完後會建立一個 `tailwind.config.js`)：

```bash
npx tailwindcss init -p
```

以下是我的 `tailwind.config.js` 設定：

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ['./index.html', './src/**/*.{vue,js,ts,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        'weather-primary': '#00668A',
        'weather-secondary': '#004171',
      },
    },
    fontFamily: {
      Roboto: ['Roboto, sans-serif'],
    },
  },
  plugins: [],
};
```

我這裡使用 [Roboto](https://fonts.google.com/specimen/Roboto) 字體：

- Light 300
- Regular 400
- Medium 500

把產生出來的 `<link>` 加到 `./index.html`：

```html
<head>
  <link href="https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500&display=swap" rel="stylesheet">
</head>
```

將 Tailwind 設定加入到 CSS 中：

- `./assets/index.css`
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

- `./src/main.ts`
```ts
import { createApp } from 'vue';
import App from './App.vue';
import router from '.';
import './assets/index.css';

const app = createApp(App);

app.use(router);

app.mount('#app');
```

重新啟動一次專案就可以套用 TailwindCSS 了。

## Navigation

> 在建立 Nab bar 之前，我已經先把沒有用到的組件(Components, CSS, views...) 都先清理了。

先在 `components` 裡建立一個 `Nav.vue`，並把它引入到 `App.vue`：

- `App.vue`
```js
<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router';
import Nav from './components/Nav.vue';
</script>

<template>
  <div class="flex flex-col min-h-screen font-Roboto bg-weather-primary">
    <Nav />
    <RouterView />
  </div>
</template>
```
- `Nav.vue`
```js
<template>
  <header class="stick top-0 bg-weather-primary shadow-lg">
    <nav class="container flex flex-col sm:flex-row items-center">
      <RouterLink :to="{ name: 'home' }">
        <div class="flex items-center flex-1 gap-4 py-6 text-white">
          <i class="fa-solid fa-mug-hot text-2xl"></i>
          <p class="text-2xl">Morning Things</p>
        </div>
      </RouterLink>
    </nav>
  </header>
</template>
<script setup lang="ts">
import { RouterLink } from 'vue-router';
</script>
```

因為這裡我有使用 [font-awesome](https://fontawesome.com/) 的 icons，必須要引入 [font-awesome cdn.js](https://cdnjs.com/libraries/font-awesome) 到 `./index.html`：

```html
<head>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css"
    integrity="sha512-iecdLmaskl7CVkqkXNQ/ZH/XLlvWZOJyj7Yy7tcenmpD1ypASozpmT/E0iPtmFIB46ZmdtAc9eNBvH0H/ZpiBw=="
    crossorigin="anonymous" referrerpolicy="no-referrer" />
</head>
```

接著重新整理畫面就可以看到結果了。

程式碼部分同步在 [et860525/Morning-things](https://github.com/et860525/Morning-things)。

---

## 結語

跟著 Doc 做都不會很難，這些引入的方式都跟後端差不多，寫過 Spring Boot 才會知道有這種互動式的 Package Manager 的方便。

下一篇就要開始串接 API，如果沒有 API KEY 是沒辦法使用 API 的，可以先到 [氣象資料開放平台](https://opendata.cwb.gov.tw/index) 來註冊會員。