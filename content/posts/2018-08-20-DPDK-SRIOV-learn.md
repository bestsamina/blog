+++
title = "關於加速 NFV Data Plane 的 SR-IOV 與 DPDK"
tags = ["ovs", "dpdk", "network", "sr-iov"]
categories = ["Network"]
date = 2018-08-20T00:20:05+08:00
+++

> 本篇是個初步理解 SR-IOV 與 DPDK 兩個技術的筆記。  
內容綜合官網的說明、參考 https://www.metaswitch.com/blog/accelerating-the-nfv-data-plane 的描述、自我的理解。 如有任何描述錯誤，敬請指教。  

本篇是個個人筆記。文章脈絡大致如下：  

- 一句話形容 SR-IOV 與 DPDK
- SR-IOV 的源起
- SR-IOV 的技術理解
- DPDK 的緣起
- DPDK 的技術理解

## 一句話形容 SR-IOV 與 DPDK

一句話秒懂的形容：  

- SR-IOV: SR-IOV 是一種技術，使用這項技術，就讓虛擬主機或是虛擬化的容器如同直接接上實體網卡一樣，封包一進到實體主機的網卡後，就等於直接進到虛擬主機或是虛擬化的容器中。
- [DPDK](https://www.dpdk.org/): DPDK 這項技術，可以讓進到實體主機的網路封包，直接跳過 Linux Kernel 層，送到 User space 的 Applications 做處理。
  
以上兩種技術，皆是 Intel 最先提出的。  

## SR-IOV 的源起

這一切要回到當初，虛擬主機的技術講起。  
假設在一個實體主機上有幾台 VM，此時當實體主機上的網卡 (NIC) 收到一個封包的時候，會向 CPU 發送一個中斷請求 (interrupt request = [IRQ](https://linuxgazette.net/issue38/blanchard.html))，然後 CPU 就必須中斷目前在做的事情，轉而處理這封包，把封包送到對的 VM 中。而封包那麼多，中斷就會一直發生，這樣也會降低 CPU 的效能。而且不只有實體主機上這個 CPU 的 IRQ 問題，VMM (Virtual Machine Manager) 的 CPU 也會被中斷，只要它辨識出 package 要送往的 VM ，它就會向 VM 自己的 CPU 請求中斷，叫 VM 的 CPU 來處理這封包。  
  
![](https://i.imgur.com/ltZut6A.png)  

> 圖片來自：https://software.intel.com/file/1919
  
所以，Intel 為了解決實體層 CPU 被打斷的問題，它推出了 Virtual Machine Device Queues (VMDq) 這項技術。如名稱，VMDq 可以讓 VMM 在 NIC 中分配一個直接的 queue 給 VM。  
  
![](https://i.imgur.com/2Qk6CXc.png)  

> 圖片來自：https://software.intel.com/file/1919  
  
不過有個麻煩，就是 data-plane-centric VNF 仍然會降低效能。畢竟都打算直接從實體網卡送到 VM ，幹嘛還要經過 VMM 的 vswitch 這一層呢？所以就來看看 SR-IOV。  

## SR-IOV 的技術理解

- 全名：Single Root I/O Virtualization and Sharing Specification
- 規範由 [PCI-SIG (Peripheral Component Interconnect Special Interest Group)](https://pcisig.com/) 定義和維護 (SR-IOV 是在 2007 年 09 月被提出，且是基於 [AMD (IOV) 的技術 input-output memory management unit (IOMMU) 標準](https://support.amd.com/TechDocs/48882_IOMMU.pdf)。)
- 只有特定網卡的型號才有支援
- SR-IOV 的兩種功能
    - 物理功能 (Physical Function, PF)
        - 用於支援 SR-IOV 功能的 PCIe 功能
        - 網卡的每個實體 port 至少會有一個 PF
    - 虛擬功能 (Virtual Function, VF)
        - 「精簡」的PCIe功能
        - 不支援實體裝置的管理
        - 支援的 VF 數量因網卡型號而不同
        - 每個VF都有自己的 PCI 配置空間

至於功能的對應，如圖(圖片來源：https://www.intel.com/content/dam/doc/application-note/pci-sig-sr-iov-primer-sr-iov-technology-paper.pdf)： 
白話文的講法就是，每個實體 port 都有個 PF，每個 PF 有多個 VF (數量依型號而定)，而 VF 對應到 VM 或是虛擬化的 continer，都會有個 VF Driver 。當然，若是這種做法，是可以直接跳過 VMM 的 vswitch 了，想當然，也可以不用打斷 VMM 的 CPU 處理。  
當然，還有另外的做法，也是可以接到 vswitch 的，但就是實體 port 上的 PF 接到 vswitch 上的 PF Driver ，接著 VM 或是虛擬化的 continer 接到它們的虛擬網卡。  
![](https://i.imgur.com/bimcJYu.png)  
  
而 Memory Translation 技術(如：Intel® VT-x & Intel® VT-d)提供 hardware assisted 技術，允許直接和 VM 進行 DMA transfers，這樣就可以繞過 VMM 的 switch。  
每個 VF 都有自己的 PCI configuration space ，而 PCI configuration space 是透過 Base Address Registers (BARs) 完成。而 VMM 會將 系統上的 PCI configuration space mapping 一份該 VM 的給該 VM。(如圖)  
![](https://i.imgur.com/8c9WFYN.png)  
  
> 補充沒有很懂的知識：[PCI](https://wiki.osdev.org/PCI), [PCIe](https://zh.wikipedia.org/wiki/PCI_Express)  
  
## DPDK 的緣起

在 SDN 的時代，我們還是需要 vSwitch，但 vSwitch 是慢的， 所以之後有人提出 Megaflow ，在 OVS kernel 中裝 secondary flow cache，來提高效能。  
不過不管怎麼樣，OVS 仍然靠 kernel space data path 轉送 packets。然而當封包量多的時候，這種情況會花費較高的 IPC (Inter-Process Communication)，網路吞吐量就會受到 Linux network stack 的限制。  
![](https://i.imgur.com/8JjcMPJ.png)  

> 圖片與文章參考來源：https://software.intel.com/en-us/articles/open-vswitch-with-dpdk-overview  
  
Linux kernel 的 overhead 包含：  

- System calls
- Context switching on blocking I/O
- Data copying from kernel to user space
- Interrupt handling in kernel 

所以 DPDK 就出來了。  

## DPDK 的技術理解

- 全名：Data Plane Developer Kit 
- 2011/09 被 Intel 提出，2013 以 open source 釋出
- license 是 BSD
- 他是加速封包處理的框架
- 他可以跑在任何的 processors 上，原本只有支援的 CPU 是 intel x86 ，不過現在延伸到 IBM power 和 ARM 都可以。

在概念上，它就是直接將封包跳過 kernel 層，直接到 user space 處理。  
![](https://i.imgur.com/KTBX4Dk.png)  
> 圖片與文章參考來源：https://software.intel.com/en-us/articles/open-vswitch-with-dpdk-overview

如同下圖這張表的差異比較。  
![](https://i.imgur.com/zWlbD2m.png)
  
然而到底怎麼做到的？  
DPDK 會在每個特殊的環境中建立 Environment Abstraction Layer (EAL)。  
這個環境抽象層(EAL)，是通過 make ，在 Linux 的 user space 編譯完成。(就是這步：`sudo make install T=$DPDK_TARGET DESTDIR=install`)  
EAL 會提供一個通用的 interface (就是 ./usertools/...) ，它隱藏內部很多的 libraries 和 Apps.  
EAL 提供的 service 有：  

- DPDK loading and launching
- Support for multi-process and multi-thread execution types
- Core affinity/assignment procedures
- System memory allocation/de-allocation
- Atomic/lock operations
- Time reference
- PCI bus access
- Trace and debug functions
- CPU feature identification
- Interrupt handling
- Alarm operations
- Memory management (malloc)

比方說使用 `igb_uio.ko` ，就是 igb 這個 intel 網卡驅動程式借助 UIO (Userspace IO) 技術，將那張網卡硬體mapping 到 user space 中。  
當綁定後， PMD（Poll Mode Drivers）會直接 accesses RX 和 TX 的 descriptors，不會有任何中斷。以便在使用者的應用程式中快速接收，處理和傳送 package。那他有兩種模式 (run-to-completion 和 pipeline)。  
  
- pipeline 模式就是透過 API 來不斷輪詢在多個 port RX 之間。
- run-to-completion 模式就是透過 API 來不斷輪詢特定 port 的 RX 。

所以就要指定特定的 CPU 來處理 DPDK。  
因為 packet buffers 需要使用到 很大的 memory pool allocation ，所以需要使用到 Hugepage。這意味著 HUGETLBFS 選項需要在 running 的 kernel 中啟用。
使用 hugepage allocations ，可以使用更少 page 並提高效能，可以減少 TLBs(Translation Lookaside Buffers)，減少虛擬的 page address 轉換到物理的 page address。如果沒有 hugepage，TLB 缺失率會變高，因為會發生在使用標準的 4k page size，降低。
而且 Hugepage 可以讓 multiple DPDK processes 一起使用，  
讓不同的 process 共同 access 同一塊 shared memory，inter-process communication (IPC) 也會變得更簡單。  



## 參考文件

- https://www.slideshare.net/garyachy/dpdk-44585840
- https://www.metaswitch.com/blog/accelerating-the-nfv-data-plane
- https://core.dpdk.org/doc/
- https://www.intel.com.tw/content/www/tw/zh/support/articles/000005722/network-and-i-o/ethernet-products.html
- https://doc.dpdk.org/guides/index.html
- https://dpdk-docs.readthedocs.io/en/latest/index.html
- https://feisky.xyz/post/2016-04-24-dpdk-introduction/
