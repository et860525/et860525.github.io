---
title: "Restaurant Management 專案(一) - 架構與初始化"
date: 2023-03-10
draft: false
author: "Chen Yu Fan"
tags: ["TypeScript", "Projects", "Prisma", "Express"]
---

這個專案會使用 OOP 的方式來建構，前端部分會以簡單的方式呈現。

整個架構會用到的重要套件：

| Package                                       | Usage        |
| --------------------------------------------- | ------------ |
| [Express](https://expressjs.com/)             | Web 應用框架 |
| [TypeScript](https://www.typescriptlang.org/) | 開發工具     |
| [Prisma](https://www.prisma.io/)              | 訪問資料庫   |
| [Docker](https://www.docker.com/)             | 應用容器化   |
| [PostgreSQL](https://www.postgresql.org/)     | 資料庫       |

此專案的目的是要讓 Express 使用 Prisma 來訪問資料庫，並且使用 Docker 來建立 PostgreSQL 資料庫。

Github：[et860525/restaurant-management](https://github.com/et860525/restaurant-management)

<!--more-->

## 專案架構

整個專案的架構大致會遵循 [et860525/express-project-architecture](https://github.com/et860525/express-project-architecture)。

最大的不同就是資料庫的部分，因為這一個專案使用的是 Prisma，所以我不需要先做連接資料庫的動作。建立 `model` 的地方會改成 `prisma/schema.prisma`。

這裡我會先建立一個簡單的 `model` 來做測試：

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Restaurant {
  id        Int      @id @default(autoincrement())
  name      String
  address   String?
  phone     String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

完成後使用 `pnpx prisma migrate dev --name init` 來將 `model`  migrate 進資料庫。

## 訪問資料庫

> 將資料庫部屬在 Docker 裡，輸入 `docker compose up -d`。因為我有在設定裡寫 `restart: always`，所以第一次部屬後，往後只要重新運行 Docker 就會自動啟動。

Prisma 提供 [Prisma Client](https://www.prisma.io/docs/concepts/components/prisma-client) 來訪問資料庫進而獲取資料。

我把訪問資料庫的程式碼寫在 `src/repositories/restaurant.repository.ts`：

```ts
import { PrismaClient } from '@prisma/client';

export class RestaurantRepository {
  private prisma = new PrismaClient();

  public async addRestaurant(name: string, address: string, phone: string) {
    const result = await this.prisma.restaurant.create({
      data: {
        name,
        address,
        phone,
      },
    });
    
    return result;
  }

  public async getRestaurant(id: string) {
    const restaurant = await this.prisma.restaurant.findUnique({
      where: {
        id: Number(id),
      },
    });
    return restaurant;
  }

  public async getRestaurants(skip: number = 0, take: number = 10) {
    const take_og = 10;
    const restaurants = await this.prisma.restaurant.findMany({
      skip: skip,
      take: Math.min(take, take_og),
    });
    return restaurants;
  }
}
```

> 這裡很多東西都沒有寫得很完整，像是回傳型別、命名有冗詞之類的，不過目前只是為了方便測試我就沒有寫得太詳細，往後會寫的更加完整。

## Controller 與 Route

通常只要能把訪問資料庫的部分做出來，在 Controller 與 Route 的部分我覺得就比較輕鬆了。不過這跟上一個 API 專案不同，這次有用到 `view` 來顯示前端，所以我必須要先更改 `src/base/controller.base.ts` 與 `src/base/route.base.ts`：
- `src/base/route.base.ts`
	```ts
	import { Router, Request, Response, NextFunction } from 'express';
	import { ControllerBase } from './controller.base';
	
	export abstract class RouteBase {
	  public router: Router = Router();
	  protected controller!: ControllerBase;
	
	  constructor() {
	    this.initial();
	  }
	
	  protected initial(): void {
	    this.registerRoute();
	  }
	
	  protected abstract registerRoute(): void;
	
	  protected responseHandler(
	    method: (req: Request, res: Response, next: NextFunction) => Promise<any>
	  ) {
	    return (req: Request, res: Response, next: NextFunction) => {
	      method
	        .call(this.controller, req, res, next)
	        .then((obj) => res.render(obj.template, obj.data))
	        .catch((err) => next(err));
	    };
	  }
	}
	```
	`responseHandler` 會呼叫 controller 丟進去的 `method`，當他完成後會回傳 template 的名字與  template 要使用的資料。
- `src/base/controller.base.ts`
	```ts
	export abstract class ControllerBase {
	  public formatResponse(template: string, data?: any) {
	    const responseObject = { template: template, data: data };
	    
	    return responseObject;
	  }
	}
	```
	只需要將 template 的名字與給 template 使用的資料丟回即可。

### 實際使用

- `src/main/restaurant/restaurant.controller.ts`
	```ts
	import { Request } from 'express';
	import { ControllerBase } from '../../base/controller.base';
	import { RestaurantRepository } from '../../repositories/restaurant.repository';
	
	export class RestaurantController extends ControllerBase {
	  private readonly restaurantRepo = new RestaurantRepository();
	
	  public async addRestaurant_get() {
	    return this.formatResponse('restaurant_form');
	  }
	
	  public async addRestaurant(req: Request) {
	    const { name, address, phone } = req.body;
	    console.log(req.body);
	
	    const result = await this.restaurantRepo.addRestaurant(
	      name,
	      address,
	      phone
	    );
	    console.log(result);
	
	    return this.formatResponse('restaurant_form', { result: result });
	  }
	
	  public async getRestaurant(req: Request) {
	    const { id } = req.params;
	    const restaurant = await this.restaurantRepo.getRestaurant(id);
	    return this.formatResponse('restaurant', { restaurant: restaurant });
	  }
	
	  public async getRestaurants(req: Request) {
	    const skip = req.query.skip || 0;
	    const take = req.query.take || 10;
	
	    const restaurants = await this.restaurantRepo.getRestaurants(
	      Number(skip),
	      Number(take)
	    );
	    return this.formatResponse('restaurant_list', {
	      restaurants: restaurants,
	    });
	  }
	}
	```
- `src/main/restaurant/restaurant.routing.ts`
	```ts
	import express, { Request, Response, NextFunction } from 'express';
	import { RouteBase } from '../../base/route.base';
	import { RestaurantController } from './restaurant.controller';
	
	export class RestaurantRoute extends RouteBase {
	  protected controller!: RestaurantController;
	
	  constructor() {
	    super();
	  }
	
	  protected initial(): void {
	    this.controller = new RestaurantController();
	    super.initial();
	  }
	
	  protected registerRoute(): void {
	    this.router.get('/', (req: Request, res: Response, next: NextFunction) => {
	      res.render('index');
	    });
	    this.router.get(
	      '/restaurant',
	      this.responseHandler(this.controller.getRestaurants)
	    );
	    this.router
	      .route('/restaurant/create')
	      .get(this.responseHandler(this.controller.addRestaurant_get))
	      .post(
	        express.json(),
	        this.responseHandler(this.controller.addRestaurant)
	      );
	    this.router.get(
	      '/restaurant/:id',
	      this.responseHandler(this.controller.getRestaurant)
	    );
	  }
	}
	```

目前只有實作：
- **新增 (add)**
	- `GET` 顯示表格
	- `POST` 新增資料
- **單個查詢 (getRestaurant)**
- **多個查詢 (getRestaurants)**

這樣整個測試的專案就完成了。

## 結語

我是第一次使用 Prisma 來訪問資料庫，以前都是使用 [Mongoose ODM](https://mongoosejs.com/) 來操作資料庫的部分。

接下來我會開始將專案改成實際使用狀態，所以現在的 `model` 我還會再做修改。