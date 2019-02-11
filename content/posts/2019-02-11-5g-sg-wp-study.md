+++
title = "5G-PPP Software Network White Paper 畫重點筆記"
date = 2019-02-11T14:33:14+08:00
tags = ["5G"]
categories = ["5G"]
+++

From Webscale to Telco, the Cloud Native Journey  

> https://5g-ppp.eu/white-papers/

這邊是一篇"畫重點"的筆記，關於 5G & Cloud Native ，From 5G-PPP(5G Infrastructure Public Private Partnership) 的白皮書。  

## 1 Introduction

- Software in 5G-PPP so far
    -  from “boxes” to “functions”, and from “protocols” to “APIs”
- Cloud impact in general
- Cloud impact in telecom ecosystem in particular
    - covering Service-Oriented Architecture (SOA), Microservices Architecture (MSA) and Service-Based Architecture (SBA), the latter being adopted in next generation CORE [15]
    -  telcograde enhancements that should be added in frameworks like Kubernetes, monitoring, stateless design, etc...
    - What are the specific requirements between Webscale and telco players?
    - What gaps do we have in container management eco-system to achieve telco-grade performance?
    - Which segments are critical from latency perspective?
    - What features are required to support micro-services architecture in all the segments?
    - Which factors are difficult to implement in telco (from https://12factor.net)?

- 章節介紹如下：
    - 2 介紹了基於服務的體系結構，12個因素以及網絡，虛擬化，編排等所需的技術。
    - 3 列出了電信公司的要求。
    - 4 描述了電信級的增強功能。
    - 5 簡要概述了技術推動因素，開源和標準。
    - 6 結論。

## 2 Cloud Native: a Webscale View

SBA(Service-Based Architecture)  
![](https://i.imgur.com/P9Zq2ki.png)  

3 ways to implement:  
either as a network element on a dedicated hardware  
as a software instance running on a dedicated hardware  
as a virtualized / cloud-native function instantiated on an appropriate platform  
  
利用雲軟件技術，3GPP NF架構可實現更高的靈活性，可編程性，自動化以及顯著的成本/能耗降低。  

### 2.1 Microservice Architecture

提供超可靠，低延遲的通信和高速  
為此，需要在架構中實現高度靈活性，可歸納如下：  

 - 支持對網絡的不同要求，如網絡容量，數據速率，傳輸延遲，信息安全;
 - 支持向運營商的服務和第三方應用程序公開網絡功能
 - 支持輕鬆部署和維護。 每個功能都應該能夠根據新的要求進行升級，並根據系統能力進行橫向擴展，而不會影響其他功能。

SBA benefits to 5G:  

- Easy update of Network:
    - Finer granularity allows individual services to be upgraded with minimal impact to other services.
    - Facilitated continuous integration reduces the time-to-market for installing bug fixes and rolling out new network features and operator applications.
- Extensibility:
    - Light-weighted service-based interfaces are needed to communicate across services.
- Modularity & Reusability:
    - The network is composed of modularized services reflecting the network capabilities and provides support to key 5G features such as network slicing.
    - A service can be easily invoked by other services (with appropriate authorization), enabling each service to be reused as much as possible.
- Openness
    - Together with some management and control functions (i.e. authentication, authorization, accounting), the information about a 5G network can be easily exposed to external users such as 3rd parties through a specific service without complicated protocol conversion
### 2.2 Twelve Factors

https://12factor.net/zh_cn/

### 2.3 Technology for Cloud Native

Cloud Native Trail Map  
https://github.com/cncf/landscape/blob/master/README.md#trail-map  

## 3 Telco Requirements

- 5G Service Enablers
    - eMBB – Enhanced Mobile Broadband
    - mMTC – Massive Machine Type Communications
    - URLLC –Ultra-Reliable and Low Latency Communications

IMT-2020 5G Triangle   
![](https://i.imgur.com/vD2TnW1.png)


- 3.1 Service Requirement: 99.999%
- 3.2 Application Type: like multi-access edge computing (MEC)
- 3.3 Deployment Environment 
- 3.4 Operability: Need monitoring system
- 3.5 Business Environment 
- 3.6 Human Factors

## 4 Toward Telco Grade Webscale Frameworks 
### 4.1 Need for Telco Grade features

-  Open and modular software
-  Intelligence in software
-  Scalable and efficient

- Typically, telco key performance indicators taken into account include latency and bandwidth, availability and security.
    - For instance, since each microservice contributes in the overall latency, it is challenging to predict the latency of a specific service, especially in cases of large and distributed systems.
    - container management technologies (Kubernetes, Mesos, Swarm) are using an overlay network (Layer 4) which is not optimized for low latency requirements, e.g. in the cloud RAN
    - network availability, as 99.999% availability is commonly set by the operators. Currently, cloud-native applications are not built to handle bandwidth, latency and network availability requirements.
    - container management include the lack of a dynamic carrier-grade orchestration, the management of containers across different sites and data centers and the lack of centralized controllers for networking aspects with respect to provisioning, resiliency, management/coordination, automatic network configuration and visibility of traffic from container-to-container across networks -whether physical or virtual.

- 為了滿足上述要求，必須滿足以下條件：
    - Automation of operations, upgrade, backup and restore as well as restart capabilities
    - Monitoring, performance counter, alarms and logging/trace support
    - Security and tenant isolation.


### 4.2 Twelve Factors in Telco Domain 

- Build/Release/Run (12.V) is not a part of ETSI standards
- dependency declaration and management (12.II) should be of particular concern
- the codebase (12.I) and Dev/Prod parity (12.X) which should accommodate various deployments strategies over a potentially multi-cluster multi-vendor environment.
- factors 12.VIII (Concurrency) and 12.VI (Processes) 12.IX (Disposability) and 12.XI (Logs) pose the greatest challenges ahead for the participating 5G projects. 
- Logs (12.XI) speed, heterogeneity and variability of origin will force a readjustment of current telco methodologies. 
- Disposability (12.IX), i.e. small start-up times and graceful termination, is major concern for five nines availability systems, especially for emergency services and in relation to QoS constraints.
-  the factor on administrative processes (12.XII) seems a key aspect, and especially critical since it has less to do with technology evolution.

### 4.3 Telco-Grade Features

Several features must be integrated to the aforementioned frameworks:  

1. Multi-network interfaces  
2. Service function chaining  
3. Data plane acceleration  
    -  low packet processing time to reach very low latency  
    -  predictability and stability of packet processing delay to minimize jitter  
    -  guaranteed and prioritized bandwidth for each Telco workload to prevent network contention  
    -  low CPU consumption for network tasks to allow low energy usage  
    -  native network performance for Telco workloads  
4. Enhanced platform awareness  
    -  CPU pinning to avoid unpredictable latency and host CPU overcommit by dedicating CPUs to the Telco containers  
    -  Numa awareness to improve the utilization of compute resources for Telco containers that need to avoid cross NUMA node memory access  
    - huge pages to accelerate the memory management by using larger page sizes.  
5. Environment aware scheduling  

## 5 Technology Enablers – Open source and Standards
### 5.1 Standards 

-  ETSI NFV
    https://www.etsi.org/technologies/nfv
-  TM Forum
    https://www.tmforum.org/virtualization/
-  Open Networking Foundation
    Open Networking Foundation (ONF) 
    Central Office Re-architected as a Datacenter (CORD)

### 5.2 Open Source Software 

可以參考 CNCF 的軟體來實作  
MANO frameworks such as Linux Foundation’s ONAP [41], ONF XOS [42], Cloudify [43] and ETSI OSM [44] are starting to evolve towards this approach for microservices.  

## 6 Conclusion

- The VNFs are now broken into pieces, support continuous integration and continuous development (CI/CD) and interact using service meshes.
- Compare the three architectures inspired from the domain driven design
    - Service Oriented Architecture
    - Service Based Architecture 
    - Microservice Architecture
- the new VNFs, microservice, are designed, a set of technology used to deploy, orchestrate, network and monitor them.
- Telco grade features
- The telco requirements that are needed to be supported in community developed open source.


