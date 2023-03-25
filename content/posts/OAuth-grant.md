---
title: "OAuth Grant"
date: 2023-03-25
draft: false
author: "Chen Yu Fan"
tags: ["Backend", "Authentication", "OAuth 2.0"]
---

> OAuth 2.0 最大的優勢是可擴展性與模塊化，但這樣的靈活性也導致於在不同的實現之間，會存在著相容性問題。當開發人員想在不同的系統上實現 OAuth 時，它提供很多的自定義選項容易讓人很困惑。

OAuth 2.0 一共定義了 7 種授權類型，可以根據不同的情況與環境使用不同的模式：

- Legacy: [密碼模式 ( Password Grant )](https://oauth.net/2/grant-types/password/)
- Legacy: [隱含模式 ( Implicit Flow )](https://oauth.net/2/grant-types/implicit/)
- [授權碼 ( Authorization Code )](https://oauth.net/2/grant-types/authorization-code/)
- [刷新令牌 ( Refresh Token )](https://oauth.net/2/grant-types/refresh-token/)
- [客戶憑證 ( Client Credentials )](https://oauth.net/2/grant-types/client-credentials/)
- [PKCE ( Proof Key for Code Exchange )](https://oauth.net/2/pkce/)
- [設備碼 ( Device Code )](https://oauth.net/2/grant-types/device-code/)

---

起初，OAuth 設計是基於 HTTP 的，但實現方法的細節可以有很多種。

在上一篇有提到，OAuth 裡定義的[四種角色](/posts/oauth-first-look/)。其中的客戶端還可以分為兩種：

- **前端客戶端**：通常前端客戶端指的是瀏覽器
- **後端客戶端**：後端客戶端指的是，實際需要取得**存取權杖 ( Access Token )** 的服務

通常的流程為：

![OAuth-work-flows.png](/images/OAuth/OAuth-work-flows.png)

1. **資源擁有者** 透過瀏覽器登入 ( 此步驟意同瀏覽器向資源擁有者授權請求 )
2. **授權伺服器** 驗證身分並確認授權
3. 授權給**客戶端** ( 獲得存取權杖 )
4. **客戶端**取得**受保護的資源**
5. **客戶端**提供**資源擁有者**服務

以上流程為 [1.2. Protocol Flow](https://www.rfc-editor.org/rfc/rfc6749#section-1.2)。

## Legacy: 密碼模式 ( Password Grant )

```flow
     +----------+
     | Resource |
     |  Owner   |
     |          |
     +----------+
          v
          |    Resource Owner
         (A) Password Credentials
          |
          v
     +---------+                                  +---------------+
     |         |>--(B)---- Resource Owner ------->|               |
     |         |         Password Credentials     | Authorization |
     | Client  |                                  |     Server    |
     |         |<--(C)---- Access Token ---------<|               |
     |         |    (w/ Optional Refresh Token)   |               |
     +---------+                                  +---------------+
     
            Figure 5: Resource Owner Password Credentials Flow
```

OAuth 有一個直接支援帳號密碼的方式。授權的結果還是由授權伺服器的權杖和資源伺服器決定，透過中央授權來限制存取權杖可以做什麼，但是直接使用帳號密碼這個方式並不是很好。

- **使用時機**：在其他模式下都不太適用時，再考慮這個模式。

## Legacy: 隱含模式 ( Implicit Flow )

```flow
     +----------+
     | Resource |
     |  Owner   |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier     +---------------+
     |         -+----(A)-- & Redirection URI --->|               |
     |  User-   |                                | Authorization |
     |  Agent  -|----(B)-- User authenticates -->|     Server    |
     |          |                                |               |
     |          |<---(C)--- Redirection URI ----<|               |
     |          |          with Access Token     +---------------+
     |          |            in Fragment
     |          |                                +---------------+
     |          |----(D)--- Redirection URI ---->|   Web-Hosted  |
     |          |          without Fragment      |     Client    |
     |          |                                |    Resource   |
     |     (F)  |<---(E)------- Script ---------<|               |
     |          |                                +---------------+
     +-|--------+
       |    |
      (A)  (G) Access Token
       |    |
       ^    v
     +---------+
     |         |
     |  Client |
     |         |
     +---------+

   Note: The lines illustrating steps (A) and (B) are broken into two
   parts as they pass through the user-agent.

                       Figure 4: Implicit Grant Flow
```

在密碼模式 ( Password Grant ) 下適用於**純前端環境**；而在隱含模式 ( Implicit Flow ) 下就是前後端分離，而前端的部分 ( User-agent ) 即可視為一個完整的應用。

### OAuth Tools 實作

可以使用 [OAuth.tools](https://oauth.tools/) 這個線上服務來學習，點選 Demo: Implicit Flow 即可。

可以看到 Start Flow 的 Start URL 為：

```text
https://login-demo.curity.io/oauth/v2/oauth-authorize?  
&client_id=demo-web-client  
&response_type=token  
&redirect_uri=https://oauth.tools/callback/implicit  
&state=1599045172021-mww  
&scope=read%20phone%20email 
```

`client_id`、`response_type`、`redirect_uri` 是必須的。

按下 Run 後可能會要求登入 ( 隨便打即可 )，登入後就可以看到 Server Response ( 這個 Server Response 就是回傳的 URL 加上相關的資訊 )：

```text
https://oauth.tools/callback/implicit
#access_token=_0XBPWQQ_76c0d4ce-5424-4757-a0d1-0be0bc936d66
&scope=read+phone+email
&iss=https%3A%2F%2Flogin-demo.curity.io%2Foauth%2Fv2%2Foauth-anonymous
&state=1599045172021-mww
&token_type=bearer
&expires_in=299
```

`token_type=bearer` 表示權杖為 `bearer` 類型。

在隱含模式的設計下，**資源擁有者使用的瀏覽器**會與**授權伺服器**保持會話階段。簡單來說，就是使用者在登入中的情況下，可以隨時獲得新的存取權杖。

### 為何稱為隱含模式？

可以看到 `redirect_uri` 的 URL 指向的是客戶端。當瀏覽器接收到這個訊息後，會發送一個請求給用戶端，但不同的是，真正的用戶端是瀏覽器本身的 Web App，而不是提供 Web 服務的伺服器裡。也因此存取權杖不會發送到伺服器，而是保留在瀏覽器裡面，在瀏覽器的網路工具也幾乎是隱藏的。

此模式雖然簡單，但將存取權杖暴露在使用者面前並不是非常好的做法，最好要快速清除訊息或是轉到其他頁面。

### 不應該使用隱含模式

通常在處理完存取權杖後，應該要跳轉畫面或至少把資訊清除，因為權杖在網址上太容易取得了。而通常需要注意的是 [**XSS 跨站腳本攻擊**](https://www.wikiwand.com/zh-tw/%E8%B7%A8%E7%B6%B2%E7%AB%99%E6%8C%87%E4%BB%A4%E7%A2%BC)。

就算不是隱含模式，可能也有跳轉頁面的需要，在跳轉後，網址列上面的 Token 自然會消失。但是，為了要讓跳轉後的頁面也能夠知道存取權杖，在頁面跳轉時，可能會將存取權杖儲存在  `localStorage` 或 `sessionStorage` 裡。

眾所皆知，將機敏資料儲存在 `localStorage` 或 `sessionStorage` 不是安全的作法，因為 XSS 攻擊可以很簡單的獲得裡面的資料。或許可以使用特別的加密方式，來降低存取權杖被猜到的風險。

## 授權碼 ( Authorization Code )

```flow
     +----------+
     | Resource |
     |   Owner  |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier      +---------------+
     |         -+----(A)-- & Redirection URI ---->|               |
     |  User-   |                                 | Authorization |
     |  Agent  -+----(B)-- User authenticates --->|     Server    |
     |          |                                 |               |
     |         -+----(C)-- Authorization Code ---<|               |
     +-|----|---+                                 +---------------+
       |    |                                         ^      v
      (A)  (C)                                        |      |
       |    |                                         |      |
       ^    v                                         |      |
     +---------+                                      |      |
     |         |>---(D)-- Authorization Code ---------'      |
     |  Client |          & Redirection URI                  |
     |         |                                             |
     |         |<---(E)----- Access Token -------------------'
     +---------+       (w/ Optional Refresh Token)

   Note: The lines illustrating steps (A), (B), and (C) are broken into
   two parts as they pass through the user-agent.

                     Figure 3: Authorization Code Flow
```

Authorization Code 是第一個在 [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) 被提到的流程，所以有時候會被稱為 **標準流程**。

它分成**前端通訊 ( Frontchannel )** 和 **後端通訊 ( Backchannel )**  兩個部分。與前兩個相比，在隱含模式下，後端通訊與前端通訊是併在一起的；密碼模式下，根本不存在前端通訊，所以資源擁有者需要高度信任客戶端。

> 再複習一下前面的四種角色：
>
> - **資源擁有者 ( resource owner )**：就是使用者
> - **資源伺服器 ( resource server )**：存放資料的伺服器
> - **客戶端 ( client )**
>	- **前端客戶端**：瀏覽器或 User-Agent
>	- **後端客戶端**
> - **授權伺服器 ( authorization server )**：

除了資源伺服器以外，其他都會參與授權流程：

- **前端通訊**：指的是**前端客戶端**與**授權伺服器 ( Authorization Server )** 交換訊息的過程。
- **後端通訊**：指的是**後端客戶端 ( Client )** 與 **授權伺服器 ( Authorization Server )** 交換訊息的過程。

![OAuth-authorization-code-channel.png](/images/OAuth/OAuth-authorization-code-channel.png)

### 透過 OAuth.Tools 完成前端通訊

在 [OAuth.Tools](https://oauth.tools/) 頁面點選 **Demo: Code Flow**。

按下 Run 一樣會進入到登入畫面。此時的 Start Flow：

```text
https://login-demo.curity.io/oauth/v2/oauth-authorize?  
&client_id=demo-web-client  
&response_type=code  
&redirect_uri=https://oauth.tools/callback/code  
&state=1599045135410-jFe  
&scope=openid%20profile%20read  
&ui_locales=en  
```

登入結束後，一樣會根據 `redirect_uri` 回到指定的 URL。這時候會發送一個 Request 給 Web 伺服器，而這個伺服器就是客戶端。

![OAuth-authorization-code-redeem-code.png](/images/OAuth/OAuth-authorization-code-redeem-code.png)

接著就是最重要的就是 query 裡面的 `code`，這個 `code` 會給客戶端，並讓客戶端拿著 `code` 去向授權伺服器**兌換 ( redeem )** 存取權杖。

![OAuth-authorization-code-do-redeem.png](/images/OAuth/OAuth-authorization-code-do-redeem.png)

這邊可以選擇按下 Redeem Code 按鈕來獲得存取權杖，又或者使用它所提供的 cURL 來獲得，只能選擇一種方法使用，因為一個 `code` 只能兌換一次。

### 流程

最後來整理一下流程：

1. 在資源擁有者登入後驗證身分
2. 資源擁有者代理 ( User-Agent ) 向系統申請一個 `code` ( 有時效性、只能使用一次 )，並且告訴授權伺服器，有人會在限定的時間內使用這個特殊密碼來獲得特定房間的鑰匙
3. 資源擁有者代理 ( User-Agent ) 把這個特殊密碼給想要授權的對象 ( 客戶端 ) 
4. 客戶端使用 `code`，向授權伺服器兌換存取權杖

這個模式與[**特殊密碼**](/posts/oauth-first-look/#特殊密碼)很像。

## 刷新令牌 ( Refresh Token )

```flow
  +--------+                                           +---------------+
  |        |--(A)------- Authorization Grant --------->|               |
  |        |                                           |               |
  |        |<-(B)----------- Access Token -------------|               |
  |        |               & Refresh Token             |               |
  |        |                                           |               |
  |        |                            +----------+   |               |
  |        |--(C)---- Access Token ---->|          |   |               |
  |        |                            |          |   |               |
  |        |<-(D)- Protected Resource --| Resource |   | Authorization |
  | Client |                            |  Server  |   |     Server    |
  |        |--(E)---- Access Token ---->|          |   |               |
  |        |                            |          |   |               |
  |        |<-(F)- Invalid Token Error -|          |   |               |
  |        |                            +----------+   |               |
  |        |                                           |               |
  |        |--(G)----------- Refresh Token ----------->|               |
  |        |                                           |               |
  |        |<-(H)----------- Access Token -------------|               |
  +--------+           & Optional Refresh Token        +---------------+

               Figure 2: Refreshing an Expired Access Token
```

使用 `refresh_token` 來取得 `access_token` 這是最簡單的一個模式了。也因為先決條件是必須要有 **Refresh Token**，所以無法單獨存在。

上面的流程圖最重要的是步驟 `(G)` 和 `(H)`，因為如何取得 **Refresh Token** 的方式有很多種，不論是密碼模式或是 Authorization Code 模式下都有可能返回 `refresh_token` 了。

延續上面**授權碼 ( Authorization Code )** 所獲得的 Token：

![OAuth-refresh-token.png](/images/OAuth/OAuth-refresh-token.png)

之後點選 [OAuth.Tools](https://oauth.tools/) 裡的 Demo: Refresh Tokens 來使用剛剛 Authorization Code 的 **Refresh Token**：

![OAuth-refresh-token-demo.png](/images/OAuth/OAuth-refresh-token-demo.png)

我們可以選擇要使用哪個 Demo 的 `refresh_token`，之後按下 **Refresh Token** 就可以看到新產生的存取權杖。

### refresh_token 的作用

Q：既然都透過密碼模式與 Authorization Code 模式取得存取權杖了，那為何還需要 `refresh_token` 呢？

A：這是因為存取權杖是會**過期**的，如果每次過期都需要資源擁有者再登入一次，那是不是超麻煩的，所以才有這個 `refresh_token`，這同樣也像**特殊密碼**。但是與 Authorization Code 模式下獲得的那個 `code` 的特殊密碼不同的是，在限定的時間內，可以透過 `refresh_token` 獲取多次的存取權杖 (`code` 只能使用一次)。而且也與 `access_token` 不同，`refresh_token` 使用的地方是在**客戶端**與**授權伺服器**；`access_token` 使用的地方是在**客戶端**與**資源伺服器**。

與 `access_token` 相比，`refresh_token` 使用的頻率沒有那麼高，相對也就不容易被竊取，存活的時間也比較久，藉由固定一段時間更新 `access_token` 的方式，也能降低一些安全問題。

最後，`refresh_token` 可以使用幾次、多長時間、是否會返回 `refresh_token`，全部都是由**授權伺服器**決定的，也就是依照所需要的應用環境、存在的安全風險、使用不同的策略。你可能頻繁的更換 `access_token` 和 `refresh_token`，也可能一用就是一年。

## 客戶憑證 ( Client Credentials )

```flow
     +---------+                                  +---------------+
     |         |                                  |               |
     |         |>--(A)- Client Authentication --->| Authorization |
     | Client  |                                  |     Server    |
     |         |<--(B)---- Access Token ---------<|               |
     |         |                                  |               |
     +---------+                                  +---------------+

                     Figure 6: Client Credentials Flow
```

這個模式很特別，它可能會與其他模式並用以外，最特別的是，如果只是單純使用它，是完全不需要資源擁有者參與的。

在 [OAuth.Tools](https://oauth.tools/) 頁面點選 **Demo: Client Credentials Flow**。

直接按下 Run 後可以看到回傳的結果：

![OAuth-client-credentials.png](/images/OAuth/OAuth-client-credentials.png)

可以發現它與隱含模式一樣都沒有 `refresh_token`，這是因為 `client_secret` 只有客戶端擁有，不會送到瀏覽器去參與前端通訊。也因為不存在前端通訊，自然就不會知道資源擁有者是誰了。

也因為 `client_secret` 只有客戶端擁有，所以客戶端可以隨時取得 `access_token`。這就像客戶端建立了一個帳號系統，`client_id` 就是帳號；`client_secret`就是密碼。

## PKCE ( Proof Key for Code Exchange )

```flow
                                                 +-------------------+
                                                 |   Authz Server    |
       +--------+                                | +---------------+ |
       |        |--(A)- Authorization Request ---->|               | |
       |        |       + t(code_verifier), t_m  | | Authorization | |
       |        |                                | |    Endpoint   | |
       |        |<-(B)---- Authorization Code -----|               | |
       |        |                                | +---------------+ |
       | Client |                                |                   |
       |        |                                | +---------------+ |
       |        |--(C)-- Access Token Request ---->|               | |
       |        |          + code_verifier       | |    Token      | |
       |        |                                | |   Endpoint    | |
       |        |<-(D)------ Access Token ---------|               | |
       +--------+                                | +---------------+ |
                                                 +-------------------+

                     Figure 2: Abstract Protocol Flow
```

PKCE 是 Authorization Code 的安全強化版。

在整個過程添加了兩個動作：產生 `code_verifier` 和 `code_challenge`，並且在最後透過 `code_challenge` 驗證 `code_verifier`。最大的目的就是建立前端通訊與後端通訊的關聯。

### 原始的風險

Authorization Code 的流程是：

1. 在資源擁有者登入後驗證身分
2. 資源擁有者代理 ( User-Agent ) 向系統申請一個 `code`
3. 資源擁有者代理 ( User-Agent ) 將 `code` 轉給客戶端
4. 客戶端使用 `code`，向授權伺服器兌換存取權杖

可以看到上面的流程，`code` 可能透過網路傳遞了很多次。傳遞越多次就代表洩漏的風險就越高，攻擊者就有可能在這中間取得存取權杖。

以下是惡意應用竊取 `code` 的方式：

```flow
    +~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
    | End Device (e.g., Smartphone)  |
    |                                |
    | +-------------+   +----------+ | (6) Access Token  +----------+
    | |Legitimate   |   | Malicious|<--------------------|          |
    | |OAuth 2.0 App|   | App      |-------------------->|          |
    | +-------------+   +----------+ | (5) Authorization |          |
    |        |    ^          ^       |        Grant      |          |
    |        |     \         |       |                   |          |
    |        |      \   (4)  |       |                   |          |
    |    (1) |       \  Authz|       |                   |          |
    |   Authz|        \ Code |       |                   |  Authz   |
    | Request|         \     |       |                   |  Server  |
    |        |          \    |       |                   |          |
    |        |           \   |       |                   |          |
    |        v            \  |       |                   |          |
    | +----------------------------+ |                   |          |
    | |                            | | (3) Authz Code    |          |
    | |     Operating System/      |<--------------------|          |
    | |         Browser            |-------------------->|          |
    | |                            | | (2) Authz Request |          |
    | +----------------------------+ |                   +----------+
```

在 `(4)` 這個步驟，惡意應用 ( Malicious App ) 就可能截取到 `code`。就算不是直接截取，也可能通過這種方式來猜測到 `code`。

所以，為了降低被攻擊的機會，可以添加一些加密方式，來提升攻擊的難度。配合使用 **Client Credentials Flow** 或許是一個方法，因為依照設計 `client_secret` 只有客戶端擁有，並且只在客戶端與授權伺服器間流通。但這樣做的前提為**客戶端是已經被認可的客戶端**，這還是沒辦法證明**客戶端、資源擁有者、使用 code 兌換存取權杖** 都是同一個人。

### 解決方法

那要如何讓前端通訊與後端通訊建立更明確的關係？客戶端能用什麼方式來證明是自己是授權的那一個？

以下是流程：

1. 客戶端告訴瀏覽器證明自己的方式，並讓瀏覽器將這個訊息告訴授權伺服器。
2. 瀏覽器取得特殊密碼 `code`，並與授權伺服器約定好，在未來，會有一個人帶著這個 `code` 與一個能證明自己就是**這個人**的方法來找你。
3. 瀏覽器將 `code` 交給客戶端。
4. 客戶端帶著 `code` 與證明自己的方式訪問授權伺服器。
5. 授權伺服器會檢查 `code` 與證明的方式，如果符合，就會將存取權杖交給客戶端。

那要如何證明自己的身分呢？那就是有一個很困難的問題，這個問題的答案只有自己知道。而且這個題目的答案很難推敲出來，從題目證明答案是否正確很容易。

這個方式就是**單向雜湊函數( One-way Hash Function )**。從一個方向運算很簡單，但反過來很難。在 [RFC 7636 － 4.6](https://www.rfc-editor.org/rfc/rfc7636#section-4.6) 就設定了 `code_challenge` 是由 SHA-256 來組成這個難題，只要 `code_verifier` 正確，就能夠證明自己的身分了。

### 使用 OAuth Tools 實作

在 [OAuth.Tools](https://oauth.tools/) 頁面點選 **Demo: Code Flow**，並勾選 Use PKCE 的選項。

![OAuth-use-PKCE.png](/images/OAuth/OAuth-use-PKCE.png)

接著，可以看到 Start Flow 多出 `code_challenge` 與 `code_challenge_method`：

```text
https://login-demo.curity.io/oauth/v2/oauth-authorize?  
&client_id=demo-web-client  
&response_type=code  
&redirect_uri=https://oauth.tools/callback/code  
&state=1599045135410-jFe  
&scope=openid%20profile%20read  
---
&code_challenge=mSrwhFdk8l_oSJvSdiyLtVe2SLf5o0hvH4h4xOnLpAU  
&code_challenge_method=S256  
---
&prompt=login  
&ui_locales=en  
&nonce=1599046102647-dv4
```

並且在後端通訊 ( 也就是兌換存取權杖時 ) 將答案告訴授權伺服器：

```text
curl -Ss -X POST \
https://login-demo.curity.io/oauth/v2/oauth-token \
-H 'Authorization: Basic ZGVtby13ZWItY2xpZW50OjZrb3luOUtwUnVvZll0MlU=' \
-H 'Content-Type: application/x-www-form-urlencoded' \
-d 'grant_type=authorization_code&redirect_uri=https%3A%2F%2Foauth.tools%2Fcallback%2Fcode&code=rtNEXkJGpsiaiPd198FK8teHJsjOAYWm& code_verifier=1Ex9PnVFClsom1mlAKVyRSTKmXpKi26a8qXfK8KSBdNJgh3hNxdrwtDq4uj01Rt3'
```

如果驗證失敗，不會通過授權並且 `code` 會被認會可能已洩漏，所以就不能再使用。如果通過，就能獲得存取權杖。

## 設備碼 ( Device Code )

```flow
      +----------+                                +----------------+
      |          |>---(A)-- Client Identifier --->|                |
      |          |                                |                |
      |          |<---(B)-- Device Code,      ---<|                |
      |          |          User Code,            |                |
      |  Device  |          & Verification URI    |                |
      |  Client  |                                |                |
      |          |  [polling]                     |                |
      |          |>---(E)-- Device Code       --->|                |
      |          |          & Client Identifier   |                |
      |          |                                |  Authorization |
      |          |<---(F)-- Access Token      ---<|     Server     |
      +----------+   (& Optional Refresh Token)   |                |
            v                                     |                |
            :                                     |                |
           (C) User Code & Verification URI       |                |
            :                                     |                |
            v                                     |                |
      +----------+                                |                |
      | End User |                                |                |
      |    at    |<---(D)-- End user reviews  --->|                |
      |  Browser |          authorization request |                |
      +----------+                                +----------------+

                    Figure 1: Device Authorization Flow
```

Device Code 的流程與前面幾個都不相同。以往都是從登入開始，然後跳轉頁面回到 App ( 客戶端 )。也就是先有前端通訊，再有後端通訊。

但 Device Code 不是由資源擁有者發起，而是客戶端發起。大致流程為：

1. 客戶端發起，向授權伺服器取得 `device_code` 和 `user_code`。
2. 客戶端將 `user_code` 交給資源擁有者，自己則保留 `device_code`。
3. 資源擁有者透過 Endpoint 與 `user_code` 進行授權。
4. 客戶端使用 `device_code` 訪問授權伺服器是否有人授權給它。

`user_code` 像是之前的特殊密碼；`device_code` 更像是 `session`。

可以到 [OAuth 2.0 Playground](https://www.oauth.com/playground/index.html) 嘗試。

首先，獲得來自授權伺服器的所有資訊：

```http
{
  "device_code": "NGU5OWFiNjQ5YmQwNGY3YTdmZTEyNzQ3YzQ1YSA",
  "user_code": "BDWD-HQPK",
  "verification_uri": "https://example.okta.com/device",
  "interval": 5,
  "expires_in": 1800
}
```

當第一次點 **poll** 時，會看到：

```http
HTTP/1.1 400 Bad Request

{
  "error": "authorization_pending"
}
```

這表示資源擁有者還沒有完成登入授權的請求，所以授權伺服器會回傳等待授權。當資源擁有者完成請求後，客戶端就可以拿到存取權杖了：

```http
HTTP/1.1 200 OK

{
  "token_type": "Bearer",
  "access_token": "RsT5OjbzRn430zqMLgV3Ia",
  "expires_in": 3600,
  "refresh_token": "b7a3fac6b10e13bb3a276c2aab35e97298a060e0ede5b43ed1f720a8"
}
```

### 使用例子

使用 Device Code 的方式隨處可見：

- 使用 QR Code 登入的應用 ( 特殊連接登入的應用 )
   ![OAuth-device-code-qr-code.png](/images/OAuth/OAuth-device-code-qr-code.png)
	QR Code 同樣是一個特殊連結。

## 結語

對我來說，要弄清楚其中運作的流程是需要一點想像力的，也多虧有 [用Keycloak學習身份驗證與授權](https://ithelp.ithome.com.tw/users/20112470/ironman/4324) 這一系列的文章幫助我理解。

## Reference

- [用Keycloak學習身份驗證與授權](https://ithelp.ithome.com.tw/users/20112470/ironman/4324) 👍
- [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749)
- [RFC 7637](https://www.rfc-editor.org/rfc/rfc7636)
- [RFC 8628](https://www.rfc-editor.org/rfc/rfc8628)
- [OAuth Tools](https://oauth.tools/)
- [OAuth2 Playground](https://www.oauth.com/playground/index.html)