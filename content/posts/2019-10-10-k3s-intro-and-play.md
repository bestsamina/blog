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

- 在 Server 端： 只需要 512 MB 的 RAM
- 在 Node 端：只需要 75MB 的 RAM
- Disk 大小只需 200MB

而且可以用在 x86_64, ARMv7, ARM64 等平台架構。  

### K3s 架構

K3s 是將 Kubernetes 的元件 (包含 kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, kube-proxy) 打包在一起，並以 server 和 agent 的 model 形式呈現。  

因此 K3s 的架構可以看到如下， 如圖所示，可以到有 Server 與 Agent。  

- Server: (Kubernetes 管理元件 + SQLite + Tunnel Proxy)
    - API Server: 來提供 API 的，並且會將資料存到資料庫(預設使用 SQLite)。
    - SQLite: 是存放 Cluster status 資料的地方。同時也有支援 etcd, Mysql, Postgres (v0.9.1 ref: [server-options K3S_STORAGE_ENDPOINT](https://rancher.com/docs/k3s/latest/en/installation/#server-options))
    - Scheduler: 和 Kubernetes 一樣有選 Node run pod 的功能。
    - Controller Manager: 算是 K3s 的後端程式，是處理 Kubernetes resource 的 controller manager。
- Agent: (Kubernetes data plan 元件 + Tunnel Proxy)
    - Kube-proxy: 就是做 forwrading 用的。
    - Flannel: Pod 溝通用的網路。
    - Kubelet: 啟動 container 用。
- Tunnel Proxy of Server & Agent: 
    - 在 Server 的 [`pkg/daemons/control/tunnel.go`](https://github.com/rancher/k3s/blob/v0.9.1/pkg/daemons/control/tunnel.go) 程式中會看到 `setupTunnel`，而在程式中也會看到 `setupProxyDialer` ，所以可想而知，在 Server 端，會有 Tunnel 和 Proxy 的功能。  
    - 而在在圖中，可以看到 Agent 的 kubelet 會傳資料給 Tunnel Proxy ，那是怎麼傳的？在 [`v0.9.1/pkg/agent/tunnel/tunnel.go#L168`](https://github.com/rancher/k3s/blob/v0.9.1/pkg/agent/tunnel/tunnel.go#L168) 我們也看到其實 agent 程式也是透過 wss 協定方式進行資料傳輸和 Server 端溝通，因此這樣就可以串起來說，Server 和 Agent 之間是怎麼溝通了。  

![](https://k3s.io/images/how-it-works-k3s.svg)  

## 部署 K3s cluster

在部署 Cluster 之前需要先準備的環境如下：  
兩台 Host (VM 或主機) ， Server VM 的 RAM 至少需要 512 MB，Work Node VM 的 RAM 至少需要 75MB，並且兩台主機都已經安裝好 Docker 了。  

### 1. 首先在 Master 上安裝 K3s Server

切換到 root 使用者，並執行以下的 Command  

```
IPADDR=$(ip a show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f1)
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v0.9.1 INSTALL_K3S_EXEC="--docker --node-ip=${IPADDR} --flannel-iface=enp0s8 --write-kubeconfig-mode 644 --no-deploy=servicelb --no-deploy=traefik" sh -
```

我們會執行 https://get.k3s.io 這程式裡面的腳本，而帶入的 option 有：  

- `INSTALL_K3S_VERSION`: 指定安裝 K3s 的版本
- `INSTALL_K3S_EXEC`: 指定 `k3s server` 執行的 option
    - `--docker`: 選用 Docker 當作 K3s 的 container 引擎。
    - `--node-ip=${IPADDR}`: 設定 Master Node 的 IP
    - `--flannel-iface=enp0s8`: 設定 flannel 溝通用的 network interface ，請記得這邊的 `enp0s8` 改成自己主機的 network interface。
    - `--write-kubeconfig-mode 644`: 設定 kubeconfig 的權限 mode。
    - `--no-deploy=servicelb --no-deploy=traefik`: 設定不要部署 packaged 的 components。

這樣 K3s server 就安裝完了。  
可以使用 `systemctl status k3s` 與 `journalctl -u k3s` 查看 K3s 的 Service 狀態。  

並在進行下一步驟前，執行 `echo "export NODE_TOKEN=$(sudo cat /var/lib/rancher/k3s/server/node-token)"` 紀錄 Cluster Node 之間認證用的 Token。  

### 2. 安裝 K3s Node

先在 `/etc/hosts` 加入 master node 與 IP 的對應。  

```
echo "$K3SMASTER_IPADDRESS       $K3SMASTER_HOSTNAME" | sudo tee -a /etc/hosts
```

接著切換到 root 使用者後，貼上剛剛 Master Node 上取得的認證 token，如下  

```
export NODE_TOKEN=K10571d20534c867fe6ce8d... 
```

並執行以下的 Command 來安裝 K3s agent  

```
export K3SMASTER_IPADDRESS=...
IPADDR=$(ip a show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f1)
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v0.9.1 INSTALL_K3S_EXEC="--docker --node-ip=${IPADDR} --flannel-iface=enp0s8" K3S_URL=https://${K3SMASTER_IPADDRESS}:6443 K3S_TOKEN=${NODE_TOKEN} sh -
```

在執行 https://get.k3s.io 這程式裡面的腳本中，`INSTALL_K3S_EXEC` 而外帶入的 option 有：  

- `K3S_URL`: 是讓 agent 知道他要和哪個 server 連線
- `K3S_TOKEN`: 是用來認證用的


這樣 K3s agent 就安裝完了。  
可以使用 `systemctl status k3s-agent` 查看 k3s-agent 的 Service 狀態。 

## 快速安裝法

如果不想要依照以上方法一步一步操作的話，也可以參考 https://github.com/sufuf3/k3s-lab repository 來嘗試快速部署起一個 K3s Cluster。  
在使用 https://github.com/sufuf3/k3s-lab 這個方法部署，環境需要具備 VirtualBox 以及 Vagrant ，並且主機尚有 1.6G 的 RAM 可以使用。  
安裝方法如下：  

1. 在 Host 上啟動兩台 VM 並且在 Master 的 VM 中安裝 K3s server。

```
$ sh deploy-vagrant.sh
```

2. 在第一步驟完成後，會得到類似 `export NODE_TOKEN=K10571d20534c867fe6ce8d...` 的字串，將這字串複製後，透過 `vageant ssh node1` 進入 node 的 VM 並且執行以下的指令  

```
sudo su -
cd /home/vagrant/
export NODE_TOKEN=K10571d20534c867fe6ce8d... // 是複製的字串
sh install-k3s-node.sh ${NODE_TOKEN}
```

這樣就完成以上部署 K3s Cluster 了。接下來我們來驗證 K3s Cluster 是否正常吧！  

## 功能驗證

首先先進入 Master Node，你可以透過 k3s 指令看到它有支援的 commands，如下，因此我們可以直接使用 kubectl, crictl, ctr：  
而因為我們使用 Docker 為我們的 container 引擎，所以本篇只用 kubectl 來和大家一起驗證。  

```
$ k3s -h
NAME:
   k3s - Kubernetes, but small and simple

USAGE:
   k3s [global options] command [command options] [arguments...]

VERSION:
   v0.9.1 (755bd1c6)

COMMANDS:
   server   Run management server
   agent    Run node agent
   kubectl  Run kubectl
   crictl   Run crictl
   ctr      Run ctr
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --debug        Turn on debug logs
   --help, -h     show help
   --version, -v  print the version
```

我們可以透過 kubectl 來查看 Cluster node 資訊與 component 的狀態。  

```
$ kubectl get no -o wide
NAME     STATUS   ROLES    AGE   VERSION         INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
master   Ready    master   97m   v1.15.4-k3s.1   192.168.0.200   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   containerd://1.2.8-k3s.1
node1    Ready    worker   93m   v1.15.4-k3s.1   192.168.0.201   <none>        Ubuntu 18.04.3 LTS   4.15.0-64-generic   containerd://1.2.8-k3s.1

$ kubectl get componentstatus
NAME                 STATUS    MESSAGE   ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
```

也可以用 kubectl 來跑一個 deployment 來啟動一個 nginx ，如以下 Lab。  

```
$ kubectl run mynginx --image=nginx --replicas=1 --port=80
deployment.apps/mynginx created
$ kubectl expose deployment mynginx --port 80
service/mynginx exposed

$ kubectl get deploy,po,svc -o wide -l run=mynginx
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES   SELECTOR
deployment.extensions/mynginx   1/1     1            1           115s   mynginx      nginx    run=mynginx

NAME                           READY   STATUS    RESTARTS   AGE    IP          NODE    NOMINATED NODE   READINESS GATES
pod/mynginx-568f57494d-nsc8r   1/1     Running   0          115s   10.42.1.2   node1   <none>           <none>

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/mynginx   ClusterIP   10.43.220.117   <none>        80/TCP    27s   run=mynginx

$ curl 10.43.220.117
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

如上，其實使用方式和 Kubernetes 使用原本資源方法是一樣的，所以要更換 install 的工具，其實並不是太困難。  

## 移除安裝

在 K3s Server 端執行  

```
systemctl stop k3s && /usr/local/bin/k3s-uninstall.sh
```

；在 k3s agent 端執行  

```
systemctl stop k3s-agent && /usr/local/bin/k3s-agent-uninstall.sh
```


## 在 ARM 上面部署 K3s Cluster

而在 ARM 上面部署，也是雷同的，不過有以下幾點是需要注意的  

- 若是要編譯 Kernel 請確定 vxlan, iptables, netfilter, ip_vs 等必要的 module 都有開啟。
- 若需要安裝 Docker ，請確定 Cgroup 在該編譯的 Kernel 版本是沒有 issue 的。也請確定使用的 Kernel 版本是符合 Kubernetes 官網上的描述。

## 結語

因緣際會下，接觸 K3s ，其實蠻有趣的，覺得邊緣運算和雲端運算有共同類似的技術可以交流，也是很開心的。最近也光了一些時間，翻了一下 K3s 的 code，覺得有趣。之後若有時間，也許會來寫一篇深入研究 K3s 源碼的分享。


## Reference
- https://k3s.io/
- https://rancher.com/docs/k3s/latest/en/

