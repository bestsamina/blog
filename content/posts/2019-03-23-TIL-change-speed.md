+++
title = "[TIL] 在 Ubuntu 16.04 改變網卡速度"
date = 2019-03-23T14:33:14+08:00
tags = ["Network", "todayilearned", "til"]
categories = ["Todayilearned"]
+++

網卡限速設定參考：https://www.shellhacks.com/change-speed-duplex-ethernet-card-linux/  
不過不曉得為何在 Ubuntu 16.04 中 ethtool 的指令不能 work。  
以下是記錄可以 work 的方法。  

1. 原本的 Network Interface 狀態  

ens11f0 沒有接上網路線，ens11f3 有接網路線 所以 Speed 和 Duplex 有接上網路線的就有顯示。 不過 ens11f3 這樣還是不能和 100Mb/s 的互相 ping 成功，Link detected 也顯示 no，interface 也是 DOWN 的。  

```sh
$ sudo ethtool ens11f0
Settings for ens11f0:
    Supported ports: [ ]
    Supported link modes:   10000baseT/Full
    Supported pause frame use: Symmetric
    Supports auto-negotiation: Yes
    Advertised link modes:  10000baseT/Full
    Advertised pause frame use: No
    Advertised auto-negotiation: Yes
    Speed: Unknown!
    Duplex: Unknown! (255)
    Port: Other
    PHYAD: 0
    Transceiver: internal
    Auto-negotiation: off
    Supports Wake-on: g
    Wake-on: g
    Current message level: 0x0000000f (15)
                   drv probe link timer
    Link detected: no

$ sudo ethtool ens11f3
Settings for ens11f3:
    Supported ports: [ FIBRE ]
    Supported link modes:   Not reported
    Supported pause frame use: Symmetric
    Supports auto-negotiation: No
    Advertised link modes:  Not reported
    Advertised pause frame use: No
    Advertised auto-negotiation: No
    Speed: 10000Mb/s
    Duplex: Full
    Port: Direct Attach Copper
    PHYAD: 0
    Transceiver: internal
    Auto-negotiation: off
    Supports Wake-on: g
    Wake-on: g
    Current message level: 0x0000000f (15)
                   drv probe link timer
    Link detected: no

$ ip a
2: ens11f0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop portid 8cea1b30da3f state DOWN group default qlen 1000
    link/ether 8c:ea:1b:30:da:3f brd ff:ff:ff:ff:ff:ff
3: ens11f1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop portid 8cea1b30da40 state DOWN group default qlen 1000
    link/ether 8c:ea:1b:30:da:40 brd ff:ff:ff:ff:ff:ff
4: ens11f2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop portid 8cea1b30da41 state DOWN group default qlen 1000
    link/ether 8c:ea:1b:30:da:41 brd ff:ff:ff:ff:ff:ff
4: ens11f3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop portid 8cea1b30da42 state DOWN group default qlen 1000
    link/ether 8c:ea:1b:30:da:42 brd ff:ff:ff:ff:ff:ff

```

2. 更改 ens11f3 的網卡速度

更改 `/etc/network/interfaces` 檔案，加上下面這幾行  

```
auto ens11f3
iface ens11f3 inet static
address 192.188.2.100
netmask 255.255.255.0
link-speed 100
link-duplex full
ethernet-autoneg off
```

```sh
$ sudo /etc/init.d/networking restart
```

3. 檢查

Link detected 有看到 yes, 同時 IP 設定好後也可以 ping 通了  

```sh
$ sudo ethtool ens11f3
Settings for ens11f3:
    Supported ports: [ FIBRE ]
    Supported link modes:   Not reported
    Supported pause frame use: Symmetric
    Supports auto-negotiation: No
    Advertised link modes:  Not reported
    Advertised pause frame use: No
    Advertised auto-negotiation: No
    Speed: 10000Mb/s
    Duplex: Full
    Port: Direct Attach Copper
    PHYAD: 0
    Transceiver: internal
    Auto-negotiation: off
    Supports Wake-on: g
    Wake-on: g
    Current message level: 0x0000000f (15)
                   drv probe link timer
    Link detected: yes

$ ip a
2: ens11f0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop portid 8cea1b30da3f state DOWN group default qlen 1000
    link/ether 8c:ea:1b:30:da:3f brd ff:ff:ff:ff:ff:ff
3: ens11f1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop portid 8cea1b30da40 state DOWN group default qlen 1000
    link/ether 8c:ea:1b:30:da:40 brd ff:ff:ff:ff:ff:ff
4: ens11f2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop portid 8cea1b30da41 state DOWN group default qlen 1000
    link/ether 8c:ea:1b:30:da:41 brd ff:ff:ff:ff:ff:ff
5: ens11f3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq portid 8cea1b30da42 state UP group default qlen 1000
    link/ether 8c:ea:1b:30:da:42 brd ff:ff:ff:ff:ff:ff
    inet 192.188.2.100/24 brd 192.188.2.255 scope global ens11f3
       valid_lft forever preferred_lft forever
    inet6 fe80::8eea:1bff:fe30:da42/64 scope link
       valid_lft forever preferred_lft forever
```


