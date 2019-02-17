+++
title = "VPP 初認識筆記"
date = 2019-01-22T16:48:35+08:00
tags = ["VPP", "userspace", "fd.io"]
categories = ["Network"]
draft = true
+++

## 前言

VPP 是 FD.io 的核心技術，是由 Cisco 捐贈的  
FD.io 全名是 `Fast data – Input/Output`，提供許多 high-throughput, low-latency and resource-efficient IO services 的軟體。  
Linux Foundation 有 host FD.io 這個專案。  

> OS: 中文的資訊好少，所以邊看文件邊做筆記了。如有錯誤資訊，還請指教。  

## VPP 簡介

- 全名：Vector Packet Processing
- 是一個軟體
- 是一個 Fast, Scalable and Deterministic, Packet Processing stack
- 可以跑在任何商用的 CPUs
- 是一個可擴展和模塊化設計 (Extensible and Modular Design) 框架
- 優點：high performance, proven technology, its modularity and flexibility, integrations and rich feature set

## FD.io VPP 功能與特徵

### 封包處理

- L2 到 L4 Network Stack
    - Routes & bridge entries 的快速查表(tables)
    - Arbitrary n-tuple classifiers
    - Control Plane, Traffic Management and Overlays
- 支援 Linux & FreeBSD
- Wide network and cryptograhic hardware support with DPDK.
- 支援 Container 和虛擬化技術
    - Vhost and Virtio
    - Native container interfaces; MemIF
- 通用 Data Plane
    - Discrete appliances; such as Routers and Switches.
    - Cloud Infrastructure & Virtual Network Functions
    - Cloud Native Infrastructure

> https://fdio-vpp.readthedocs.io/en/latest/overview/features/index.html#features
...

### Fast, Scalable and Deterministic

### Developer Friendly

### Extensible and Modular Design


## Ref

- https://fd.io/about/
- https://www.linuxfoundation.org/projects/
- https://fdio-vpp.readthedocs.io/en/latest/
- https://wiki.fd.io/view/Main_Page
