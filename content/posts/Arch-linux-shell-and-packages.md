---
title: "Arch Linux 終端機 - Shell與套件(二)"
date: 2023-05-17
draft: false
author: "Chen Yu Fan"
tags: ["Arch", "Linux"]
---

在開始安裝套件前，首先認識一下 [pacman](https://wiki.archlinux.org/title/pacman)，它是 Arch Linux 的 package manager。幾個使用方式如下：

- `pacman -Suy` 更新所有已安裝的套件
- `pacman -S package_name1 package_name2` 下載一(多)個套件
- `pacman -Q` 已下載的套件
- `pacman -R package_name1 package_name2` 移除一(多)個套件

更多的資訊可以輸入 `pacman -h`。

---

## 預先安裝

- Windows 的使用者請先安裝 [Windows Terminal](https://apps.microsoft.com/store/detail/windows-terminal/9N0DX20HK701?hl=zh-tw&gl=tw&icid=CatNavSoftwareWindowsApps)，往後都會使用 SSH 的方式來進行操作
- 安裝 [Nerd Fonts](https://www.nerdfonts.com/) 以解決遺失 Icons 的問題

## SSH 連線

開始設定 SSH：

```bash
sudo pacman -S openssh # 上一篇有安裝了
sudo systemctl enable sshd.service # 開機自動啟動服務
sudo systemctl start sshd.service # 啟動服務
```

> SSH 預設的 Port 為 `22`。

接著要到 VirtualBox 進行 Port 的相關設定，將 VM 的 `Port 22` 映射到主機的 `Port 3022`：

![Arch-port-setting.png](/images/Arch-linux/Arch-port-setting.png)

接著打開 Windows Terminal 進行連線：

```powershell
ssh -p 3022 username@localhost
```

輸入密碼後就可以進入虛擬機。

## Fish Shell

先從 Shell 開始，這裡我會選擇使用 [Fish](https://fishshell.com/) 來當作我的 `Alternative shells`。

```bash
sudo pacman -Suy # 先確認現存的套件都是最新的
sudo pacman -S fish
exec fish # 進入 fish shell
```

為了讓每一次開啟都會自動進入到 fish shell，使用 `nvim ./.bashrc` 並加入：

```bash
exec fish
```

如果不想在每一次 `sudo` 時都輸入密碼，使用 `sudo nvim /etc/sudoers.d/` 並選擇正在使用的帳戶 ( 危險但是方便 )：

```text
username ALL=(ALL) NOPASSWD: ALL
```

### Fisher

安裝 Fish 的插件管理器 [jorgebucaran/fisher](https://github.com/jorgebucaran/fisher)：

```bash
curl -sL https://raw.githubusercontent.com/jorgebucaran/fisher/main/functions/fisher.fish | source && fisher install jorgebucaran/fisher
```

接著使用 Fisher 安裝 [IlanCosman/tide](https://github.com/IlanCosman/tide)。它能讓使用者自己設 Status line 和主題：

```bash
fisher install IlanCosman/tide@v5
```

安裝完後就能看到設定畫面，根據自己的喜好做設定即可。

此時會注意到某些 Icons 無法顯示，如果有安裝 [Nerd Fonts](https://www.nerdfonts.com/) 就到  Windows Terminal 套用就可以解決了：

![Arch-nerd-fonts.png](/images/Arch-linux/Arch-nerd-fonts.png)

想重新設定可以輸入：

```bash
tide configure
```

如果你是 Node.js 的開發者，你就會需要 [nvm](https://github.com/jorgebucaran/nvm.fish) 來管理 Node.js 的版本：

```bash
fisher install jorgebucaran/nvm.fish
```

安裝 LTS 版本：

```bash
nvm install lts
```

接著可以使用以下指令確認是否安裝成功：

```bash
npm -v
// 9.5.1

node -v
// v18.16.0
```

如果重開機沒有 Node.js 可以輸入 `nvm use lts` 來解決這個問題。

> 除了上面兩個插件外，我還安裝了：
> - [z](https://github.com/jethrokuan/z) 快速跳到常用的資料夾
> - [Sponge](https://github.com/meaningful-ooo/sponge) 不會去記錄失敗的指令到歷史指令清單

其他更多的 Fisher 插件可以到 [jorgebucaran/awsm.fish](https://github.com/jorgebucaran/awsm.fish) 查詢。

## 實用套件

### git

[git](https://git-scm.com/) 版控系統

```bash
sudo pacman -S git
```

相關設定 ( 直接在命令列輸入 )：

```bash
git config --global core.autocrlf input
git config --global core.safecrlf true
git config --global init.defaultBranch main
git config --global core.editor "nvim"
```

### exa

[exa](https://the.exa.website/)  `ls` 的現代版本。

```bash
sudo pacman -S exa
```

根據[官方設定](https://fishshell.com/docs/current/cmds/alias.html)，直接將設定加到 `.config/fish/config.fish`：

```fish
alias l "exa -lg --icons"
alias ll "exa -alg --icons"
```

退出後輸入 `exec fish` 來套用新的設定。

### tmux

[tmux](https://github.com/tmux/tmux) 在一個終端機下，同時開啟多個視窗 ( `windows` )、分割區塊 ( `panes` )。

```bash
sudo pacman -S tmux
```

使用方式可以參考 [終端機 session 管理神器 — tmux](https://larrylu.blog/tmux-33a24e595fbc)。

### neovim

[neovim](https://neovim.io/) 是一個以 vim 為基底的可擴展文字編輯器。如果在上一篇沒有安裝的話，這邊可以使用 pacman 來安裝：

```bash
sudo pacman -S neovim
```

如果沒有下載任何插件那它就和 vim 一樣，如果有需要請參考我的設定 [et860525/dotfiles](https://github.com/et860525/dotfiles)，未來我也會針對 neovim 來寫一篇文章。

## 結語

這就是目前我專案建構與運行的環境，最一開始卡關的地方是 `Port` 映射的部分，之後的挑戰就是建構屬於自己的 neovim。雖然我通常還是使用 VSCode 再用 SSH 連線到虛擬機，但是有時候立即更改檔案 neovim 對我的幫助就很大。

# Reference

- [Arch Linux - Command-line shell](https://wiki.archlinux.org/title/command-line_shell)
- [Fish docs](https://fishshell.com/docs/current/index.html)
- [Fisher](https://github.com/jorgebucaran/fisher)