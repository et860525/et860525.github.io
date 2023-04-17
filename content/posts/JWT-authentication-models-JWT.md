---
title: "Node.js + JWT Authentication 專案(二) - 資料庫 Models 與 JWT"
date: 2023-04-16
draft: false
author: "Chen Yu Fan"
tags: ["Node.js", "TypeScript", "Express", "MongoDB", "Projects", "API"]
---

此篇章會使用 [Typegoose](https://typegoose.github.io/typegoose/) 來建立資料庫的 models，並且設定獲得與驗證 JWT 的方法。

<!--more-->

## 建立資料庫欄位

資料庫會包含四個欄位：

- name
- email
- password
- role

建立檔案 `src/models/user/user.model.ts`：

```typescript
import {
  DocumentType,
  getModelForClass,
  index,
  modelOptions,
  pre,
  prop,
} from '@typegoose/typegoose';
import bcrypt from 'bcryptjs';

// Make email field have index
@index({ email: 1 })
@pre<User>('save', async function () {
  // Hash password if new or update
  if (!this.isModified('password')) return;
  // Hash password
  this.password = await bcrypt.hash(this.password, 12);
})
@modelOptions({
  schemaOptions: {
    // Add createAt and updateAt fields
    timestamps: true,
  },
})

/* User class */
export class User {
  @prop({ required: true })
  public name!: string;

  @prop({ unique: true, required: true })
  public email!: string;

  @prop({ required: true, minlength: 8, maxlength: 32, select: false })
  password!: string;

  @prop({ default: 'user' })
  role!: string;

  // Check password match or not
  async comparePasswords(hashPassword: string, password: string) {
    return await bcrypt.compare(password, hashPassword);
  }
}

// Create the user model
const userModel = getModelForClass(User);
export default userModel;
```

- `@index({ email: 1 })`：給 email 欄位建立索引，其功用為提升查詢效率
- `@pre`：可以設定當欄位的資料在執行某些動作時，會做的事情
- `@modelOptions`：新增額外的欄位
- `@prop`：設定欄位相關的設定
- `bcrypt` 會將密碼加鹽(加密)

## 產生公鑰與私鑰

要產生 JWT 憑證前，要先有公鑰與私鑰來做加密與解密。這邊可以借助網路的力量，直接找尋產生 RSA Key 的網站，可以到 [devglan.com/rsa-decryption](https://www.devglan.com/online-tools/rsa-encryption-decryption) 來產生。

可以自行選擇 RSA Key Size，這裡我使用 2048 bit。按下 Generate RSA Key Pair 可以得到已經轉換成 Base64 編碼的 Public Key 與 Private Key。複製並放到 `.env` 檔案裡：

```env
PORT=3000
MONGODB_USERNAME=jwtweb
MONGODB_PASSWORD=pwd123
MONGODB_DATABASE=jwtAuth

ACCESS_TOKEN_EXPIRES_IN=15
ACCESS_TOKEN_PRIVATE_KEY=MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQCeBN6jGxOdRFdgoryT+Lm2o+HB8M6xfBHaW/I2UYdu1cQdEgEpcApSVJtLQB+aIJ78kDwOexjwczVjsUmzVZBArHXEz9VTvH86v5hr7fsHRo44LRDpaN1CK6P/2e4rmgTsm8/3aKpsblsnUI5r9ta7GhIwuppOXZYcjyS+6dtdAc9yKm8xY2X/bCzfU7bXdbo5MbhbglYvMQyfVBMkr4pvrThD98O7qPcHuA9kTjLBMfPoFxoSvNB9XQGKu7cG2AvA/smL9Q/LR0gAQ2KT98io+OcklNWgu4i4RFuceV1EiOl0pPHlnVjdEYa9ZpMh1IG1hemiKGaDNDUjPPpF7+xFAgMBAAECggEAUWIerBBw7Kla+yk1SFxsgXUr+2+jdGN66mQ6feFFiD7OT06LjKTom/h5Nqti20V7vIYoeCjL8mLTl3GijJs/vR9VVDTaINNPD5nHzaZ2iAu9iY8kS6I3ejHxt/6snIYpjRa+aCTeyROZHMlvYIlzlE9cGP6yJDQs8K6EdVMKKH7KZa2u+1cU/6yBpY7Gf5hyWzzhD0yM2uwasqJ6iF+4k8vBqSRz3ITFjYF2tfGnLKwZ1ZY9QSIO2TrCRiuNkYYAWRs36SPF6ITh2s1y4KZMOeEH7ljeEAuWX0qjQzP96x3SVc/Y3n6hO0rvtCGoSSJoN8cWsGGIOPwylQ/8dIbCzQKBgQDj6eDKsai9pmYzJ6QeY5RLtAcAq0sGbZ5wDkdUj+U4weXQjZCP9qP8MKTzqmyYUQKxKIjlgPV5M+z2F9aadoPHDRk7OtcJx5facE/xpB4eTUVj31LDI/z+raGIhV3XLxxTTR6YXBaBV2I/H0nMwqx4FsNsQeByLZ1+Cd/HqoaR9wKBgQCxffy+7Fa+iMd4FSOZpfBT+//LZi3Uaco1fMw8pu43h/XECh6/UyftwTdI52iH5EGBVog+wKvcx5gE0/xQiEM0RhtKxDjjMRKT1BxS+EKZHgWjqT8JCaMG6+Nq6w/02spQAB1C3VEN+sXpdoaujOLAGiJ328z9YPRo9DIVWYfkowKBgQCh5rMb6eZfioQRFLjeKYjf2iwbSpNKJralDU+Yf3uqvPqfEuE9k0xcSsXynf70mJ+b75qHxfsatUtAaiC1qzjjPqfMzniRZuq1bpErq5UFm4iOcMce/kKrO/aCv5Kw2LN7bU4tl0UZblTJWFWZkjToPetmzMk+8q5tKWCBOt7LcwKBgCl5eR/b9gEb0RB8UA9NOTVGw2TyAW+LMNcCzG63yx5qxMEEZF7svX3PEm4UtNZcPfpNEBUpzH8QnLM0Hddrn9iNMT9tTqW4B9FHVT8GB/njjAnMOJCSEehCIqgPOXFL1s6O2EeRk6kimjCNo7cR8MJW2QsM73+dsj78IN/gReLlAoGBAMYaVug98ZA+ZU8YGnwhmlgpTNxhwvpkifMf//5GxO7K30g6gSOVUkbYq7oVdTgO1/Fgn5PJ17gh+GWTrt1nWcmJy0oyguvxSjODvQWjPlmkpBcQm2M5LUsO5snfHyIA/S977bdyKgb7h9J4j/UsIoa+aa7pyIJDiltydM1Qr9wE
ACCESS_TOKEN_PUBLIC_KEY=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAngTeoxsTnURXYKK8k/i5tqPhwfDOsXwR2lvyNlGHbtXEHRIBKXAKUlSbS0AfmiCe/JA8DnsY8HM1Y7FJs1WQQKx1xM/VU7x/Or+Ya+37B0aOOC0Q6WjdQiuj/9nuK5oE7JvP92iqbG5bJ1COa/bWuxoSMLqaTl2WHI8kvunbXQHPcipvMWNl/2ws31O213W6OTG4W4JWLzEMn1QTJK+Kb604Q/fDu6j3B7gPZE4ywTHz6BcaErzQfV0Biru3BtgLwP7Ji/UPy0dIAENik/fIqPjnJJTVoLuIuERbnHldRIjpdKTx5Z1Y3RGGvWaTIdSBtYXpoihmgzQ1Izz6Re/sRQIDAQAB
```

這邊還多加入了一個 `ACCESS_TOKEN_EXPIRES_IN=15`，這是設定 JWT 存在的時間。

## 建立 JWT 中間件( Middleware )

JWT 中間件可以透過上面所設定的**私鑰**來獲得 JWT，並使用**公鑰**來驗證使用者傳來的 JWT。

新增檔案 `./src/middleware/jwt.ts`：

```typescript
import jwt, { SignOptions } from 'jsonwebtoken';

const accessTokenPrivateKey = process.env.ACCESS_TOKEN_PRIVATE_KEY;
const accessTokenPublicKey = process.env.ACCESS_TOKEN_PUBLIC_KEY;

export function signJwt(payload: Object, options: SignOptions) {
  if (!accessTokenPrivateKey) return null;
  const privateKey = Buffer.from(accessTokenPrivateKey, 'base64').toString(
    'ascii'
  );
  return jwt.sign(payload, privateKey, {
    ...options,
    algorithm: 'RS256',
    allowInsecureKeySizes: true,
  });
}

export function verifyJwt<T>(token: string): T | null {
  try {
    if (!accessTokenPublicKey) return null;
    const publicKey = Buffer.from(accessTokenPublicKey, 'base64').toString(
      'ascii'
    );
    return jwt.verify(token, publicKey) as T;
  } catch (error) {
    console.log(error);
    return null;
  }
}
```

- `signJwt`：負責獲得 JWT
- `verifyJwt`：負責驗證 JWT 是否有效，無效則會回傳 `null`

> `allowInsecureKeySizes: true`：會加上這個設定是因為我遇到了錯誤 `secretOrPrivateKey has a minimum key size of 2048 bits for RS256`，可以參考 [secretOrPrivateKey size error when size is larger enough #888](https://github.com/auth0/node-jsonwebtoken/issues/888)

## 處理錯誤

當用戶輸入的資料出現錯誤時，就會回傳一個錯誤 class，裡面會包含**錯誤的訊息**與狀態碼 ( status code )。

建立檔案 `./src/exceptions/default.exception.ts` 來自定義錯誤：

```typescript
export default class DefaultError extends Error {
  status!: string;

  constructor(message: string, statusCode: number = 500) {
    super(message);
    this.status = statusCode.toString().startsWith('4') ? 'fail' : 'error';
    
    Error.captureStackTrace(this, this.constructor);
  }
}
```

- 當 `statusCode` 的開頭是 **4** 時就都是錯誤。

接著將以下程式碼加入到 `.src/app.ts`：

```typescript
//...
app.use((err: any, req: Request, res: Response, next: NextFunction) => {
  err.status = err.status || 'error';
  err.statusCode = err.statusCode || 500;

  res.status(err.statusCode).json({
    status: err.status,
    message: err.message,
  });
});

app.listen(port, () => {
  console.log(
    `[server]: Server is running at http://localhost:${port} in ${env}`
  );
  mongoDB.connectDB();
});
```

這樣當錯誤發生時，就會回傳給使用者相關的錯誤資訊。

## Zod 驗證

**永遠不要相信使用者傳入的資訊**，這個是後端應用的鐵則，所以驗證使用者所輸入的資料是很重要的。雖然 Mongoose 本身就帶有驗證的機制，但這裡可以使用更加嚴謹的 Zod 驗證。

> 為什麼要使用 Zod？直接使用 TypeScript 來驗證外部傳進來的資料不就可以了嗎？
> 其實 TypeScript 並不會真的去驗證從外部傳進來的型別，這個我們留到專案結束後再來說。

建立檔案 `./src/models/user/user.schema.ts`：

```typescript
import { z } from 'zod';

export const createUserSchema = z.object({
  body: z
    .object({
      name: z.string({ required_error: 'Name is required' }),
      email: z
        .string({ required_error: 'Email is required' })
        .email('Invalid email'),
      password: z
        .string({ required_error: 'Password is required' })
        .min(8, 'Password must be more than 8 characters')
        .max(32, 'Password must be less than 32 characters'),
      passwordConfirm: z.string({
        required_error: 'Please confirm your password',
      }),
    })
    .refine((data) => data.password === data.passwordConfirm, {
      path: ['passwordConfirm'],
      message: 'Password do not match',
    }),
});

export const loginUserSchema = z.object({
  body: z.object({
    email: z
      .string({ required_error: 'Email is required' })
      .email('Invalid email or password'),
    password: z
      .string({ required_error: 'Password is required' })
      .min(8, 'Invalid email or password'),
  }),
});

export type CreateUserInput = z.infer<typeof createUserSchema>['body'];
export type LoginUserInput = z.infer<typeof loginUserSchema>['body'];
```

- `createUserSchema` 用在**使用者註冊**
- `loginUserSchema` 用在**使用者登入**

### 驗證使用者資料的中間件

那該如何使用上面所設定的 Schema 來做驗證？這邊來建立一個 middleware 來處理，它會根據傳入的 Schema (上面所設定的 `createUserSchema` 或 `loginUserSchema`) 來驗證使用者所傳入的資料。

建立 `src/middleware/validate.ts`：

```typescript
import { Request, Response, NextFunction } from 'express';
import { z } from 'zod';

// This function is for validating the use input based on the schema
export const validate =
  (schema: z.AnyZodObject) =>
  (req: Request, res: Response, next: NextFunction) => {
    try {
      schema.parse({
        params: req.params,
        query: req.query,
        body: req.body,
      });
      next();
    } catch (err: any) {
      if (err instanceof z.ZodError) {
        return res.status(400).json({
          status: 'fail',
          error: err.errors,
        });
      }
      next(err);
    }
  };
```

## 與 Database 溝通的 Service

與 Database 溝通的功能通常都會獨立出來，並做成一個 Service。如果直接寫在 Controller 裡，往後會很難擴充並重複使用。

建立檔案 `src/services/user.service.ts`：

```typescript
import { omit } from 'lodash';
import userModel, { User } from '../models/user/user.model';
import { FilterQuery, QueryOptions } from 'mongoose';
import { DocumentType } from '@typegoose/typegoose';
import { signJwt } from '../middleware/jwt';
import Redis from '../database/connectRedis';

export default class UserService {
  private accessTokenExpiresIn = process.env.ACCESS_TOKEN_EXPIRES_IN;
  private excludedFields = ['password'];
  private redisDB = new Redis();

  // Create user
  public async createUser(input: Partial<User>) {
    const user = await userModel.create(input);
    return omit(user.toJSON(), this.excludedFields);
  }

  // Find user by id
  public async findUserById(id: string) {
    const user = await userModel.findById(id).lean();
    return omit(user, this.excludedFields);
  }

  // Find all users
  public async findAllUsers() {
    return await userModel.find();
  }

  // Find one users by any fields
  public async findUser(query: FilterQuery<User>, options: QueryOptions = {}) {
    return await userModel.findOne(query, {}, options).select('+password');
  }

  // Sign token
  public async signToken(user: DocumentType<User>) {
    // Sign the access token
    const access_token = signJwt(
      {
        sub: user._id,
      },
      {
        // Expires: 15m
        expiresIn: `${Number(this.accessTokenExpiresIn)}m`,
      }
    );
    
    // Create a session
    this.redisDB.redisClient.set(String(user._id), JSON.stringify(user), {
      EX: 60 * 60,
    });
    
    return { access_token };
  }
}
```

- `createUser` 建立使用者
- `findUserById` 根據 ID 找到使用者
- `findAllUsers` 找尋所有使用者
- `findUser` 根據回傳的欄位與相對的資料，找尋使用者
- `signToken` 根據傳入的 `Access Token`，來驗證登入的狀態
- `excludedFields` 這個會將密碼欄位排除，不會回傳

## 結語

以前都是使用 Mongoose 本身的驗證機制來過濾資料，這是第一次使用 Zod 來實作更加嚴謹的驗證。

下一章就是此專案的最後，目前就剩下專案的 Controller 與 Router 的實作，還有一些 middleware 就可以完成整個專案了。 

> 此專案的程式碼：[et860525/jwt-auth-node](https://github.com/et860525/jwt-auth-node)