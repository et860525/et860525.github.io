---
title: "理解 OAuth"
date: 2023-03-24
draft: false
author: "Chen Yu Fan"
tags: ["TypeScript", "Projects", "Prisma", "Express"]
---

![OAuth-logo.png](/images/OAuth/OAuth-logo.png)

OAuth 是一個開放標準的**授權協議**，它允許使用者讓**第三方應用**存取該使用者在某網站儲存的私密資源。

試想有一棟房子，裡面有很多個房間，裡面有一位房東 ( 使用者 ) 擁有一把萬能鑰匙，可以開啟所有的房間門。除此之外，這把萬能鑰匙還有一個作用，就是可以產出**特定門**的鑰匙，而產生出來的鑰匙可以交給其他人，這樣其他人就可以進出**特定的房間**，這個動作就是「授權」。

OAuth 2.0 是 OAuth 的進化版，它是**授權框架 ( authorization framework )**，允許應用向使用者請求授權，然後取得 Token，並且用它來訪問資源。

> The OAuth 2.0 authorization framework enables a third-party
   application to obtain limited access to an HTTP service, either on
   behalf of a resource owner by orchestrating an approval interaction
   between the resource owner and the HTTP service, or by allowing the
   third-party application to obtain access on its own behalf. － [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749)
>
> OAuth 2.0 授權框架能讓第三方應用以有限的方式訪問 HTTP 服務，建構資源擁有者與 HTTP 服務之間的許可交互機制，或是通過第三方應用代表自己訪問服務。

作為一個授權框架，首先要先了解各個在 OAuth 裡定義的四種角色 ( [RFC 6794 - Roles](https://www.rfc-editor.org/rfc/rfc6749#section-1.1) )：

1. **資源擁有者 ( resource owner )**：能將訪問授權委託給出去，資源擁有者指的是使用者本身，又或是系統中的帳號。
2. **資源伺服器 ( resource server )**：託管受保護資源的伺服器，接受訪問 Token 並回應受保護的資源。
3. **客戶端 ( client )**：指的就是代表資源擁有者訪問資源伺服器的應用。
4. **授權伺服器 ( authorization server )**：驗證資源所有者的身分並發放訪問 Token。

## OAuth 不是什麼

- **OAuth 不是身分驗證協議**：OAuth 重點是**授權**而不是進行身分驗證。
- **OAuth 沒有定義用戶對用戶的授權機制**
- **OAuth 沒有定義授權處理機制**：OAuth 定義一些授權框架或流程，但不定義授權的內容。
- **OAuth 沒有定義權杖格式**：OAuth 協議明確申明了權杖的內容，對於客戶端是完全不透明的。但接受權杖處理的服務，就必須要理解權杖，這表示授權伺服器僅產生權杖，理解權杖的這件事情都依賴於資源伺服器。使用上面的例子：萬能鑰匙產生新的鑰匙，但這把鑰匙能不能使用全部都由門鎖決定，與獲得鑰匙的人、房東都沒有關係。
- **OAuth 2.0 不定義加密方法**：許多服務都會要求客戶端支援 HTTPS，但框架本身是不關心這個部分，也不管權杖如何簽名、加密、驗證。
- **OAuth 2.0 不是單體協議**：分成多個定義和流程，每個定義和流程都有各自的場景。

## OAuth 1.0

OAuth 2.0 並不相容於 OAuth 1.0，現在看到的 **OAuth 大多都是直接指 OAuth 2.0**。2009年4月23日，OAuth 宣告了一個1.0協定的安全漏洞。

# OAuth 2.0 概念

## 假裝自己是用戶 使用帳號密碼

有一支程式能幫你收發信件，只需要告訴它你信箱的帳號密碼，這表示你完全信任它。也不用自己寫程式，Gmail 就有這一個服務了，假如你想要讓 Gmail 收發來自 Yahoo 的信件，只要告訴它你的 Yahoo 帳號密碼就行，這也表示你信任 Google 的服務。

## 特殊密碼

如果這種**代理( proxy )** 的方式不只是在一般的軟體，而是作業系統或是硬體裡。

如果需要在公司電腦或公共電腦登入，或許還是使用 Google 的服務，但實際上中間還通過好幾層的代理。首先，在硬體、作業系統上可能會有安裝鍵盤紀錄器之類的東西，瀏覽器也是一個代理，但使用的是可信任的瀏覽器嗎？

這時候就可以與服務約定一種「**特殊密碼**」，只能使用一次的密碼 ( OTP, One Time Password )。目前更常見的是**雙重認證 ( 2FA, Two Factor Authentication )**，實際上的形式為：

- **簡訊或信箱驗證**：一組限時且只能使用一次的密碼
- **特殊連接驗證**：一個特殊的連接，該連接有時效性，且只能存取一次
- **透過時間產生特殊密碼**：TOTP, Time-base One Time Password
- **透過雜湊產生特殊密碼**：HOTP, HMAC-based One Time Password
- **與系統服務約定好數組一次性密碼**：例如救援碼

## 開發者權杖

在開發的應用下，你本身就是開發者之一，但是在內部可能會有控制的問題，所以還需要一個看門人 ( gatekeeper ) 來檢查是否有權限操作。

## 委託授權

以上都是**整份授權**，都還沒有把權限拆分更小的部分並分開授權，例如只授權存取角色、Email 地址、使用者資訊，並且產生一組特殊密碼。

## 結語

過去，在使用 [Passport.js](https://www.passportjs.org/) 寫專案時，其實都沒有真正的了解它實際的概念。看到現在，只要能釐清各個角色在之中做的事情，對於理解上就不會太過複雜了。

## Reference

- [用Keycloak學習身份驗證與授權](https://ithelp.ithome.com.tw/users/20112470/ironman/4324)