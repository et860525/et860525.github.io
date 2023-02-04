---
title: "TypeScript: 初始化 Express 專案"
date: 2023-02-04
draft: false
author: "Chen Yu Fan"
tags: ["TypeScript", "Express"]
---

這篇文章會紀錄如何在 [Express](https://expressjs.com/) 專案裡設定 [TypeScript](https://www.typescriptlang.org/)。

先決條件：

- 安裝 [Node.js](https://nodejs.org/en/) ( LTS ) 在你的開發環境上
- 基本的 `Node.js` 、 `Express` 與 `TypeScript` 知識

<!--more-->

## 專案初始化

建立一個空的資料夾，並初始化 `package.json`：

```bash
mkdir express-typescript
cd express-typescript
npm init -y
```

下載 Express 與開發伺服器的相關套件：

```bash
npm i express dotenv
```

- [dotenv](https://www.npmjs.com/package/dotenv) 是一個管理環境變數的套件，可以根據環境的不同設定不同的配置

安裝 TypeScript 與 `@types` 宣告套件：

```bash
npm i -D typescript @types/express @types/node
```

- `@types/<package-name>`：只要該套件有支援 TypeScript，你就能在直接使用這個方式，下載預先定義好的宣告檔案
- `-D` 是 `--save-dev`：表示套件只會存在於 `devDependencies`

都完成後可以檢查 `package.json` 檔案 (沒有特殊需求，套件的版本以自己的為準)：

```json
{
  "name": "express-typescript",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "dotenv": "^16.0.3",
    "express": "^4.18.2"
  },
  "devDependencies": {
    "@types/express": "^4.17.17",
    "@types/node": "^18.11.18",
    "typescript": "^4.9.5"
  }
}
```

## 產生 TypeScript 的配置文件

使用以下指令來產生 `tsconfig.json`：

```bash
npx tsc --init
```

在輸出可以看到產生的檔案與預設的設定：

```bash
Created a new tsconfig.json with:
  target: es2016
  module: commonjs
  strict: true
  esModuleInterop: true
  skipLibCheck: true
  forceConsistentCasingInFileNames: true


You can learn more at https://aka.ms/tsconfig
```

這裡也可以打開 `tsconfig.json` 來修改自己需要的設定。以下是我的設定：

```json
{ 
  "compilerOptions": {
    "target": "es2017",
    "module": "commonjs",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "rootDir": "./src",
    "outDir": "./dist",
  },
  "include": ["src/**/*.ts"], 
  "exclude": ["node_modules", "dist"]
}
```

預設的設定：

- `target`：指定編譯器所產生的 JavaScript version
- `module`：編譯 JavaScript 程式碼使用的 modules manager。`commonjs` 為 Node.js 的標準
- `strict`：嚴格類型檢查選項
- `esModuleInterop`：讓我們能將 ES6 modules 編譯成 commonJS modules
- `skipLibCheck`：如果為 `true`，則會跳過檢查基礎宣告檔案 ( declaration files )
- `forceConsistentCasingInFileNames`：如果為 `true`，啟用區分大小寫命名文件

新增的設定：

- `rootDir`：欲編譯的路徑，把該資料夾裡面的 `.ts` 都編譯成 JavaScript
- `outDir`：編譯後輸出檔案的地方
- `include`：納入編譯的範圍
- `exclude`：不納入編譯的範圍

## 建立 Express app

首先，先在根目錄 ( root ) 下建立一個檔案 `.env`，這個檔案存放的是敏感資料供 `dotenv` 來讀取的。該檔案目前會有以下設定：

```txt
PORT=3000
```

之後再新增檔案 `src/app.ts` 並輸入下面的程式碼來建立一個伺服器：

```typescript
import express, { Express, Request, Response } from 'express';
import dotenv from 'dotenv';

dotenv.config();

const app: Express = express();
const port = process.env.PORT;

app.get('/', (req: Request, res: Response) => {
  res.send('Express + TypeScript Server');
});

app.listen(port, () => {
  console.log(
    `[server]: Server is running at http://localhost:${port}`
  );
});
```

- 透過 `dotenv` 就可以讓 `port` 獲取在 `.env` 檔案裡的設定

要運行伺服器前必須要先知道一件事情，瀏覽器是無法理解 TypeScript 程式碼的，必須要先將檔案轉成 JavaScript 瀏覽器才能使用。那要如何才能讓 TypeScript 知道哪些檔案是需要編譯的呢？這時候設定 `tsconfig.json` 就很重要了 (請參考上方 `tsconfig.json` 所新增的設定)。

如果已經有設定指定的資料夾，就可以使用下列指令來將 `.ts` 檔案轉為 `.js`：

```bash
tsc
```

轉換完成後就可以運行伺服器了：

```bash
node dist/app.js
```

畫面上會顯示 `[server]: Server is running at http://localhost:3000` 連接到後面的網址，就可以看到伺服器正常運行的畫面了。

> 按下 `Ctrl+c` 就可以停止伺服器

## 持續監看 TypeScript 檔案

在開發期間，通常會頻繁的改變網頁內容並觀察其結果，如果以上述方式每一次都要先編譯再執行，肯定會造成許多不方便，而且隨著時間增長程式也會越來越大，所編譯的時間也會相對增加。

可以使用 [nodemon](https://www.npmjs.com/package/nodemon) 來解決這個問題。nodemon 是一個幫助開發者的工具，當目錄裡的檔案改變時，它會自動偵測並幫助我們重開應用程式。但是重開應用程式是不夠的，因為 TypeScript 還有要先編譯的問題，所以可以搭配 [ts-node](https://www.npmjs.com/package/ts-node) 一起使用 (如果沒有安裝此套件，在執行 nodemon 時就會要你安裝)。

安裝兩個所需要的套件：

```bash
npm i -D nodemon ts-node
```

打開 `package.json` 來編寫腳本：

```json
"scripts": {
  "build": "tsc",
  "dev": "NODE_ENV=development nodemon ./src/app.ts"
}
```

運行腳本：

```bash
npm run dev
```

你可以嘗試改變 `res.send('Express + TypeScript Server');` 裡的文字，儲存後重整網頁就能看到改變的結果，就不需要進行編譯再執行的動作了。

## 結論

以上就是建立一個最基本 Express + TypeScript 的方式。