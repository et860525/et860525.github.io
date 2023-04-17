---
title: "Node.js + JWT Authentication 專案(三) - 專案 Controller 與 Middleware"
date: 2023-04-17
draft: false
author: "Chen Yu Fan"
tags: ["Node.js", "TypeScript", "Express", "MongoDB", "Projects", "API"]
---

最後一章就要來完成整個專案，把剩下的 Controller、Middleware 與 Routes 完成即可。

<!--more-->

## Controller

### Authentication Controller

Authentication controller 包含所有關於身分驗證的機制。

建立檔案 `src/controllers/auth.controller.ts`：

```typescript
import { Request, Response, NextFunction, CookieOptions } from 'express';
import { CreateUserInput, LoginUserInput } from '../models/user.schema';
import { UserService } from '../services/user.service';
import DefaultError from '../exceptions/default.exception';

export default class AuthController {
  private accessTokenExpiresIn = process.env.ACCESS_TOKEN_EXPIRES_IN;
  private userService!: UserService;

  constructor() {
    this.userService = new UserService();
    // Set secure to true in production
    if (process.env.NODE_ENV === 'production')
      this.accessTokenCookieOptions.secure = true;
  }

  // Cookie options
  accessTokenCookieOptions: CookieOptions = {
    expires: new Date(
      Date.now() + Number(this.accessTokenExpiresIn) * 60 * 1000
    ),
    maxAge: Number(this.accessTokenExpiresIn) * 60 * 1000,
    httpOnly: true,
    sameSite: 'lax',
  };

  // Register
  public registerHandler = async (
    req: Request<{}, {}, CreateUserInput>,
    res: Response,
    next: NextFunction
  ) => {
    try {
      const user = await this.userService.createUser({
        email: req.body.email,
        name: req.body.name,
        password: req.body.password,
      });

      res.status(201).json({
        status: 'success',
        data: {
          user,
        },
      });
    } catch (err: any) {
      // MongoDB error code: Duplicate
      if (err.code === 11000) {
        return res.status(409).json({
          status: 'fail',
          message: 'Email already exist',
        });
      }
      next(err);
    }
  };

  // Login
  public loginHandler = async (
    req: Request<{}, {}, LoginUserInput>,
    res: Response,
    next: NextFunction
  ) => {
    try {
      const user = await this.userService.findUser({ email: req.body.email });

      if (
        !user ||
        !(await user.comparePasswords(user.password, req.body.password))
      ) {
        return next(new DefaultError('Invalid email or password', 401));
      }

      // Create an Access Token
      const accessToken = await this.userService.signToken(user);

      // Send Access Token in Cookie
      res.cookie('accessToken', accessToken, this.accessTokenCookieOptions);
      res.cookie('logged_in', true, {
        ...this.accessTokenCookieOptions,
        httpOnly: false,
      });

      // Send Access Token
      res.status(200).json({
        status: 'success',
        accessToken,
      });
    } catch (err: any) {
      next(err);
    }
  };
}
```

- `registerHandler` 使用者提供所需要的資訊來**註冊**
	- 錯誤代碼 `11000`：這是 MongoDB 回傳的錯誤代碼，表示該使用者已經存在。
- `loginHandler` 使用者提供 email 與 password 來進行**登入**，登入後會回傳給使用者一組 `Access Token` 

### User Controller

這個 Controller 可以讓登入的使用者使用。

建立 `src/controllers/user.controller.ts`：

```typescript
import { Request, Response, NextFunction } from 'express';
import { UserService } from '../services/user.service';

export default class UserController {
  private userService!: UserService;

  constructor() {
    this.userService = new UserService();
  }

  public getMeHandler = (req: Request, res: Response, next: NextFunction) => {
    try {
      const user = res.locals.user;
      res.status(200).json({
        status: 'success',
        data: {
          user,
        },
      });
    } catch (err: any) {
      next(err);
    }
  };

  public getAllUsersHandler = async (
    req: Request,
    res: Response,
    next: NextFunction
  ) => {
    try {
      const users = await this.userService.findAllUsers();
      res.status(200).json({
        status: 'success',
        result: users.length,
        data: {
          users,
        },
      });
    } catch (err: any) {
      next(err);
    }
  };
}
```

- `getMeHandler` 回傳目前登入的使用者個人資訊
- `getAllUsersHandler` 獲得所有使用者的資訊 ( 僅限 role 是 `admin` )

## Middleware

### Deserialize the User

這個 middleware 會獲得 header 或 cookie 裡的 JWT Authorization bearer token，驗證該 `token` 是否有效，並且也同時會驗證 user 的 session 是否還有效。

建立 `src/middleware/deserializeUser.ts`：

```typescript
import { Request, Response, NextFunction } from 'express';
import DefaultError from '../exceptions/default.exception';
import { verifyJwt } from '../middleware/jwt';
import redisClient from '../database/connectRedis';
import { UserService } from '../services/user.service';

const userService = new UserService();

export const deserializeUser = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    let access_token;
    if (
      req.headers.authorization &&
      req.headers.authorization.startsWith('Bearer')
    ) {
      access_token = req.headers.authorization.split(' ')[1];
    } else if (req.cookies.access_token) {
      access_token = req.cookies.access_token;
    }

    if (!access_token) {
      return next(new DefaultError('You are not logged in', 401));
    }

    // Validate Access Token
    const decoded = verifyJwt<{ sub: string }>(access_token);

    console.log(decoded);

    if (!decoded) {
      return next(new DefaultError(`Invalid token or user doesn't exist`, 401));
    }

    // Check if user has a valid session
    const session = await redisClient.get(decoded.sub);

    if (!session) {
      return next(new DefaultError(`User session has expired`, 401));
    }

    // Check if user still exist
    const user = await userService.findUserById(JSON.parse(session)._id);

    if (!user) {
      return next(
        new DefaultError(`User with that token no longer exist`, 401)
      );
    }

    // Help us know if user is logged
    res.locals.user = user;

    next();
  } catch (err: any) {
    next(err);
  }
};
```

### Check User Logged in Status

在 `Deserialize the User` 最後把 user 放到 `res.locals` 裡，所以我們可以由此來確認，user 目前是不是登入的狀態。

建立 `src/middleware/requireUser.ts`：

```typescript
import { Request, Response, NextFunction } from 'express';
import DefaultError from '../exceptions/default.exception';

/* Check if the user is logged in */
export const requireUser = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    const user = res.locals.user;
    if (!user) {
      return next(
        new DefaultError(`Invalid token or session has expired`, 401)
      );
    }

    next();
  } catch (err: any) {
    next(err);
  }
};
```

### Restrict Unauthorized Access

有些路徑與功能，是需要特定的身分組才能進行訪問與使用的。所以這裡會有一個 `allowedRoles` array 來設定可以訪問的身分組，如果有其他不再此 array 身分組的使用者進行訪問，則會拋出錯誤。

建立 `src/middleware/restrictTo.ts`：

```typescript
import { Request, Response, NextFunction } from 'express';
import DefaultError from '../exceptions/default.exception';

/* Restrict Unauthorized Access */
export const restrictTo =
  (...allowedRoles: string[]) =>
  (req: Request, res: Response, next: NextFunction) => {
    const user = res.locals.user;
    if (!allowedRoles.includes(user.role)) {
      return next(
        new DefaultError('You are not allowed to perform this action', 403)
      );
    }
    next();
  };
```

## Routes

### Authentication Routes

建立 `src/routes/auth.route.ts`：

```typescript
import express from 'express';
import { validate } from '../middleware/validate';
import { createUserSchema, loginUserSchema } from '../models/user.schema';
import AuthController from '../controllers/auth.controller';

const router = express.Router();
const authController = new AuthController();

// Register user route
router.post(
  '/register',
  validate(createUserSchema),
  authController.registerHandler
);

// Login user route
router.post('/login', validate(loginUserSchema), authController.loginHandler);

export default router;
```

### User Routes

建立 `src/routes/user.route.ts`：

```typescript
import express from 'express';
import { deserializeUser } from '../middleware/deserializeUser';
import { requireUser } from '../middleware/requireUser';
import { restrictTo } from '../middleware/restrictTo';
import UserController from '../controllers/user.controller';

const router = express.Router();
const userController = new UserController();

router.use(deserializeUser, requireUser);

// Admin get all Users
router.get('/', restrictTo('admin'), userController.getAllUsersHandler);

// Get my info
router.get('/me', userController.getMeHandler);

export default router;
```

## 更新 app.ts：加入 Routes 與完善功能

```typescript
import express, { Express, Request, Response, NextFunction } from 'express';
import dotenv from 'dotenv';
import morgan from 'morgan';
import cors from 'cors';
import cookieParser from 'cookie-parser';
import connectDB from './database/connectMongo';
import userRouter from './routers/user.route';
import authRouter from './routers/auth.route';

dotenv.config();

const app: Express = express();
const port = process.env.PORT;
const env = process.env.NODE_ENV;

// Body parser
app.use(express.json({}));

// Cookie parser
app.use(cookieParser());

// Logger
if (process.env.NODE_ENV === 'development') app.use(morgan('dev'));

// Cors
app.use(
  cors({
    origin: `http://localhost:${port}`,
    credentials: true,
  })
);

// Routes
app.use('/api/users', userRouter);
app.use('/api/auth', authRouter);

// Testing
app.get('/testChecker', (req: Request, res: Response, next: NextFunction) => {
  res.status(200).json({
    status: 'success',
    message: 'Welcome to JWT test',
  });
});

app.get('/', (req: Request, res: Response) => {
  res.send('JWT + Express + TypeScript Server');
});

// Unknown Routes
app.all('*', (req: Request, res: Response, next: NextFunction) => {
  const err = new Error(`Route ${req.originalUrl} not found`) as any;
  err.statusCode = 404;
  next(err);
});

// Error conrtol
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
  connectDB();
});
```

## 測試 JWT Authentication API

首先，先將執行資料庫：

```bash
docker compose up -d
```

運行伺服器：

```bash
pnpm dev
```

### 註冊

我使用 Chrome 線上商店的 [Talend API Tester](https://chrome.google.com/webstore/detail/talend-api-tester-free-ed/aejoelaoggembcahagimdiliamlcdmfm) 來測試這個 API。

到 `[POST]` http://localhost:3000/api/auth/register 進行註冊

![JWT-register.png](/images/JWT-authentication/JWT-register.png)

![JWT-register-response.png](/images/JWT-authentication/JWT-register-response.png)

這樣就表示註冊成功。

### 登入

到 `[POST]` http://localhost:3000/api/auth/login 進行登入

![JWT-login.png](/images/JWT-authentication/JWT-login.png)

![JWT-login-response.png](/images/JWT-authentication/JWT-login-response.png)

成功登入後，會回傳一個 `access_token`。

### 使用 Access Token 來獲得自己的資訊

到 `[GET]` http://localhost:3000/api/users/me 傳入 `access_token` 來獲的自己的資訊

![JWT-logged-in.png](/images/JWT-authentication/JWT-logged-in.png)

![JWT-logged-in-response.png](/images/JWT-authentication/JWT-logged-in-response.png)

剩下的 Get All Users 要先去資料庫更改 role 身分組再進行測試。

## 結語

在真實串接 API 的狀態下，就是用這組 `Access Token` 來進行身分驗證。以中央氣象局的[氣象資料開放平台](https://opendata.cwb.gov.tw/index)為例，當你註冊完後，就可以使用他給你的 API 授權碼來串接 API 並獲得資料。

![JWT-real-access-token.png](/images/JWT-authentication/JWT-real-access-token.png)

> 此專案的程式碼：[et860525/jwt-auth-node](https://github.com/et860525/jwt-auth-node)