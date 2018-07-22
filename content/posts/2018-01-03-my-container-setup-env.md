+++
title = "[Build container] Docker 簡介與 Go 環境安裝"
tags = ["container", "docker", "Go", "Linux"]
categories = ["container"]
date = 2018-01-03T14:51:05+08:00
+++

# Docker 簡介
就軟體的 Docker 來說，Docker 是一個開放原始碼軟體專案。可以讓應用程式佈署在軟體容器下並執行。在 Linix 中， Docker 利用 Linux Kernel 中的資源分離機制，如cgroups, Namespace ，來建立獨立的軟體容器（containers）。
它最早釋出於2013/03/13，原作者為 Solomon Hykes 。
他於 2013 PyCon 的 5 分鐘 lightning talk 中提出 [`The Future of Linux Containers`](https://www.youtube.com/watch?v=9xciauwbsuo)。
並在 lightning talk 中簡單展示 Docker 的使用方法。

> 小小疑惑: 為什麼 go 語言的專案會跑到 PyCon 中發表呢？我覺得比較有關的會是 open source 或是 Linux 相關的 Conference 呢！？

#### Docker 擁有的特點有：
- 輕量化：共享同一台系統的 Kernel 資源，所以可以迅速啟動而且用的 memory 較少。而且 image 是 Filesystem 建制的，所以可以共享相同文件。
- open source 標準：可以運行於 Linux distributions 和 Windows OS.
- 安全：透過 cgroups, Namespace 等來隔離應用程式。

#### 產生的好處：
- 加速開發
- 可以擺脫 OS 環境的限制
- 消除環境不一致

#### Containers V.S. Virtual Machines
- Container
Container 是一個抽象的應用層，將 code 與 dependencies 打包在一起。多個 Container 可以在同一台機器上運行，並與其他容器共享操作系統 Kernel ，且每個 Container 在 user space 中皆為獨立運作的 processes。
![](https://www.docker.com/sites/default/files/Container%402x.png)

- Virtual Machines
一種轉體程式或作業系統，能夠像獨立的計算機一樣執行。一台主機上可以同時存在多個虛擬機。
![](https://www.docker.com/sites/default/files/VM%402x.png)

> 本系列文注重如何寫自己的 Container ，不是記錄自己的 Docker 安裝與使用筆記，所以這邊不多述使用 Docker 相關的紀錄。

# Go 環境安裝
Docker 專案使用 Go 語言開發出來的。透過 `自己動手寫 Docker`一書，將以 Go 來實現。

> 我希望我也可以挑戰使用其他語言來實作，所以後面的文章，會希望自己不是只紀錄 Go 的寫法。

### 安裝環境
- Ubuntu 16.04
- Kernel version: 4.13.0-1002-gcp
- Go version: v1.9.2 linux/amd64

### 安裝
依據 [https://github.com/golang/go/wiki/Ubuntu](https://github.com/golang/go/wiki/Ubuntu) 提供的文件來安裝。

```
$ sudo add-apt-repository ppa:longsleep/golang-backports
$ sudo apt-get update
$ sudo apt-get install golang-go
```
來查看 go 環境變數  

```
$ go env
GOARCH="amd64"
GOBIN=""
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/home/sufuf3/go"
GORACE=""
GOROOT="/usr/lib/go-1.9"
GOTOOLDIR="/usr/lib/go-1.9/pkg/tool/linux_amd64"
GCCGO="gccgo"
CC="gcc"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build791999341=/tmp/go-build -gno-record-gcc-switches"
CXX="g++"
CGO_ENABLED="1"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
```

# 小結
本篇主要簡單介紹 Docker 是什麼，以及配置好 Container 的開發環境。

Ref: 
- [https://www.docker.com/company](https://www.docker.com/company)
- [https://www.docker.com/what-container](https://www.docker.com/what-container)
- [https://github.com/golang/go/wiki/Ubuntu](https://github.com/golang/go/wiki/Ubuntu)
- [`自己動手寫 Docker`](http://m.sanmin.com.tw/product/index/006384800)

### 相關系列文：
[[Build container] 自己的 Container 自己寫](/series/my-own-container)
