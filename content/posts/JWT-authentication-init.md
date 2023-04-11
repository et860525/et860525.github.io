---
title: "Node.js + JWT Authentication 專案(一) - 初始化專案"
date: 2023-04-11
draft: false
author: "Chen Yu Fan"
tags: ["Node.js", "TypeScript", "Express", "MongoDB", "Projects", "API"]
---

這個專案會使用 Node.js 和 TypeScript 來建構 REST API 後端，使用 [JWT](https://jwt.io/) 來實作身分認證與授權。

> 此專案會遵循我慣用的 OOP 架構  [et860525/express-project-architecture](https://github.com/et860525/express-project-architecture)，有鑑於上一次專案的經驗，由於這些都只是小專案，我不會把所有東西都全部都包在 class 裡面

建構此專案會用到的重要套件：

| Package                                                    | Usage                              |
| ---------------------------------------------------------- | ---------------------------------- |
| [Express](https://expressjs.com/)                          | Web 應用框架                       |
| [TypeScript](https://www.typescriptlang.org/)              | 開發工具                           |
| [Mongoose](https://mongoosejs.com/)                        | 訪問資料庫                         |
| [Docker](https://www.docker.com/)                          | 應用容器化                         |
| [MongoDB](https://www.mongodb.com/)                        | 儲存使用者的資料庫                 |
| [Redis](https://www.npmjs.com/package/redis)               | 儲存使用者緩存的 session 資料庫    |
| [JsonWebToken](https://github.com/auth0/node-jsonwebtoken) | 產生 JWTs                          |
| [Bcryptjs](https://github.com/dcodeIO/bcrypt.js)           | 密碼加密                           |
| [Zod](https://github.com/colinhacks/zod)                   | 驗證使用者的輸入                   |
| [Typegoose](https://typegoose.github.io/typegoose/)        | 使用 TypeScript 優化 Mongoose 模型 |
| [Dotenv](https://github.com/motdotla/dotenv)               | 讀取環境變數                       |
| [Cors](https://github.com/expressjs/cors)                  | 允許資料能在前端與後端之間分享     |
| [lodash](https://lodash.com/)                              | 對 JavaScript 的功能擴充           |
| [ts-node-dev](https://github.com/wclr/ts-node-dev)         | 當檔案變更時自動重啟               |

<!--more-->

## JWT 驗證流程

![JWT-authentication-flow.png](/images/JWT-authentication/JWT-authentication-flow.png)

使用者需要提供瀏覽器帳號密碼來做登入。**前端**會發送帶有使用者憑證的請求到**後端**，當**後端**驗證成功後，會傳送使用者相關的 `cookies`  回到**前端**。

- 使用者註冊的流程：

	![JWT-registration-flow.png](/images/JWT-authentication/JWT-registration-flow.png)

- 使用者登入的流程：

  ![JWT-login-flow.png](/images/JWT-authentication/JWT-login-flow.png)

## API 路由設置

| HTTP Method | Route                | Description          |
| ----------- | -------------------- | -------------------- |
| GET         | `/api/users`         | 回傳所有使用者的資訊 |
| GET         | `/api/users/me`      | 回傳已登入者的資訊   |
| POST        | `/api/auth/register` | 註冊新用戶           |
| POST        | `/api/auth/login`    | 登入                 |

## 初始化專案

- 建立資料夾並初始化：
```bash
mkdir jwt-auth-node
cd jwt-auth-node
pnpm init
```

### 安裝套件

- 安裝相關套件：
```bash
pnpm install @typegoose/typegoose bcryptjs cookie-parser dotenv express jsonwebtoken lodash mongoose redis ts-node-dev zod cors
```

- 安裝開發用套件：
```bash
pnpm install -D morgan typescript
```

- 安裝 Type Definition 檔案：
```bash
pnpm install -D @types/bcryptjs @types/cookie-parser @types/express @types/jsonwebtoken @types/lodash @types/morgan @types/node @types/cors
```

### TypeScript 相關設定

- 產生 `tsconfig.json` 檔案：
```bash
tsc --init
```

- `tsconfig.json`：
```json
{
  "compilerOptions": {
    "target": "es2017",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "module": "commonjs",
    "rootDir": "./src",
    "outDir": "./dist",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

> 在 [How to Start using typegoose](https://typegoose.github.io/typegoose/docs/guides/quick-start-guide#how-to-start-using-typegoose) 有說明，以下兩個設定一定要開啟：
> - `experimentalDecorators: true`
> - `emitDecoratorMetadata: true`

### 簡單的 Express Server

- 建立 `./.env` 檔案：
```env
PORT=3000
```

- `./src/app.ts`：
```typescript
import dotenv from 'dotenv';
import express, { Request, Response } from 'express';

dotenv.config();

const env = process.env.NODE_ENV;
const port = process.env.PORT;

const app = express();

app.get('/', (req: Request, res: Response) => {
  res.send('JWT + Express + TypeScript Server');
});

app.listen(port, () => {
  console.log(
    `[server]: Server is running at http://localhost:${port} in ${env}`
  );
;
```

- 在 `./package.json` 裡寫一個啟動腳本：
```json
"scripts": {
  "dev": "NODE_ENV=development ts-node-dev --respawn src/app.ts"
}
```

- 啟動伺服器：
```bash
pnpm dev
```
並訪問 `http://localhost:3000` 就可以看到成功畫面。

## 使用 Docker Compose 設定資料庫

- 在專案目錄下新增一個檔案 `./docker-compose.yaml`：

```yaml
services:
  mongo:
    image: mongo:4.4.19-focal
    container_name: mongo
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGODB_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGODB_PASSWORD}
      MONGO_INITDB_DATABASE: ${MONGODB_DATABASE}
    env_file:
      - ./.env
    ports:
      - '6017:27017'
    volumes:
      - mongo:/data/db
      - ./init-mongo.sh:/docker-entrypoint-initdb.d/init-mongo.sh:ro
  redis:
    image: redis:latest
    container_name: redis
    ports:
      - '6379:6379'
    volumes:
      - redis:/data

volumes:
  mongo:
  redis:
```

 `docker-compose.yaml` 檔案會使用專案下 `.env` 裡的設定，所以要新增所需要的相關設定：

- `.env`
```json
PORT=3000

MONGODB_USERNAME=jwtweb
MONGODB_PASSWORD=pwd123
MONGODB_DATABASE=jwtAuth
```

當 MongoDB 建立後，它並不會自動建立 database。這就是 `./init-mongo.sh:/docker-entrypoint-initdb.d/init-mongo.sh:ro` 的功用，寫一個初始化資料庫的檔案並掛載到容器裡。

在專案目錄下新增一個檔案 `./init-mongo.sh`：

```shell
mongo << EOF

db = db.getSiblingDB('jwtAuth')

db.createUser({
  user: '${MONGODB_USERNAME}',
  pwd: '${MONGODB_PASSWORD}',
  roles: [
  {
    role: 'readWrite',
    db: '${MONGODB_DATABASE}',
  },
  ],
});

EOF
```

完成後使用下面指令來運行：

```bash
docker compose up -d
```

如果要停止的話可以使用：

```bash
docker compose down
# or
docker compose down -v # 停止並刪除 volume
```

## 連接到 MongoDB

- `./src/database/connectMongo`
```typescript
import mongoose from 'mongoose';

const db_user = process.env.MONGODB_USERNAME;
const db_pwd = process.env.MONGODB_PASSWORD;
const db_name = process.env.MONGODB_DATABASE;

mongoose.set('strictQuery', false);
const url = `mongodb://${db_user}:${db_pwd}@localhost:6017/${db_name}`;

const connectDB = async () => {
  try {
    await mongoose.connect(url);
    console.log('Database connected...');
  } catch (error: any) {
    console.log(error.message);
    setTimeout(connectDB, 5000);
  }
};

export default connectDB;
```

- 讓 `app.ts` 連接：
```typescript
import connectDB from './database/connectMongo';

//...
app.listen(port, () => {
  console.log(
    `[server]: Server is running at http://localhost:${port} in ${env}`
  );
  connectDB();
});
```


## 連接到 Redis

- `./src/database/connectRedis`
```typescript
import { createClient } from 'redis';

const redisUrl = 'redis://localhost:6379';
const redisClient = createClient({
  url: redisUrl,
});

const connectRedis = async () => {
  try {
    await redisClient.connect();
    console.log('Redis client connect...');
  } catch (err: any) {
    console.log(err.message);
    setTimeout(connectRedis, 5000);
  }
};

connectRedis();

redisClient.on('error', (err) => console.log(err));

export default redisClient;
```

因為 Redis 只會儲存 session 狀態，所以目前還不需要套用在任何地方。

> 可以到我的 [et860525/jwt-auth-node](https://github.com/et860525/jwt-auth-node) 來查看程式碼

## 結語

自從上一次發文後馬上就確診了，所以新的專案就停了一陣子。這次的專案是結合以前所學過的東西，來實作一個簡單的 JWT (JSON Web Tokens) RESTful API。

下一個章節會從建立資料庫的 model 開始。