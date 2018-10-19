+++
title = "IPVS-based Kube-proxy for Scaled Kubernetes Load Balancing"
tags = ["Kubernetes", "K8s", "container", "kube-proxy", "IPVS"]
categories = ["Kubernetes"]
date = 2018-10-15T12:05:37+08:00
+++

> 這篇為 [10月19日 talk](https://www.accupass.com/event/1810120759471826477649) 的文字版， slides 是 https://speakerdeck.com/sufuf3/ipvs-based-kube-proxy-for-scaled-kubernetes-load-balancing 。

內容脈絡  

- Preface (前言)
- Introduction (介紹)
- Kube-Proxy
   - What is Kube-proxy (什麼是 Kube-proxy)
   - Kube-Proxy mode
- IPVS
   - LVS
   - What is IPVS (什麼是 IPVS)
   - IPVS with Netfilter (IPVS 和 Netfilter)
   - IPVS vs iptables (IPVS 與 iptables 的比較)
- IPVS-based Kube-proxy
   - Why using IPVS? (為什麼要用 IPVS)
   - How IPVS-based Kube-proxy work? (IPVS-based Kube-proxy 是怎麼運作的)
   - Run Kube-proxy in IPVS mode (來執行 IPVS mode 的 Kube-proxy)
   - IPVS Service Network Topology 
   - Example
- Implement IPVS-based K8s service load balancing (實現 IPVS-based K8s service load balancing)
- Conclusion (結論)

## Preface (前言)

在一般使用 Kubernetes 的 kube-proxy 情況下，通常都使用 iptables 模式。  
然而在 2015/11/19 ，K8s 團隊的成員 - [Tim Hockin](https://github.com/thockin) 提出了一個 Issue - ["Try kube-proxy via ipvs instead of iptables or userspace"](https://github.com/kubernetes/kubernetes/issues/17470) 而底下也出現了一些討論。  
有趣的是，大家都說 IPVS 比 iptables 好。  
在去年，華為提出了使用 IPVS 優化 service 性能的場景。([ref1](https://github.com/kubernetes/kubernetes/issues/44063), [ref2](https://zhuanlan.zhihu.com/p/37230013))  
今年 2018/06/28 [K8s v1.11](https://github.com/kubernetes/kubernetes/releases/tag/v1.11.0) 也新增了 IPVS-based kube-proxy 的功能。([ref1](https://kubernetes.io/blog/2018/06/27/kubernetes-1.11-release-announcement/), [ref2](https://kubernetes.io/blog/2018/07/09/ipvs-based-in-cluster-load-balancing-deep-dive/))  
  
也因此，本篇就來探討  

1. 到底 IPVS 是什麼
2. 為什麼 kube-proxy 使用 IPVS mode 會比 iptables mode 好
3. kube-proxy 的 IPVS 是怎麼實現的
4. 要怎麼使用 kube-proxy 的 IPVS mode 來實現 Kubernetes service 的 load balance

## Introduction (介紹)

在本篇，將介紹 IPVS 與 Kube-proxy 的 IPVS mode 。並且說明 IPVS 與 iptable 的差異。以及實作。  

## Kube-Proxy
### What is Kube-proxy (什麼是 Kube-proxy)

[![](https://i.imgur.com/apjNBut.png)](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)  
在官網中的這張圖，會看到每一台 Minion node 都有 kube-proxy 這個元件。  
所以 kube-proxy 是什麼樣的功能要長在每一台 node 上呢？  
要談論 kube-proxy 就不得不和 service 一起談論了。  
  
我們知道當 backend 的 pod 需要提供 port 給外部 access 時候，就需要 service 。  
Service 是抽象的一層，主要定義一組 pod 和網路的規則讓外部可以 access 。  
那 service 定義好這些網路的訪問規則，那總要有人來實現這些規則，這就要提到 kube-proxy 了。  
因為 kube-proxy 是實現 service 這功能的關鍵。  
  
![](https://d33wubrfki0l68.cloudfront.net/cc38b0f3c0fd94e66495e3a4198f2096cdecd3d5/ace10/docs/tutorials/kubernetes-basics/public/images/module_04_services.svg)  
  
kube-proxy 介紹：  

- 管理 host 上的網路規則來確保 k8s 的 service 定義的規則
- 執行連線的 forwarding
- 在每一台 node 上都有 kube-proxy
- proxies UDP, TCP and SCTP
- 提供 load balancing
- 專門實現 service
  
那要怎麼達到管理 host 上的網路規則，這就要來看他有提供哪些 mode。  

### Kube-Proxy mode

有[三種](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)：  

- [userspace (older)](https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/userspace):
從圖片中我們可以看到，當外部 Client 請求 service IP 後，會比對 iptales 的 rule 然後將封包轉送到 kube-Proxy 這個 app pod ，再由 kube-proxy 處理封包的轉送到後端的 pod。  
![](https://d33wubrfki0l68.cloudfront.net/b8e1022c2dd815d8dd36b1bc4f0cc3ad870a924f/1dd12/images/docs/services-userspace-overview.svg)

- [iptables (faster)](https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/iptables):
iptables 就比 userspace 好一些，因為 client 請求 service IP 後，就交給 iptables 處理封包的轉送。而下 iptables rule 就是當 api-server 拿到 service 的 yaml 檔後，api-server 會給 kube-proxy 來處理，而 kube-proxy 就來下 iptables 的 rule 囉！  
![](https://d33wubrfki0l68.cloudfront.net/837afa5715eb31fb9ca6516ec6863e810f437264/42951/images/docs/services-iptables-overview.svg)

- [ipvs (experimental)](https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/ipvs):
和 iptables mode 方式雷同，但是是使用 kernel space 的 IPVS 方法處理要轉送的封包，不受限於 iptables 的規則。  
![](https://d33wubrfki0l68.cloudfront.net/3ece58e40119539d5e999fe137e79b5015c24275/77563/images/docs/services-ipvs-overview.svg)

## IPVS

在講 IPVS 之前，我們不得不提到 LVS。  

### LVS

LVS 全名是 `Linux Virtual Server`。它是在 cloud 上或是真實 server 上的高可擴展與高可用的 server。通常會伴隨 load balancer 一起在 Linux OS 上運行。  
最有名的就是這張圖片拉！  
![](https://i.imgur.com/9vLMHjv.png)  
對外面的 user 來說，他 access 的是個機器，但對集群來說 user access 到的其實是虛擬的 server，而虛擬的 server 是由一群在同個 LAN 或是 WAN 中的真實的 server 和 Load Balancer 所組成的。  
  
那 Virtual Server 總是要有個 IP 給外面的 user 知道吧！所以[官方網站](http://www.linuxvirtualserver.org/HighAvailability.html)也提供下面這張圖，由圖所知，就是會有個 Virtual IP 拉！  
![](https://i.imgur.com/EU0gAUv.png)  

那 LVS 和 IPVS 的關係又是什麼呢？  
這要來看看 [LVS 的框架](http://www.linuxvirtualserver.org/about.html)了。  
![](https://i.imgur.com/lE6iI9F.png)  
由圖中所知，IPVS 是在 LVS 框架中最底層的位置。也就是說 LVS 是 base on IPVS 來實現的拉！那我們就要來看看 IPVS 了。  

### What is IPVS (什麼是 IPVS)

- IPVS 全名 IP Virtual Server
- 實現 L4 的 load balancing
- 也稱 Layer-4 switching
- 主要在 cluster 前的實體主機上運行
- 直接請求 base on service 的 TCP / UDP 到真實的 server 
- 使真實 server 的 service 顯示為有一個 IP 的虛擬 service 
- 實現為 Netfilter 框架上的 module 
- Based on kernel 中的 hash tables
- Kernel source code: net/netfilter/ipvs
- ipvsadm 是管理 IPVS 的工具

### IPVS with Netfilter (IPVS 和 Netfilter)

那我們知道 IPVS 是使用 Netfilter 的 module，那就來看看 IPVS 和 Netfilter 之間的關係吧！  
> Ref: http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/LVS-HOWTO.filter_rules.html, https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture, http://www.178linux.com/13570  

![](https://i.imgur.com/i60QKw4.png)  

1. 當 client 的請求從外部進到 Load balencer 的 kernel space 時，會先到 PREROUTING 的 chain。
2. 接著 Route 會依據進來的封包 destination address 判斷是不是本機的(通常是 VIP 存在於本機的 network interface 上)。如果是本機的封包，就將封包往 INPUT chain 送。
3. 而 ip_vs_in 掛勾到 LOCAL_IN，所以最終會給 IPVS 來檢查這個進來的封包。
(module 以優先權來決定，優先權低的先看封包，IPVS 的優先權高於 iptables 所以會先給 iptable 看，之後給 IPVS 看。)
4. 在 LOCAL_IN Hook 如果 ip_vs_in 發現這個封包都有在 IPVS 規則中，就會 trigger INPUT chain 之後到 POSTROUTING chain 就結束。
由上面可知，使用 IPVS 不會接觸到 iptables 的 rule。

### IPVS vs iptables (IPVS 與 iptables 的比較)

大致了解 IPVS ，那 IPVS 和 iptables 有什麼差別呢？  
他們其實都是使用 Netfilter ，來讓封包達到轉送的機制。  
但實作面卻是不一樣的。  
在 INPUT chain 這邊，不論是 IPVS 或是 iptables 都會到 userspace 來進行封包解析，來看看封包要往哪邊走。  
然而，就是在判斷封包要往哪邊走，這邊的方法兩者使用的方式是不一樣的。  
iptables 規則設定是：n 張 table，每張 table 內有 m 個 chain ，每個 chain 中有 rule。如圖(source: https://www.thegeekstuff.com/2011/01/iptables-fundamentals/)  
![](https://static.thegeekstuff.com/wp-content/uploads/2011/01/iptables-table-chain-rule-structure.png)  
雖然封包不會跑過所有的 Table 和 chain。  
但 rules 通常會是很多的。雖然封包只要被拆解一次，但封包在比對每個 rule 時，都要再看要用進來的這個封包用什麼欄位來和目前輪到的 rule 進行比對。  
這也是為什麼我們在下 rule 的時候很重視 rule 的順序性。(所以 rule 也會因人的設定好不好，效能也會有差異)  
IPVS 在判斷上面就簡單很多，他是用 hash table(雜湊表)，在時間複雜度上，通常是是 O(1)，即便遇到最差的情況(worst case)，也只是 O(n)。理論可以參考：https://zh.wikipedia.org/wiki/%E5%93%88%E5%B8%8C%E8%A1%A8 (至於怎麼做，就要翻 source code 了)  

就上面的兩種分析封包判斷要往哪走的方法，我們也可以了解到他們在封包轉送上的效率是有差別的。
  
而華為在去年的[演講](https://www.slideshare.net/LCChina/scale-kubernetes-to-support-50000-services)中，也有提供 k8s kube-proxy 中使用 iptables 和 IPVS mode 的差異。(From 華為投影片)  
![](https://i.imgur.com/HrW0m0z.png)
![](https://i.imgur.com/uUFNSMa.png)

## IPVS-based Kube-proxy
### Why using IPVS? (為什麼要用 IPVS)

承接上面，我們可以得出為什麼要使用 IPVS 的結論([ref](https://www.cncf.io/wp-content/uploads/2018/08/CNCF-Webinar_-Kubernetes-1.11-1.pdf ))。  

1. 有較佳的 performance (Hashing 和 Chain 的效率)
2. 有比較多的 load balancing 演算法(多達 8 種)
3. 支援 server 運行狀況檢查和連接重試
4. 支援 sticky session
5. iptables 操作在大規模集群中顯得比較慢

### How IPVS-based Kube-proxy work? (IPVS-based Kube-proxy 是怎麼運作的)

那是怎麼運作的呢？我們回頭再來看這張圖片  
![](https://d33wubrfki0l68.cloudfront.net/3ece58e40119539d5e999fe137e79b5015c24275/77563/images/docs/services-ipvs-overview.svg)  
我們知道在 Virtual Server 這邊是使用 IPVS ，然後透過他管理的 virtual server table 來讓 Netfilter 把封包送到對的地方。  
這就是他的運作原理囉！  

### Run Kube-proxy in IPVS mode (來執行 IPVS mode 的 Kube-proxy)

那在 kube-proxy 中我們要怎麼使用 IPVS mode 呢？([ref](https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/ipvs))  

**1. 首先我們要把 IPVS 的 kernel module load 進來**

```
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
cut -f1 -d " "  /proc/modules | grep -e ip_vs -e nf_conntrack_ipv4
```

**2. 接著在啟動 kube-proxy 時，參數設為**

```
--proxy-mode=ipvs
```

(如果要使用其他演算法，那可以設定 `--ipvs-scheduler=rr` rr 改為其他的)  

**3. 如果是在 v1.10 之前的版本， kube-proxy 要加下面的參數**

```
--feature-gates=SupportIPVSProxyMode=true
```

### IPVS Service Network Topology 

當建立 ClusterIP type 的 service 時，IPVS proxier 將執行以下三項操作：

- 確保節點中存在 dummy 接口，默認為 kube-ipvs0
- 將 service IP 綁到 dummy 接口
- 分別為每個 service IP 建立 IPVS virtual servers

### Example

- v1.10.x

```
# kubectl describe svc nginx -n a-ns
Name:              nginx
Namespace:         a-ns
Labels:            run=nginx
Annotations:       <none>
Selector:          run=nginx
Type:              ClusterIP
IP:                10.105.12.124
External IPs:      100.67.151.9
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.241.156:80,10.244.241.158:80
Session Affinity:  None
Events:            <none>

# ip a
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:26:2d:08:03:a4 brd ff:ff:ff:ff:ff:ff
    inet 100.67.151.2/16 brd 100.67.255.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 100.67.151.9/16 brd 100.67.255.255 scope global secondary eth0:1
       valid_lft forever preferred_lft forever
18: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether e6:f5:f6:9f:0b:9a brd ff:ff:ff:ff:ff:ff
    inet 10.96.0.1/32 brd 10.96.0.1 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.96.0.10/32 brd 10.96.0.10 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.105.12.124/32 brd 10.105.12.124 scope global kube-ipvs0
       valid_lft forever preferred_lft forever

# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.105.12.124:80 rr
  -> 10.244.241.156:80            Masq    1      0          0
  -> 10.244.241.158:80            Masq    1      0          0
TCP  100.67.151.9:80 rr
  -> 10.244.241.156:80            Masq    1      0          0
  -> 10.244.241.158:80            Masq    1      0          0
```

- v1.11.x

```
# kubectl describe svc nginx -n a-ns
Name:              nginx
Namespace:         a-ns
Labels:            run=nginx
Annotations:       <none>
Selector:          run=nginx
Type:              ClusterIP
IP:                10.105.12.124
External IPs:      100.67.151.9
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.241.156:80,10.244.241.158:80
Session Affinity:  None
Events:            <none>

# ip a
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:26:2d:08:03:a4 brd ff:ff:ff:ff:ff:ff
    inet 100.67.151.2/16 brd 100.67.255.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
18: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether e6:f5:f6:9f:0b:9a brd ff:ff:ff:ff:ff:ff
    inet 10.96.0.1/32 brd 10.96.0.1 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.96.0.10/32 brd 10.96.0.10 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.105.12.124/32 brd 10.105.12.124 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 100.67.151.9/16 brd 100.67.255.255 scope global kube-ipvs0
       valid_lft forever preferred_lft forever

# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.105.12.124:80 rr
  -> 10.244.241.156:80            Masq    1      0          0
  -> 10.244.241.158:80            Masq    1      0          0
TCP  100.67.151.9:80 rr
  -> 10.244.241.156:80            Masq    1      0          0
  -> 10.244.241.158:80            Masq    1      0          0
```

## Implement IPVS-based K8s service load balancing (實現 IPVS-based K8s service load balancing)

請參考這篇[筆記](https://bestsamina.github.io/posts/2018-10-15-hands-on-k8s-kube-proxy-w-ipvs-lb/)  

## Conclusion (結論)

回過頭來，再次複習一下，這次我們要探討的是 kube-proxy 中的 IPVS，我們知道 kube-proxy 是要處理封包針對不同的 service 做轉送，是處理網路規則的。講白話一點就是， pod 開出給外面 access 的 port 或是對應的 IP 等， repuest 進來， kube-proxy 要把這個封包送到對應的 pod，讓 pod 處理，在回應出去。  
而可以做到封包轉送這件事的在 kube-proxy mode 中有三種方式可以做到。  
目前普遍是 iptables。但 IPVS 也可以做到，而且效能更好。  
以下是 IPVS 與本次介紹的一些重點整理  

- IPVS 是 LVS 中的 L4 負載均衡器
- IPVS提供
    - 在 cluster 中更好的可擴展性和效能
    - 有比 iptables 更多的 load balancing 演算法
    - server 的健康檢查和連接重試等
- 我們可以使用 kube-proxy 的 IPVS 模式
- 了解 kube-proxy 的 IPVS mode 的運作原理

## 其他

在搜尋 IPVS 的過程中，發現了 [BPF(Berkeley Packet Filter)](https://lwn.net/Articles/747551/) 一個即將要取代 iptables 的工具。 cilium 這篇 blog 也指出為何要用 BPF。 而 Facebook 也發現 BPF 效能比 IPVS 好，所以換成 BPF。  
有時間看來要研究一下。  

- Blog https://cilium.io/blog/2018/04/17/why-is-the-kernel-community-replacing-iptables/
- FB 簡報：https://www.netdevconf.org/2.1/slides/apr6/zhou-netdev-xdp-2017.pdf

## 參考

- https://gigenchang.wordpress.com/2014/04/19/10%E5%88%86%E9%90%98%E5%AD%B8%E6%9C%83iptables/
