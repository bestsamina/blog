+++
title = "實現 IPVS-based K8s service load balancing - 不同 namespace 擁有自己的 external IP"
tags = ["Kubernetes", "K8s", "container", "kube-proxy", "IPVS"]
categories = ["Kubernetes"]
date = 2018-10-15T11:25:11+08:00
date = 2018-08-20T00:20:05+08:00
+++

文章脈絡
- 前置作業
- 環境說明
- K8s 於不同 namespace 擁有自己的 external IP 之環境
   - 部署
   - 測試


## 前置作業

1. 把 IPVS 的 kernel module load 進來

```
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
cut -f1 -d " "  /proc/modules | grep -e ip_vs -e nf_conntrack_ipv4
```

2. 在啟動 kube-proxy 時，參數設為

```
--proxy-mode=ipvs
```

(如果要使用其他演算法，那可以設定 `--ipvs-scheduler=rr` rr 改為其他的)

3. 如果是在 v.10 之前的版本， kube-proxy 要加下面的參數

```
--feature-gates=SupportIPVSProxyMode=true
```

4. 建立 k8s 環境(安裝 tool: https://github.com/kairen/kube-ansible)

## 環境說明

100.67.151.2 master
100.67.151.4 master
100.67.151.5 master
100.67.151.6 worker
100.57.151.8 masters VIP (keepalived)

## K8s 於不同 namespace 擁有自己的 external IP 之環境
### 部署

1. create namespaces
在其中一台 master 主機上執行以下指令  

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: user-a-ns
---
apiVersion: v1
kind: Namespace
metadata:
  name: user-b-ns
EOF
```

2. Create deployments
在其中一台 master 主機上執行以下指令  

```
kubectl run nginx --namespace=user-a-ns --image=nginx --replicas=2

kubectl run nginx --namespace=user-b-ns --image=nginx --replicas=2
```

3. Create service
在其中一台 master 主機上執行以下指令  
external-ip 後的參數需設定為IP Pool (VIP List)裡的IP  
ex. VIP List: 100.67.151.9,100.67.151.10  

```
kubectl expose deployment nginx --namespace=user-a-ns --port 80 --external-ip 100.67.151.9
kubectl expose deployment nginx --namespace=user-b-ns --port 80 --external-ip 100.67.151.10
```

4. 將 VIP List 綁定在網卡上
在其中一台 master 主機上執行以下指令  
暫解將eth0掛上VIP List的資訊，  
ex. VIP List: 100.67.151.9,100.67.151.10  

```
sudo ifconfig eth0:0 100.67.151.9 netmask 255.255.0.0 broadcast 100.67.255.255
sudo ifconfig eth0:1 100.67.151.10 netmask 255.255.0.0 broadcast 100.67.255.255
```

### 測試

1. 從外部 access (可 access 100.67.0.0/16 網段的 client)
打開瀏覽器連 http://100.67.151.9 與 http://100.67.151.10

2. 相關資訊

```
$ ip a
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:26:2d:08:03:a4 brd ff:ff:ff:ff:ff:ff
    inet 100.67.151.2/16 brd 100.67.255.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 100.67.151.9/16 brd 100.67.255.255 scope global secondary eth0:0
       valid_lft forever preferred_lft forever
    inet 100.67.151.10/16 brd 100.67.255.255 scope global secondary eth0:1
       valid_lft forever preferred_lft forever
    inet6 fe80::226:2dff:fe08:3a4/64 scope link 
       valid_lft forever preferred_lft forever

$ ipvsadm -ln
TCP  100.67.151.9:80 rr
  -> 10.244.241.156:80            Masq    1      0          0         
  -> 10.244.241.158:80            Masq    1      0          0         
TCP  100.67.151.10:80 rr
  -> 10.244.241.157:80            Masq    1      0          0         
  -> 10.244.241.159:80            Masq    1      0          0   
```

## Ref

- https://www.hi-linux.com/posts/29792.html
