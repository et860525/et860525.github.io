---
title: "Restaurant Management 專案(二) - 建立 Model 與 Repository"
date: 2023-03-14
draft: false
author: "Chen Yu Fan"
tags: ["TypeScript", "Projects", "Prisma", "Express"]
---

接下來就要設計 `Model` 與建立 `Repository` 來跟資料庫進行交互。

首先，設計 `Model` 的範本是來自 [Cheseto Restaurant POS App - Full Preview](https://dribbble.com/shots/20762377-Cheseto-Restaurant-POS-App-Full-Preview)，此範本包含四個 table：

1. **Table**：餐廳裡的桌子
2. **MenuItem**：菜單品項
3. **Order**：訂單
4. **Customer**：客人的資訊

完成 `Model` 後，先寫出 `repository.base` 再套用到各自的 table 上，以上。

Github：[et860525/restaurant-management](https://github.com/et860525/restaurant-management)

<!--more-->

## 設計 Model

![model.png](/images/Restaurant-management/model.png)

以上資料庫的重點為：

- 一個顧客 (Customer) 可以有多張訂單 (Order)
- 一張桌子 (Table) 可以有多張訂單 (Order)
- 訂單 (Order) 與 菜單品項 (MenuItem) 是**多對多 (many-to-many)** 的關係，而這裡使用 Explicit  many-to-many relations，原因是我想將菜單品項的總和與數量放在裡面

以下是 Prisma 的程式碼：

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Customer {
  id        Int     @id @default(autoincrement())
  name      String
  phone     String
  email     String @unique
  address   String
  orders    Order[]
}

model MenuItem {
  id          Int     @id @default(autoincrement())  
  name        String
  description String?
  price       Float 
  orders      OrderItem[]
}

model Order {
  id             Int          @id @default(autoincrement())
  customer       Customer     @relation(fields: [customerId], references: [id])
  customerId     Int
  table          Table        @relation(fields: [tableId], references: [id])
  tableId        Int
  items          OrderItem[]
  tag            String       @default(dbgenerated("lpad(nextval('order_tag_seq')::text, 6, '0')"))
  status         Order_status @default(PLACED)
  payment_method Payment      @default(Cash)     
  createdAt      DateTime     @default(now())
}

model OrderItem {
  order      Order    @relation(fields: [orderId], references: [id])
  orderId    Int
  menuItem   MenuItem @relation(fields: [menuItemId], references: [id])
  menuItemId Int
  quantity   Int
     
  @@id([orderId, menuItemId])
}

model Table {
  id         Int          @id @default(autoincrement())
  number     Int
  capacity   Int
  status     Table_status @default(Free)
  orders     Order[]
}

enum Order_status {
  PLACED
  IN_PROGRESS
  COMPLETED
  CANCELLED
}

enum Table_status {
  Free
  Checked_in
  Reserved
}

enum Payment {
  Cash
  Debit
  E_wallet
}
```

首先要注意的是 Order 裡面的 `tag`  欄位，它的作用是將 tag 顯示成 `000001`。 `@default(dbgenerated("lpad(nextval('order_tag_seq')::text, 6, '0')"))`：
- [dbgenerated()](https://www.prisma.io/docs/reference/api-reference/prisma-schema-reference#dbgenerated)：表示有些功能沒辦法再 Primsa 使用，但是在資料庫裡可以
- [lpad(string, length[, fill])](https://www.postgresqltutorial.com/postgresql-string-functions/postgresql-lpad/)：將指定字串的左邊填充指定字串到指定長度
- [nextval](https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-sequences/)：根據 `Sequences` 的 `INCREMENT` 來獲得下一個值

由於 `order_tag_seq` 它的欄位型別是 `SEQUENCE`，但是在 Primsa 是無法加入這個型別的欄位，所以這裡必須要先將 `migrate` 檔案產生出來後，再來改裡面的 SQL 語法。其步驟為：

1. 先輸入 `prisma migrate dev --create-only` ( 參考來源 [Customizing migrations](https://www.prisma.io/docs/concepts/components/prisma-migrate/migrate-development-production#customizing-migrations) )
2. 到剛剛產生的 migrate 檔案 (`prisma/migrations/20230310190516_test/migration.sql`) 下新增 `CREATE SEQUENCE order_tag_seq START 1;`
3. 再使用 `prisma migrate dev` 即可完成

如果沒有上面的步驟，而是直接 migrate 就會出現錯誤，因為找不到 `order_tag_seq` 這個欄位。

## Repository

如果是使用 Mongoose 的專案，為了滿足[Layered Architecture](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/ch01.html) 就要先建立一個 [Repository](https://github.com/et860525/express-project-architecture/tree/main/src/repositories) 來存放與資料庫交互的程式碼，並再由需要資料庫資料的相關程式碼來呼叫，如：[Service](https://github.com/et860525/express-project-architecture/blob/main/src/main/api/todo/todo.service.ts)。但在 Prisma 裡就會比較簡單，這是因為 [Prisma Client](https://www.prisma.io/docs/concepts/components/prisma-client) 本身就是一個 Object 直接使用即可，相關的資料格式可以在 `Service` 檔案裡做調整。

### 嘗試寫一個 Repository Base

在使用 Prisma Client 時，我一直在嘗試讓程式碼更簡潔。因為我有四個 tables 就要建立四個 `Repository` 的檔案，我有想過讓每一個 table 都繼承一個 `Repository Base` 物件，並讓這個 `Repository Base` 直接實作所有獲取資料的方式，以下只是思考的程式碼並不能執行：

```ts
import { PrismaClient, MenuItem, Order, Customer, Table } from '@prisma/client';

export abstract class RepositoryBase {
	protected modelName: string = '';
	private prisma = new PrismaClient();

	public async get(id: number): Promise<MenuItem | Order | Customer | Table | null> {
		return await this.[modelName].findUnique({
			where: { id: id },
		})
	}
}
```

雖然這個樣子可以少寫程式碼，但是在有些 table 有 relations 時，可能就會需要使用到 `include`。如果是在 `create` 的情況下，就要在相應的 `Repository` 下重寫整個 `create` 程式碼，那這樣還不如就直接根據相對應的 table，來寫相對應的程式碼就好了。

> 我有試過使用 `switch`，但是那個可讀性還不如寫在各自的 `Repository` 檔案裡

### 實作

上面有說到，直接使用 Prisma Client 物件就可以了，所以這裡我會直接在 `Service` 檔案裡直接使用 Prisma Client：

```ts
import { Prisma, PrismaClient, MenuItem } from '@prisma/client';

export class MenuItemService {
  private readonly prisma = new PrismaClient();

  public async get(id: number): Promise<MenuItem | null> {
    return await this.prisma.menuItem.findUnique({
      where: { id: id },
    });
  }

  public async get_many(
    skip: number,
    take: number
  ): Promise<MenuItem[] | null> {
    return await this.prisma.menuItem.findMany({
      skip: skip,
      take: take,
    });
  }

  public async create(
    name: string,
    description: string,
    price: string
  ): Promise<MenuItem> {
    return await this.prisma.menuItem.create({
      data: {
        name: name,
        description: description,
        price: Number(price),
      },
    });
  }

  public async delete(id: number): Promise<MenuItem> {
    return await this.prisma.menuItem.delete({
      where: { id: id },
    });
  }
}
```

`Controller` 會將獲得的參數 ( `req` 或 `Controller` 本身的參數 ) 送到 `Service` 裡，`Service` 會使用 Prisma Client 來跟資料庫交互，最後獲得的結果再回傳給 `Controller`。

> 目前我只完成 `MenuItem` table，後面三個參數我會慢慢完成，完成後，就會開始把前端的部分完成

## 結語

因為[官網文件](https://www.prisma.io/docs/concepts/components/prisma-schema)很完整，所以設計 `Model` 的部分就沒有那麼多的問題。但是在設計 `Repository` 的部分就卡的比較久一點，就像上面嘗試的部分撰寫的一樣，我一直很想少寫這些近乎重複的程式碼，但也就是這些微的不同就會影響資料顯示的不同，但我還是想先把它們都寫出來後，再來慢慢的修改。
