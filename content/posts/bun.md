---
title: "嘗試 Bun 1.0"
date: 2023-09-11
draft: false
author: "Chen Yu Fan"
tags: ["Frontend", "Tools"]
---

Bun 在 2023/9/8 釋出了 1.0 版本，這篇文章就來說一下為什麼 Bun 會被關注，還有該怎麼使用它。

<!--more-->

## Bun Intro

[Bun](https://bun.sh/) 用於開發、測試、運行與打包 JavaScript 或 TypeScript 專案，它將執行階段( runtime )和工具包合成一體化的 JavaScript 進而來提升速度。其中包含打包工具( 如：Webpack、Vite )、測試運行器和與 Node.js 兼容的 package manager( 如：npm、pnpm )。

Bun 的**目標就是要替代 Node.js**，成為世界上最多使用 JavaScript 運行的伺服器，Bun 有著更高的性能和降低複雜度。

有這樣的目標也難怪大家會這麼關注 Bun 了，而目前整體使用下來，這個目標感覺真的有機會成功。( 對的，沒人 care Deno 🤣 )

### 為什麼能提高速度？

Node.js 和 Deno 它們都是使用 Chrome V8 引擎來執行的，而 Bun 是使用 Safari 的 JavaScriptCore 來建構的，所以速度上更快。

以下是由 Bun 官方所測試出來的比較圖：

![bun-server-benchmark.png](/images/bun/bun-server-benchmark.png)

### Bun APIs

上面有說到，Bun 的目標是為了要取代 Node.js，所以也有很多相對應的 APIs 相應而生，像是讀取檔案就可以使用：

```ts
const file = Bun.file(import.meta.dir + '/package.json');

const pkg = await file.json();
pkg.name = 'my-package';
pkg.version = '1.0.0';

await Bun.write(file, pkg);
```

其他更多的 APIs 可以參考 [Bun guides](https://bun.sh/guides)。

### Bundler

Bundler 就像是 Node.js 的 npm，是用來管理套件管理器，官網標榜比 npm 快 30 倍、比 pnpm 快 17 倍。

用起來的方式基本與 npm 一樣，這邊簡單介紹一下：

| npm                    | bundler             |
| ---------------------- | ------------------- |
| npm insatll            | bun insatll         |
| npm insatll packname   | bun add packname    |
| npm uninstall packname | bun remove packname |

Bun 還可以使用 Git 來獲得 dependency：

```bash
bun add git+https://github.com/lodash/lodash.git
bun add github:colinhacks/zod
```

## Installation

首先，先確認自己的環境，如果你的環境是 macOS 與 Linux，那基本上沒什麼問題，但如果你的環境是 Windows，Bun 就會被限制，頂多只能運行不能做其他動作。

這裡我會使用 Linux 環境來示範。如果是 Linux 需要先有 `unzip` 再輸入：

```bash
curl -fsSL https://bun.sh/install | bash 
```

> 如果是 Fish Shell 使用者，Bun 在安裝時就會幫你配置好路徑了

## Quick Start

新建資料夾：

```bash
mkdir test-bun
cd test-bun
```

運行以下指令初始化專案：

```bash
bun init
```

來建立一個簡單的 HTTP server，修改專案裡的 `index.ts`：

```ts
// index.ts
const server = Bun.serve({
  port: 3000,
  fetch(req) {
    return new Response(`Bun!`);
  },
});

console.log(`Listening on http://localhost:${server.port} ...`);
```

運行：

```bash
bun index.ts
```

接著訪問 http://localhost:3000 就可以看到建立好的伺服器了。

## 使用 Vite 與 Bun 建構前端

根據 [Build a frontend using Vite and Bun](https://bun.sh/guides/ecosystem/vite) 敘述，雖然還沒有優化，不過 Vite 與 Bun 是可以一起使用的。

使用 Bun 來建構 Vite 專案：

```bash
bunx create-vite my-app
```

## 結語

整體使用下來最有感的莫非就是 bundler，速度比以前介紹的 pnpm 還要快。一些 API 的使用方式比 Node.js 來的簡單並且效能也更好，運用在後端的建構上是非常方便的。

目前前端會使用到的還是打包工具與 bundler，運行的部分可能還是要交由 Vite 之類的，React 是可以直接與 Bun 結合使用的，不過 Vite 可能就還要在等上一陣子了。