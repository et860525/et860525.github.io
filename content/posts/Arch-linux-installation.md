---
title: "Arch Linux 終端機 - 安裝(一)"
date: 2023-05-16
draft: false
author: "Chen Yu Fan"
tags: ["Linux"]
---

在 Windows 上做開發時，常常會遇到有些套件只能在 Linux 上使用。如今 Windows 推出了 [WSL](https://learn.microsoft.com/zh-tw/windows/wsl/install) 能讓使用者同時使用 Windows 與 Linux。但是如果提到要學習或想完整體驗 Linux，還是建議安裝虛擬機並實際操作。

所以此篇文章會使用 [VirtualBox](https://www.virtualbox.org/) 來建立一個未含 GUI 的純 [Arch Linux](https://archlinux.org/) 終端機並且安裝相關的開發套件。

> 基本上只要是使用 Linux OS，不管哪個 Linux distribution，安裝套件的方式都大同小異

## Arch Linux 安裝

[Arch Linux](https://www.archlinux.org/) 有著輕量、彈性與簡潔，並嚴格遵守 Arch 設計哲學。無論是擴充、打造任何類型的系統都變得更容易。

首先，先到 [Arch Linux Downloads](https://archlinux.org/download/) 下載映像檔，完成後，再到 VirtualBox 裡建立一個新的虛擬機，並放入剛才下載的 ISO 檔案。

相關設定 (根據自己需求調整)：
- 處理器：2 CPUs
- 記憶體：4096 MB
- 硬體：128 GB

按下啟動後就能看到以下畫面

![Arch-init.png](/images/Arch-linux/Arch-init.png)

這裡我使用 [archinstall](https://wiki.archlinux.org/title/archinstall) 來安裝 Arch。在上面輸入 `archlinux` 讓它跑完後會出現

![Arch-archinstall.png](/images/Arch-linux/Arch-archinstall.png)

參考 [Archinstall Guided Installation](https://python-archinstall.readthedocs.io/en/latest/installing/guided.html) 來進行設定，以下是我的設定 (沒列出來的就是維持預設)：

- Mirror region：`Taiwan`
- Drive(s)：剛剛新增的那顆硬碟
- Disk layout：讓它自動選擇建議的分區
	- 選擇 `Wipe all selected drives and use a best-effort default partition layout`
	- 選擇 `btrfs`
	- 選擇 `yes`
	- 選擇 `yes`
- Hostname：可自行設定名稱
- Root password：自行輸入密碼
- User account：建立一個使用者
	- 輸入完帳號密碼後，選擇是否為 superuser 的地方選擇 `yes`
- Profile：`minimal`
- Audio：`No audio server`
- Additional packages：`neovim`、`openssh`
- Network configuration：`Use NetworkManager`
- Timezone：`Asia/Taipei`

完成後按下 **Install** 就會開始安裝。完成後，會跳出選項並選擇 `no`，之後輸入 `sudo shutdown now` 來關機。

之後移除 ISO：

![Arch-remove-iso.png](/images/Arch-linux/Arch-remove-iso.png)

再開機後，就能進入安裝完成的 Arch Linux 了。

## 結語

如果不想使用 `archinstall` 安裝方式，可以參考官方的 [Installation guide](https://wiki.archlinux.org/title/Installation_guide) 來安裝。

下一篇是關於 Shell 與套件的安裝與設定。

## Reference

- [Should You Run Linux in a Virtual Machine or WSL?](https://www.makeuseof.com/linux-virtual-machine-or-wsl/)
- [Archinstall](https://archinstall.readthedocs.io/installing/guided.html)