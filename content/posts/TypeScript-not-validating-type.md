---
title: "TypeScript 不會真的驗證你資料的型別"
date: 2023-05-08
draft: false
author: "Chen Yu Fan"
tags: ["TypeScript"]
---

在 [JWT-authentication-models-JWT](/posts/jwt-authentication-models-jwt/#zod-%e9%a9%97%e8%ad%89) 裡有提到 Zod 驗證使用者輸入的資料，這一篇會提到為什麼需要使用 Zod 來驗證。

<!--more-->

TypeScript 需要你定義型別，強制你去思考放進去變數或函式的資料會是甚麼。但好玩的是，TypeScript 其實不會真的驗證你資料的型別。

因為 TypeScript 只在編譯層運作，而不是在運行時。如果對比一下 TypeScript 的程式碼與它轉成的 JavaScript 程式碼：

- TypeScript 程式碼
```typescript
const justAFunction = (n: number): string => {
	return `${n}`
}

console.log(justAFunction)
```
- JavaScript 程式碼
```javascript
"use strict";
const justAFunction = (n) => {
		return `${n}`;
};
console.log(justAFunction);
```

由上述可知，**它只根據你程式碼的來源來檢查型別是否正確**，所以它並不是真的驗證真實的資料。

## 檢查型別

TypeScript 只要使用得當，還是會強制你檢查不確定的型別。

將上面的範例進行修改：

```typescript
const justAFunction = (str: string[] | string): string => {
  return str.join(' ')
}

console.log(justAFunction(["Hello", "World"]))
console.log(justAFunction("Hello World"))
```

當編譯時會產生錯誤：

```text
index.ts:2:14 - error TS2339: Property 'join' does not exist on type 'string | string[]'.
  Property 'join' does not exist on type 'string'.

2   return str.join(' ')
               ~~~~

Found 1 error in index.ts:2
```

這是因為編譯器認定 `str` 是 `string` 型別時，它並沒有 `join()` 這個方式。

這個時候有兩個解決方法，第一個就是把 `string` 拿掉，只留下 `string[]`，另一個就是驗證變數的的型別：

```typescript
const justAFunction = (str: string[] | string): string => {
  if (typeof str === 'string') {
    return str
  }

  return str.join(' ')
}

console.log(justAFunction(["Hello", "World"]))
console.log(justAFunction("Hello World"))
```

它也會轉譯成 Javascript 並且被驗證型別。

## 對照外部資料的型別

假設一個 API 回傳一個使用者：

```json
{
  "firstname": "John",
  "lastname": "Doe",
  "birthday": "1985-04-03"
}
```

我們要為這份資料建立一個 interface：

```typescript
interface User {
  firstname: string
  lastname: string
  birthday: string
}
```

`fetch` 這個 API 來取回使用者資料：

```typescript
const retrieveUser = async (): Promise<User> => {
  const resp = await fetch('/user/me')
  return resp.json()
}
```

這樣看起來沒什麼問題，但假設 `birthday` 是 `timestamp` 型別，TypeScript 依舊會將這個型別視為 `string`，雖然有數字在裡面，但是 TypeScript 並不會檢查它真實的值。

所以老方法就是，寫一個驗證的函式：

```typescript
const validate = (obj: any): obj is User => {
  return obj !== null
    && typeof obj === 'object'
    && 'firstname' in obj
    && 'lastname' in obj
    && 'birthday' in obj
    && typeof obj.firstname === 'string'
    && typeof obj.lastname === 'string'
    && typeof obj.birthday === 'string'
}

const user = await retrieveUser()

if (!validate(user)) {
  throw Error("User data is invalid")
}
```

這樣做確實就可以保證資料的型別，但是遇到更複雜的 API 那就會寫到死。

# Zod

[Zod](https://zod.dev/) 就是讓 TypeScript 型別強制驗證 Javascript 裡的型別。它允許你定義 schema、推斷型別、驗證資料。

讓 User 使用 Zod：

```typescript
import { z } from 'zod'

const User = z.object({
  firstname: z.string(),
  lastname: z.string(),
  birthday: z.string()
})
```

把該 schema 提取出來成為一個類型：

```typescript
const UserType = z.infer<User>
```

驗證的方式會像：

```typescript
const userResp = await retrieveUser()
const user = User.parse(userResp)
```

現在就能獲得被驗證過的資料。

---

# Reference

- [Zod](https://zod.dev/)
- [Typescript: It's not actually validating your types.](https://dev.to/syeo66/typescript-its-not-actually-validating-your-types-1mn3)