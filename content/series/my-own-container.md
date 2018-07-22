+++
title = "[Build container] 自己的 Container 自己寫"
tags = ["container", "docker", "Go", "Linux"]
categories = ["container"]
date = 2018-01-03T14:51:05+08:00
+++

# 手工打造自己的  - mini-container
<img src="/images/my-own-container/Susie-Blake-1024x512.png"  />

### 前言
有在使用 Docker 的人一定對這 Container 的工具不陌生，但身為一個資工人的自己，我開始問自己，那這工具怎麼實作出來的？結果，我完全沒有任何頭緒。看著 docker 或 moby 的程式碼，我完全不知的怎麼下手。
有的只有承認自己是多麼的弱。再想想，我光是期末專題的純手工打造 Mini-filesystem 就已經花了好幾天在看 linux 的 ext2 原始碼。還研究了很久。雖然最後終於從如何切 block 到 inode 到 VFS 到 CLI 一連串的有點貫通後，和隊友一同開發出一個 ext2 的變形 FS 。為了能夠好好的掌握這技術背後是怎麼打造的，這系列，將是我研讀 [`自己動手寫 Docker`](http://m.sanmin.com.tw/product/index/006384800) 這本書，再加上自己實作與理解的部分，用自己的想法所寫下的紀錄。也希望透過這系列的紀錄，能夠讓自己將這技術儲存在自己的腦袋記憶中。

# 目錄

#### 1. [Docker 簡介與 Go 環境安裝](/posts/2018-01-03-my-container-setup-env/)
#### 2. [Container with Linux Namespace](/posts/2018-01-14-container-linux-namespace/)
#### 3. [(待) 關於 Linux Namespace]()
#### 4. [(待) 關於 Linux Cgroups]()
(待續)
