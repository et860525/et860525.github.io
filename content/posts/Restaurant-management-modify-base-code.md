---
title: "Restaurant Management 專案(三) - 新增 Response Object 與完成 MenuItem"
date: 2023-03-16
draft: false
author: "Chen Yu Fan"
tags: ["TypeScript", "Projects", "Prisma", "Express"]
---

當我開始完善各個功能時，就發現回傳 view 所需要的函式不只有 `render`，在某些時候還是要使用 `redirect`，而當這樣會有兩種格式要回傳時，我就會建立 **Response Object** 來制定回傳的格式，並且在 `route.base` 中就要多一個專門處理 `redirect` 的函式。

當這些都完成後，就可以開始實作網頁的功能了，在這篇文章裡我會實作 `MenuItem` 的 CRUD。

Github：[et860525/restaurant-management](https://github.com/et860525/restaurant-management)

<!--more-->

## Response Object

首先，**Response object** 會將 Contoller 所回傳的資料做成統一的格式，並能讓 Route 使用。以下將兩個 Response object 放到 `src/commom/response/response.object.ts`：

```ts
export class renderResponseObject {
  public readonly view: string = 'index';
  public readonly data: any = null;

  constructor(options: { view: string; data?: any }) {
    this.view = options.view || this.view;
    this.data = options.data || this.data;
  }
}

export class redirectResponseObject {
  public readonly status: number = 302;
  public readonly url: string = '/';

  constructor(options: { url: string; status?: number }) {
    this.url = options.url || this.url;
    this.status = options.status || this.status;
  }
}
```


根據 `render` 與 `redirect` 會使用到的參數製作成 obejct，完成後再到 `controller.base` 新增相對應的函式來建立 Response object 並回傳：

```ts
import {
  renderResponseObject,
  redirectResponseObject,
} from '../common/response/response.object';

export abstract class ControllerBase {
  public formatResponse(view: string, data?: any) {
    const responseObject = new renderResponseObject({ view, data });

    return responseObject;
  }

  public formatRedirectResponse(url: string, status?: number) {
    const responseObject = new redirectResponseObject({ url, status });

    return responseObject;
  }
}
```

由 Route 會獲得從 Controll 回傳來的 Response object，所以要更改 `route.base` 的程式碼：

```ts
import { renderResponseObject, redirectResponseObject } from '../common/response/response.object';

export abstract class RouteBase {

  // 略...
  protected responseHandler(
    method: (
      req: Request,
      res: Response,
      next: NextFunction
    ) => Promise<renderResponseObject>
  ) {
    return (req: Request, res: Response, next: NextFunction) => {
      method
        .call(this.controller, req, res, next)
        .then((obj) => res.render(obj.view, obj.data))
        .catch((err) => next(err));
    };
  }

  protected responseRedirectHandler(
    method: (
      req: Request,
      res: Response,
      next: NextFunction
    ) => Promise<redirectResponseObject>
  ) {
    return (req: Request, res: Response, next: NextFunction) => {
      method
        .call(this.controller, req, res, next)
        .then((obj) => res.redirect(obj.status, obj.url))
        .catch((err) => next(err));
    };
  }
}
```

接下來就要開始完善 MenuItem 的各種功能。

## MenuItem Route

以下是讓使用者可以對菜單進行 CRUD 的動作，首先先新增路由，到 `main/menu/menuItem.routing.ts`：

```ts
import express from 'express';
import { RouteBase } from '../../base/route.base';
import { MenuItemController } from './menuItem.controller';

export class MenuItemRoute extends RouteBase {
  protected controller!: MenuItemController;

  constructor() {
    super();
  }

  protected initial(): void {
    this.controller = new MenuItemController();
    super.initial();
  }

  protected registerRoute(): void {
    this.router.get(
      '/menuItems',
      this.responseHandler(this.controller.get_many)
    );
    this.router
      .route('/menuItems/create')
      .get(this.responseHandler(this.controller.form))
      .post(
        express.json(),
        this.responseRedirectHandler(this.controller.create)
      );
    this.router.get(
      '/menuItems/:id',
      this.responseHandler(this.controller.get)
    );
    this.router
      .route('/menuItems/:id/delete')
      .get(this.responseHandler(this.controller.delete_get))
      .post(this.responseRedirectHandler(this.controller.delete));
    this.router
      .route('/menuItems/:id/update')
      .get(this.responseHandler(this.controller.update_get))
      .post(
        express.json(),
        this.responseRedirectHandler(this.controller.update)
      );
  }
}
```

- `/menuItems`：MenuItem 的主頁面
- `'/menuItems/:id'`：讀取指定 ID 的 MenuItem
- `'/menuItems/create'`：新增
- `'/menuItems/update'`：更新
- `'/menuItems/delete'`：刪除

## MenuItem Controller

接著開始實作各個路由的功能，到 `main/menu/menuItem.controller.ts`。

### 前置

```ts
import { MenuItem } from '@prisma/client';
import { Request } from 'express';
import { ControllerBase } from '../../base/controller.base';
import { MenuItemService } from './menuItem.service';

export class MenuItemController extends ControllerBase {
  private readonly menuService = new MenuItemService();
  // 實作的其他功能
}
```

- `menuService`：Controller 會把資料傳給 Service，並等待 Service 傳回從 Database 讀取來的資料

有些功能會在開始執行前，會先確定該筆資料是否存在於資料庫，如果沒有就會出現錯誤，所以掌握錯誤就很重要。建立一個檢查從資料庫回傳回來的資料是否為空的，如果是空的那就顯示錯誤的資訊：

```ts
 private checkMenuItem = async (id: string, menuItem: MenuItem | null) => {
    if (menuItem === null) {
      return this.formatResponse('error', {
        error: new Error(`MenuItem ${id} is not exist`),
      });
    } else {
      return menuItem;
    }
};
```

### Create

```ts
public async form() {
    return this.formatResponse('menuItem_form');
}

public async create(req: Request) {
	const { name, description, price } = req.body;
	const menuItem = await this.menuService.create(name, description, price);
	
	return this.formatRedirectResponse(`/menuItems/${menuItem.id}`);
}
```

### Read one

```ts
public async get(req: Request) {
    const { id } = req.params;
    const menuItem = await this.menuService.get(Number(id));

    this.checkMenuItem(id, menuItem);

    return this.formatResponse('menuItem_detail', { menuItem: menuItem });
  }
```

### Read many

```ts
public async get_many(req: Request) {
    const skip = req.query.skip || 0;
    const take = req.query.take || 10;

    const menuItems = await this.menuService.get_many(
      Number(skip),
      Number(take)
    );

    return this.formatResponse('menuItem', {
      menuItems: menuItems,
    });
}
```

### Update

```ts
public async update_get(req: Request) {
    const { id } = req.params;
    const menuItem = await this.menuService.get(Number(id));
    this.checkMenuItem(id, menuItem);
    
    return this.formatResponse('menuItem_form', { data: menuItem });
}

public async update(req: Request) {
	const { id } = req.params;
	const { name, description, price } = req.body;
	
	const menuItem = await this.menuService.update(Number(id), {
	  name: name,
	  description: description,
	  price: Number(price),
	});
	
	return this.formatRedirectResponse(`/menuItems/${menuItem.id}`);
}
```

### Delete

```ts
  public async delete_get(req: Request) {
    const { id } = req.params;
    const deleteUrl = req.path.split('/')[1];
    
    const menuItem = await this.menuService.get(Number(id));
    this.checkMenuItem(id, menuItem);
    
    return this.formatResponse('delete', {
      deleteItem: menuItem,
      deleteUrl: deleteUrl,
    });
}

  public async delete(req: Request) {
    const { id } = req.params;
    await this.menuService.delete(Number(id));
    return this.formatRedirectResponse('/menuItems');
}
```

## MenuItem Views

View 的部分就很簡單，只要把後端傳來的資料依據自己喜歡的方式放置即可，如需要參考可以到 [et860525/restaurant-management/views](https://github.com/et860525/restaurant-management/tree/main/views)。

## 結語

整個專案到這裡就告一段落了，剩下的其他 table 的功能基本上建立的方式都大致相同。這樣用 OOP 方式分開各個元件有幾個好處：

- 專責分類：根據功用分開才不會讓單一支程式負擔太大
- 清楚標示：分類後就能知道哪一個元件做哪一件事情，就不會 controller 還要獲得資料庫資料
- 測試方便：可以對單一個元件進行測試