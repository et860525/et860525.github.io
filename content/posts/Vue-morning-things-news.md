---
title: "Vue - Morning Things 專案(四) - 顯示新聞區域"
date: 2023-06-12
draft: false
author: "Chen Yu Fan"
tags: ["Vue", "Frontend", "TypeScript", "Projects", "API", "Express"]
---

這一篇來講顯示新聞的區域，從建立一個後端來獲得 RSS 並將它們整理好，再由 Vue 來呼叫 API 獲得並在最後顯示出來。

<!--more-->

整個請求的流程會像是這樣：

![vue-project-frontend-backend-flow.png](/images/Vue-project/vue-project-frontend-backend-flow.png)


## 後端建立

一開始我想直接向各大有提供 RSS 的新聞網站直接請求，不過遇到了一個問題：

![vue-project-proxy-cors-error.png](/images/Vue-project/vue-project-proxy-cors-error.png)

[CORS (Cross-Origin Resource Sharing)](https://developer.mozilla.org/zh-TW/docs/Web/HTTP/CORS) 跨站來源請求的錯誤，解決的方法通常都是請該伺服器端做 CORS 標頭設定，但是我們並沒有辦法去告訴伺服器端幫我們開通，所以這裡我使用插件的方式來獲得 RSS。

這裡會在 `root` 下建立一個 `server` 資料夾，並放入 `index.ts` 檔案：

```ts
import express, { Express, Request, Response } from 'express';
import Parser, { type Item } from 'rss-parser';

const port = process.env.PORT || 3000;

const app: Express = express();
const parser = new Parser();

// 獲得文章的最大數
const MAX_POST: number = 5;

// 要獲得的 RSS 們
const urls: string[] = [
  'https://www.nasa.gov/rss/dyn/breaking_news.rss',
  'https://news.ltn.com.tw/rss/all.xml',
  'https://spacenews.com/feed/',
];

app.get('/api/v1/rss', async (req: Request, res: Response) => {
  let feeds: Item[] = [];
  
  for (let i = 0; i < urls.length; i++) {
    const rss = await parser.parseURL(urls[i]);

    // If items < 5, then get all elements
    if (rss.items.length < MAX_POST) {
      rss.items.forEach((item) => {
        item.origin_title = rss.title;
        item.origin_link = rss.link;
        feeds.push(item);
      });
    }

    // Get 5 posts from every rss feed
    for (let i = 0; i < MAX_POST; i++) {
      rss.items[i].origin_title = rss.title;
      rss.items[i].origin_link = rss.link;
      feeds.push(rss.items[i]);
    }
  }

  // return post like: [1,2,3,1,2,3,1,2,3]
  res.json({ feeds: feeds });
});

app.listen(port, () => {
  console.log(`[server]: Server is running at http://localhost:${port} in dev`);
});
```

- 使用 [rss-parser](https://www.npmjs.com/package/rss-parser) 來獲得 RSS
- 使用 for looping 來依序獲得資料
- `origin_title` 與 `origin_link` 來設定這些文章來自哪裡

上面程式我曾經遇到的一個問題：最開始我使用 `forEach` 來獲得 looping 我的 RSS 們，並把獲得的資料放進預先設定的 Array。但是不管我怎麼樣跑，它都不能把獲得的 RSS 資料放進我預先設定好的 Array。最後我在 [Cannot push into array from inside forEach loop](https://stackoverflow.com/questions/68072857/cannot-push-into-array-from-inside-foreach-loop) 查到原來 `forEach` 方法不會等待 async 呼叫，它會直接執行，所以它才無法獲得資料並推進 Array 裡。

## Vite 設定檔設定 Proxy

在 Vite 設定要獲得的後端 API 網址，可以預先進行一些設定：

```ts
export default defineConfig({
  //...
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        changeOrigin: true,
      },
    },
  },
});
```

- 在前端就只要使用 `/api/v1/rss` 就可以獲得資料了
- `changeOrigin` 將 host 改成 target 的 URL

## News 元件建立

後端完成後，建立 `News.vue` 元件到 `compoents` 資料夾下：

```ts
<template>
  <NewsCardGroup>
    <NewsCard v-for="(news, key) in news_list.items" :item="news" :key="key" />
  </NewsCardGroup>
</template>

<script setup lang="ts">
import NewsCardGroup from './news/NewsCardGroup.vue';
import NewsCard from './news/NewsCard.vue';
import { reactive, onMounted } from 'vue';
import axios from 'axios';

const news_list: any = reactive({ items: [] });

// 從後端 Server 獲得 RSS API
const fetchRSS = async () => {
  try {
    // Get RSS from server
    const rss = await axios.get('/api/v1/rss');

    // Copy all feeds into 'news_list'
    news_list.items = [];
    news_list.items = Array.from(rss.data.feeds);
  } catch (err) {
    console.log(err);
  }
};

onMounted(() => {
  fetchRSS();
});
</script>
```

### NewsCardGroup.vue

```ts
<template>
  <div class="container grid grid-cols-1 lg:grid-cols-2 gap-4 my-5">
    <slot></slot>
  </div>
</template>
```

### NewsCard.vue

```ts
<template>
  <div class="rounded-lg text-white shadow-2xl" :class="{ 'row-span-1': isImage, 'row-span-2': !isImage }">
    <a :href="props.item.link">
      <h2 class="text-2xl lg:text-3xl text-center p-4" :class="classObject.a_tag_hover">{{ props.item.title }}</h2>
    </a>
    <a :href="props.item.link" v-if="props.item.enclosure !== undefined">
      <img class="max-h-13 w-full p-3 md:px-7" :src="props.item.enclosure?.url" alt="rss-picture" />
    </a>
    <div class="p-3 md:px-7 text-sm md:text-base">
      <div class="flex justify-between items-center">
        <p>
          來源: <a :href="props.item.origin_link" :class="classObject.a_tag_hover">{{ props.item.origin_title }}</a>
        </p>
        <p class="text-end text-gray-300 py-3">{{ dateFormat() }}</p>
      </div>
      <div class="item-content" v-html="props.item.content"></div>
      <p class="text-blue-400 text-right">
        <a :href="props.item.link" class="text-blue-400 text-right" :class="classObject.a_tag_hover">Read More</a>
      </p>
    </div>
  </div>
</template>
<script setup lang="ts">
import { ref, type Ref, reactive } from 'vue';
import { type Item } from 'rss-parser';

// Controll class
const classObject = reactive({
  a_tag_hover: 'hover:underline underline-offset-8',
});

// Extends more detail of item
interface CustomItem extends Item {
  origin_title: string;
  origin_link: string;
}

const props = defineProps<{
  item: CustomItem;
}>();

// Format date
const dateFormat = () => {
  return new Intl.DateTimeFormat('zh-TW', {
    year: 'numeric',
    month: 'short',
    day: 'numeric',
  }).format(new Date(props.item.isoDate!));
};

// Check post have image or not
const isImage: Ref<boolean> = ref(!props.item.content!.includes(`<img`) && props.item.enclosure == undefined);
</script>

<style>
.item-content img {
  width: 100%;
  height: full;
  margin: 0 auto;
}
</style>
```

## 將 News 元件加入 HomeView

與天氣元件一樣，將 News 元件加入到 HomeView：

```ts
<script setup lang="ts">
import Weather from '../components/Weather.vue';
import News from '../components/News.vue';
</script>

<template>
  <Weather />
  <News />
</template>
```

## 啟動專案

現在不單單要啟動 Vite 還必須要同時啟動 server，所以這裡需要 [concurrently](https://www.npmjs.com/package/concurrently) 來同時啟動兩個腳本：

```json
"scripts": {
    "dev:frontend": "vite --host",
    "dev:server": "nodemon server/index.ts",
    "dev": "concurrently 'pnpm:dev:frontend' 'pnpm:dev:server'",
    //...
}
```

這樣後端就會根據前端的請求來獲得相應的資料。

---

## 結語

最近都在面試，所以這禮拜跟下禮拜更新的幅度都會減少一些 🫠。