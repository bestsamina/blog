+++
title = "[TIL] VPP 相關 Troubleshooting 紀錄"
date = 2019-03-08T14:33:14+08:00
tags = ["VPP", "Network", "todayilearned", "til"]
categories = ["Todayilearned"]
+++

# VPP 目前只能使用 2M 的 Hugepage

依據目前以下的兩份文件，是建議使用 2M 的 Hugepage，自己測試時，即便已經宣告 1G 的 Hugepage 是沒辦法執行的  

- https://fdio-vpp.readthedocs.io/en/latest/gettingstarted/users/configuring/hugepages.html
- https://media.readthedocs.org/pdf/adenisco-vpp-docs/vpp-13957/adenisco-vpp-docs.pdf - 2.1.2

# 安裝 VPP 後，DOWN 的 Network Interface 使用 `ifconfig` (or `ip a` ) 會看不到。

在這之前，不僅安裝了 VPP ，還更新 i40e 的 driver  
在眾多的程序後，發現 `ifconfig` 看不到那些有支援 SR-IOV VF 的 interface，以為是 driver 被我更新更新到壞掉了。  
一直重新 `make install` driver 但還是沒效。  
仔細一看，發現 `igb` 的另外一個 DOWN 的 Network Interface 也不見了。 
不過硬體資訊有看到  
  
```sh
$ sudo lshw -class network -businfo
[sudo] password for winlab:
Bus info          Device           Class      Description
=========================================================
pci@0000:01:00.0                   network    Ethernet Controller X710 for 10GbE SFP+
pci@0000:01:00.1                   network    Ethernet Controller X710 for 10GbE SFP+
pci@0000:01:00.2                   network    Ethernet Controller X710 for 10GbE SFP+
pci@0000:01:00.3                   network    Ethernet Controller X710 for 10GbE SFP+
pci@0000:08:00.0  enp8s0           network    I210 Gigabit Network Connection
pci@0000:09:00.0                   network    I210 Gigabit Network Connection
```
  
這樣就開始研究，為何 DOWN 的 Interface 會自己不見。  
使用 dmesg 也看不出所以然  

```sh
$ dmesg | grep igb
[    1.569802] igb: Intel(R) Gigabit Ethernet Network Driver - version 5.4.0-k
[    1.569803] igb: Copyright (c) 2007-2014 Intel Corporation.
[    5.522996] igb 0000:08:00.0: added PHC on eth0
[    5.522997] igb 0000:08:00.0: Intel(R) Gigabit Ethernet Network Connection
[    5.522999] igb 0000:08:00.0: eth0: (PCIe:2.5Gb/s:Width x1) 8c:ea:1b:30:da:09
[    5.523043] igb 0000:08:00.0: eth0: PBA No: 200500-000
[    5.523044] igb 0000:08:00.0: Using MSI-X interrupts. 4 rx queue(s), 4 tx queue(s)
[    5.757962] igb 0000:09:00.0: added PHC on eth1
[    5.757963] igb 0000:09:00.0: Intel(R) Gigabit Ethernet Network Connection
[    5.757965] igb 0000:09:00.0: eth1: (PCIe:2.5Gb/s:Width x1) 8c:ea:1b:30:da:0a
[    5.758010] igb 0000:09:00.0: eth1: PBA No: 200500-000
[    5.758012] igb 0000:09:00.0: Using MSI-X interrupts. 4 rx queue(s), 4 tx queue(s)
[    5.759074] igb 0000:08:00.0 enp8s0: renamed from eth0
[    5.780354] igb 0000:09:00.0 enp9s0: renamed from eth1
[   11.768618] igb 0000:08:00.0 enp8s0: igb: enp8s0 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX/TX
[   17.134762] igb 0000:09:00.0: removed PHC on enp9s0
```

使用 `journalctl` 看的時候，發現一開機啟動 OS 確實有出現 Interface ，但當啟用了 VPP 後，DOWN 的 interface 因為 kernel 的被呼叫，導致 `removed PHC`，而就此也看不到 Network Interface 了！  

```sh
Mar 07 17:21:30 kubecord-a vpp[2189]: /usr/bin/vpp[2189]: vlib_pci_bind_to_uio: Skipping PCI device 0000:08:00.0 as host interface
Mar 07 17:21:30 kubecord-a /usr/bin/vpp[2189]: vlib_pci_bind_to_uio: Skipping PCI device 0000:08:00.0 as host interface enp8s0 is
Mar 07 17:21:30 kubecord-a kernel: igb 0000:09:00.0: removed PHC on enp9s0
```
  
目前因為還沒仔細研究 VPP 也沒有特別要使用到，因此先移除。移除 VPP 並重開機後，DOWN 的 Interface 就可以看到了。  
  
# 在 `/etc/default/grub` 的 `GRUB_CMDLINE_LINUX` 同時宣告 Hugepages 1G 和 2M ，導致 kubernetes get no 不到

使用 `journalctl -f -u kubelet` 看到  
  
```sh
Mar 11 13:17:56 kubecord-a kubelet[8532]: E0311 13:17:56.621316    8532 kubelet_node_status.go:92] Unable to register node "kubecord-a" with API server: Node "kubecord-a" is invalid: [status.capacity.hugepages-1Gi: Invalid value: resource.Quantity{i:resource.int64Amount{value:8589934592, scale:0}, d:resource.infDecAmount{Dec:(*inf.Dec)(nil)}, s:"", Format:"BinarySI"}: may not have pre-allocated hugepages for multiple page sizes, status.allocatable.hugepages-2Mi: Invalid value: resource.Quantity{i:resource.int64Amount{value:2147483648, scale:0}, d:resource.infDecAmount{Dec:(*inf.Dec)(nil)}, s:"2Gi", Format:"BinarySI"}: may not have pre-allocated hugepages for multiple page sizes, status.allocatable.memory: Invalid value: resource.Quantity{i:resource.int64Amount{value:56150003712, scale:0}, d:resource.infDecAmount{Dec:(*inf.Dec)(nil)}, s:"54833988Ki", Format:"BinarySI"}: may not have pre-allocated hugepages for multiple page sizes, status.allocatable.pods: Invalid value: resource.Quantity{i:resource.int64Amount{value:110, scal# If you change this file, run 'update-grub' afterwards to update
e:0}, d:resource.infDecAmount{Dec:(*inf.Dec)(nil)}, s:"110", Format:"DecimalSI"}: may not have pre-allocated hugepages for multiple page sizes, status.allocatable.cpu: Invalid value: resource.Quantity{i:resource.int64Amount{value:31800, scale:-3}, d:resource.infDecAmount{Dec:(*inf.Dec)(nil)}, s:"31800m", Format:"DecimalSI"}: may not have pre-allocated hugepages for multiple page sizes, status.allocatable.ephemeral-storage: Invalid value: resource.Quantity{i:resource.int64Amount{value:226240619760, scale:0}, d:resource.infDecAmount{Dec:(*inf.Dec)(nil)}, s:"226240619760", Format:"DecimalSI"}: may not have pre-allocated hugepages for multiple page sizes]
Mar 11 13:17:56 kubecord-a kubelet[8532]: E0311 13:17:56.693956    8532 kubelet.go:2236] node "kubecord-a" not found
```

原本會宣告 1G 的 Hugepages 是因為 DPDK，2M 的 Hugepages 是因為 VPP。  
而因為 `/etc/default/grub` 的 `GRUB_CMDLINE_LINUX` 沒把 2M 的移除，導致 k8s 起不來。  
而把 VPP 移除，2M 的設定也不需要，所以移除 2M 的設定，`update-grub` 後重開機，Kubernetes 就回來了！  
