+++
title = "邊緣運算之容器管理工具 - K3s 之介紹與玩耍"
date = 2019-10-10T16:48:35+08:00
tags = ["Kubernetes", "K8s", "K3s", "cloud-native", "container"]
categories = ["cloud-native"]
+++

## 前言

Kubernetes 是雲原生技術 (cloud-native technologies)，也是雲原生計算 (Cloud Native Computing) 重要的技術之一，而且在使用 Kubernetes 時，應用的硬體資源通常也比較好。然而， 容器 (Container) 技術的崛起，也帶動了邊緣運算 (Edge computing) 導入 Container 技術的風潮。  
  
邊緣運算顧名思義就是將應用程式等服務運算，由網路中心節點，移往網路邏輯上的邊緣節點來處理。也就是從網路中心節點處理大型服務來分解，切割成更小、更容易管理的部份，分散到邊緣節點去處理。可以想像，如果網路中心節點的角色如同總經理，邊緣節點的角色如同部門經理，而終端裝置的角色如同基層員工。在這架構下，想當然耳就是個分散式架構，而邊緣節點因為更接近於終端裝置，因此可以加快資料的處理與傳送速度，減少延遲。如同基層員工直經回報部門經理，而重要的資訊，部門經理再回報給總經理一樣。也因此在邊緣運算的應用中一直都有重要的應用場景在，例如大家常聽到的 IoT。  
  
當邊緣運算節點硬體效能越來越強，能做的事情當然也就可以越來越多，即便是個嵌入式主機板，也是可以跑很多應用程式的 Container 的。然而有了越來越多的容器，當然也需要厲害的 Container 管理工具。  
  
K3s 就是在這需求下誕生的產品。  
  

## K3s 介紹

### K3s 簡介

K3s 是由 Rancher Labs 推出的 輕量化 Kubernetes 開源專案，也是 CNCF 官方認證的 Kubernetes 發布版本。而且是以產品設計出發的，讓管理者在設備資源有限的環境下，仍然可以良好的運作 Kubernetes，並管理 containers。因此，在這優勢下，K3s 可以很好的應用在 Edge, IoT, CI, ARM 等情境、環境下。  
它所需的系統資源並不多：  

- 在 Server 端： 只需要 512 MB
- 在 Node 端：只需要 75MB
- Disk 大小只需 200MB

而且可以用在 x86_64, ARMv7, ARM64 等平台架構。  

### K3s 架構

K3s 是將 Kubernetes 的元件 (包含 kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, kube-proxy) 打包在一起，並以 server 和 angent 的 model 形式呈現。  

因此 K3s 的架構可以看到如下， 如圖所示，可以到有 Server 與 Angent。  

- Server: (Kubernetes 管理元件 + SQLite + Tunnel Proxy)
    - API Server: 來提供 API 的，並且會將資料存到資料庫(預設使用 SQLite)。
    - SQLite: 是存放 Cluster status 資料的地方。同時也有支援 etcd, Mysql, Postgres (v0.9.1 ref: [server-options K3S_STORAGE_ENDPOINT](https://rancher.com/docs/k3s/latest/en/installation/#server-options))
    - Scheduler: 和 Kubernetes 一樣有選 Node run pod 的功能。
    - Controller Manager: 算是 K3s 的後端程式，是處理 Kubernetes resource 的 controller manager。
- Angent: (Kubernetes data plan 元件 + Tunnel Proxy)
    - Kube-proxy: 就是做 forwrading 用的。
    - Flannel: Pod 溝通用的網路。
    - Kubelet: 啟動 container 用。
- Tunnel Proxy of Server & Angent: 
    - 在 Server 的 [`pkg/daemons/control/tunnel.go`](https://github.com/rancher/k3s/blob/v0.9.1/pkg/daemons/control/tunnel.go) 程式中會看到 `setupTunnel`，而在程式中也會看到 `setupProxyDialer` ，所以可想而知，在 Server 端，會有 Tunnel 和 Proxy 的功能。  
    - 而在在圖中，可以看到 Angent 的 kubelet 會傳資料給 Tunnel Proxy ，那是怎麼傳的？在 [`v0.9.1/pkg/agent/tunnel/tunnel.go#L168`](https://github.com/rancher/k3s/blob/v0.9.1/pkg/agent/tunnel/tunnel.go#L168) 我們也看到其實 angent 程式也是透過 wss 協定方式進行資料傳輸和 Server 端溝通，因此這樣就可以串起來說，Server 和 Angent 之間是怎麼溝通了。  

![](https://k3s.io/images/how-it-works-k3s.svg)  

## 部署 K3s cluster


## 功能驗證

## 在 ARM 上面部署 K3s Cluster

## 結語



## Reference
- https://k3s.io/
- https://rancher.com/docs/k3s/latest/en/

