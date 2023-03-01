---
title: "Pnpm ( Performant Node Package Manager )"
date: 2023-02-26
draft: false
author: "Chen Yu Fan"
tags: ["Node.js", "tools"]
---

[Pnpm](https://pnpm.io/) ( Performant Node Package Manager ) 是一個套件管理器。根據官網表示，可以節省磁碟空間並提升安裝速度。

> Fast, disk space efficient package manager

<!--more-->

# pnpm 的特點

1. **不同的專案共用依賴套件，節省磁碟空間**
	所有依賴套件的檔案都會被存在同一個位置中，當有專案需要依賴套件時，它的檔案會被**硬連結**到該位置，這樣就不會消耗額外的磁碟空間
2. **非扁平化的 node_modules 目錄**
	npm 與 yarn 都是以扁平化的方式建立 node_modules 目錄，他們會產生**非法訪問依賴**的問題。舉個例子，今天有 A、B 和 C 三種套件，而他們之中的關係是 A 依賴 C 的 API，如果 B 版本更新並依賴 C 的 2.0.1 版本，但 A 依然還是使用舊版的 C，那就會報錯。
	
	而 pnpm 是使用**軟連結**的方式來建構 node_modules 目錄：
	
	![rust-same-string.png](/images/pnpm/pnpm-link.jpg)
	
	當開發者安裝套件 `bar@1.0.0`，而 `bar@1.0.0` 又依賴 `foo@1.0.0`套件，此時 pnpm 就會將專案直接依賴的套件**硬連結**於 `.pnpm store`。而 `bar@1.0.0` 的 `foo@1.0.0` 就會使用軟連結直接依賴的 `foo@1.0.0`。
	
	這樣就可以避免非法訪問依賴的問題了。

更多與 npm/yarn 的比較 [Feature Comparison](https://pnpm.io/feature-comparison)

# 安裝方式

我使用的是 Linux 系統，根據官網的文件下載：

```bash
# bash
wget -qO- https://get.pnpm.io/install.sh | ENV="$HOME/.bashrc" SHELL="$(which bash)" bash -
```

其他下載方式：[pnpm Installation](https://pnpm.io/zh-TW/installation)

# Commands

| npm command    | pnpm equivalent |
| -------------- | --------------- |
| npm init       | pnpm init       |
| npm install    | pnpm install    |
| npm i \<pkg>   | pnpm add \<pkg> |
| npm run \<cmd> | pnpm \<cmd>     |

- `pnpm update`：如果沒有加上 `--filter` 來指定 package，那就是更新全部
- `pnpm remove`：也可以使用 `rm`, `uninstall`, `un`

更多的指令可以參考官網的 [CLI commands](https://pnpm.io/cli/add)。

# 結論

最近開始轉成使用 pnpm，有感的就是速度很快。會使用它最大的原因就是它能將依賴件都放在同一資料夾管理 (專案裡的`.pnpm`)，這樣就不會因為不同的依賴件版本，造成其他依賴件錯誤的問題。

---

# Source

- [pnpm, Fast, disk space efficient package manager](https://pnpm.io/)
- [为什么现在我更推荐 pnpm 而不是 npm/yarn?](https://blog.csdn.net/qiwoo_weekly/article/details/114109197)