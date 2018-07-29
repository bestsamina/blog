+++
title = "[CORD] CORD 平台學習筆記"
tags = ["CORD", "trellis", "openCORD", "XOS"]
categories = ["CORD"]
date = 2018-07-26T15:51:05+08:00
+++

> From: https://wiki.opencord.org/display/CORD/Documentation  

分為三個 components:  

- Trellis:
    - CORD fabric 的網路架構
    - service composition 中的 overlay 虛擬化角色。
- CORD Monitoring Service
    - 是 CORD service ，專門蒐集與分析即時的 metrics。
- XOS
    - 專門 configure 和控制 CORD services.

## Trellis
> From: https://wiki.opencord.org/display/CORD/Trellis%3A+CORD+Network+Infrastructure  

- underlay leaf-spine fabric + overlay virtual networking + unified SDN control (underlay + overlay)
  
![](https://wiki.opencord.org/download/attachments/1278078/trellis1.jpg?version=1&modificationDate=1469902523365&api=v2)  
![](https://wiki.opencord.org/download/attachments/1278078/cord_trellis.jpg?version=1&modificationDate=1469902673504&api=v2)  

- The efficient of unified SDN control (underlay + overlay)
    - 為了 tenant 網絡的分佈式虛擬 routing
    - 多播流量傳輸的優化
    - 目前有兩個 ONOS cluster
        - onos-cord
            - 負責 overlay((virtual networking and service composition)) 和 access 的 infrastructure。
            - 分別 hosts VTN 和 vOLT 的 APP。
            - Multicast control: IGMP snooping
        - onos-fabric
            - 負責控制 fabric
            - 與upstream routers 的介接。
            - Multicast control: PIM-SSM

### Trellis Underlay Fabric

> Ref: https://wiki.opencord.org/display/CORD/Trellis+Underlay+Fabric  

- SDN based Leaf-Spine fabric
- 使用 data plane 的 headers: ARP, MAC, VLAN, IPv4, MPLS, VXLAN
- 不使用 distributed protocols
- fabric 特點：
    - L2 switching within a rack handled at leaf-switches (ToRs)
    - L3 forwarding across racks using ECMP hashing and MPLS segment routing.
    - vRouter integration for interfacing with upstream metro-router, providing reachability to publicly routable IP addresses.
    - VLAN cross-connect feature to switch QinQ packets between OLT I/O blades and vSG containers (R-CORD feature).
    - IPv4 multicast forwarding and pruning for IPTV streams (with vRouter integration) from upstream router to residential subscribers (R-CORD feature).
    - XOS integration via REST API calls to dynamically configure end-hosts and VLAN cross-connects at runtime.
    - Ability to use the fabric in single-rack or multi-rack configurations.

![](https://lh4.googleusercontent.com/xpxx5oAF_Xf6IjDmAcYFbSfyUV6Y0QeyTyMNsXCtWyZyKSmBQQzp31zBwvYVHm3IULB4XL9Qy31NzcjYLi2TzcM1IEhIQbf_zcKUGh2myHOxiZGvXFerclZQAIK7UqGm8aVY-CGK)  
Overlay and Underlay packet walk-through.  
![](https://lh6.googleusercontent.com/tCwnE8UL-xKyXKDGugWVLakYGam-9sa6fkRYN-IOtAfqXiMogqzOwmkRhfNELE-3TqBQ6GmqYgz6mZpQ5gHH9DsK4fpJdO2aYL8_NFF4JWnZyAqwXezom-iIyFZMhfDBdm3X17Zn)  

### Virtual Network Overlay
> Ref: https://wiki.opencord.org/display/CORD/Virtual+Network+Overlay  

![](https://lh6.googleusercontent.com/butp-czda5S_yVBYwUk-eLt49rrVhA_Q2H96sYraZjRBHX0p4TQQLqOG0K_1ZGq4_W9xSOaQDjmrCJnOxHLHeTuzcAJqUDvLkSvgkzvRK9VVEph0mRA3lZlTEUKb--j5qdNAMWhN)

- Services 有他們自己的 virtual networks (VNs) - 不論是 VM 或是 container 都是在同一個
- scale
- 每個 compute node 上的 VM 或是 container 都要連到 OVS 上
- 每個 Virtual Network (or service) 都要有自己的 Load-Balancer(LB) 分布在每個 OVS 網路中。LB 專門選擇 VM instance 的。
- Service composition walkthrough
- XOS 和 ONOS 的 VTN app 會互相協調以保持虛擬的 infrastructure 狀態是最新的。 ( VTN updates tables in a special purpose OVS pipeline to reflect the desired subscriber service flow. OVS forwarding pipeline 如下)
    ![](https://lh3.googleusercontent.com/-tfPjWtqHZkTH2tLadgs-frvETcuRtwsTts9GB8H5U5EXpz4hVbiO_wWsaJHdQK509azZ8GohNkWb8K3kT-_Q78e6rFWlyDienEUrFCPLsxyRgGxvbDgzMv1qTjF5dp_3Oc70ddl)

- Subscriber traffic is NATted by the vSG. 

#### [Virtual Tenant Network(VTN)](posts/2018-07-29-cord-vtn)

### Virtual Router (vRouter)

> Ref: https://wiki.opencord.org/pages/viewpage.action?pageId=1278093

- implemented as a network control application   
- running on ONOS
- Perform L3 unicast routing to/from CO; participate in dynamic routing protocols (current supports OSPF and BGP)
- Multicast signaling and forwarding (currently supports PIM-SSM)
- Quagga 支持多種路由協議，允許 vRouter支持這些協議
- Quagga 將配置為與上游路由器通信: OSPF & iBGP
- 使用 FIB Push Interface (FPI): enables it to push routes to an external entity

![](https://lh6.googleusercontent.com/JjekSLLwBTY0jhCh_HSazXyAt5IU4mfoZ8yJairOUDwJXq_t1AB8sJyLeUY5yEejoJLWt942pBFfss55W8fPouD2PgL4gVwSOlNryUThqA0CWR-d7SfFtjb6Nwob7GX8oTY9b04A)

#### Routing control traffic is handled by redirecting to towards the Quagga instance connected on the dataplane
![](https://lh4.googleusercontent.com/PRBfvmUayyZenmcfptmn7pGoG6zdJdtA17Vz0RiczIvqHwRW1EossWMoIlkBc4PR2kap8FXQKQzIXNsJI1sG0CjOpoGg06RTSexhv9Sc7rEtGQzpP03XPd1uofco25tBaPQj80Kg)

#### Multicast
#### Dataplane
Components that make up the vRouter ONOS app  
![](https://lh5.googleusercontent.com/5NWpDLDewAmgwV63TAaKaU9BKjmVUe9Fa_kmjgvkqFyaPjALjbgGBHBY4MbJNtTvblO5pZWuCEwKBzhYDgDx3mm9hEW62Ou7o1y8jUOJk-DlV8_NO5Tn3I71AYkkXRNMKy2iP3Fb)
* Forwarding Plane Manager (FPM): decode routes from the external Netlink protocol  
FIB installer: use ONOS BgpRouter application 的 SingleSwitchFibInstaller component  
- SingleSwitchFibInstaller component
    - install routes into a single switch
    - generate FlowObjectives and submit them to ONOS


## CORD Monitoring Service

XOS Monitoring service 支持：  

- real time network observability for SDN fixed
- mobile networks in a Telco Central Office

須提供：

- Analytics
- support multi-tenancy
- 檢測服務
- adjust the level of probing in the underlying devices
- aggregate probing information
- redirect data streams through a "probe VM" for deeper level of instrumentation that is not otherwise available from underlying devices

![](https://lh6.googleusercontent.com/0ISh2XYNBi7NoWLlzU6dY4zkXE8B16VcpcvofANS3OrIj5IFJ64U0OXGN-MvKcfD6VFKqeeASLkTKPNnoYMvb43sLIqOafgE3I8Y-qAam1GW8hiBnq7VTH8Ep0MXBnRVDpzALAjI)  
  
Using [OpenStack Ceilometer framework](http://docs.openstack.org/developer/ceilometer/) as Monitoring service.  
![](https://lh5.googleusercontent.com/kKcvHnPTpVIbPgY7QZpQpVdXadW181PYEDh28AfnVUK8_btWf3dfnO92nFnnoTe5Vfm9LnuQxZjYCgMcYXfzo-4OL3gqbRSIoYUGVeI5DnaVpDrBB3kB4-f6sDWpPxloCPGwHzEZ)  
Internal implementation based on OpenStack’s Ceilometer.  
![](https://lh6.googleusercontent.com/jOKxVnuSb8brtYM8JAO-8gGXX12-t1hVmnRE1KWlYSje8BhpsGfM1e4Y9Xfovbc_yJLxQN7i8Px7YejRxBCgl8suIXEKL3Tldcqu-UP1FQLtdQoE5syhfh_HLBM3WZKzp0U7CnwP)  
Monitoring service integrated into XOS.  
![](https://lh5.googleusercontent.com/hVsGneGPeYIhLbkrTuyldws94U1TL31p6PjBQ2WNBpCHyMHn8ntS9PNHkQAJbH1jqZmcWWx7nGS32VTwPF2u9-_n7PfvSwD0kqu87luu3gXPQlj5K8jn5rsccNo6To6hF8mJdOAb)  


### CORD 定義

> https://wiki.opencord.org/display/CORD/Defining+CORD  

現今所謂的 R-CORD, M-CORD 都是 CORD 的其中一個 Solution 。  
每個 solution 都是為了解決不同的 access technology  
![](https://lh6.googleusercontent.com/L83lsymO848rFWfmU9sxNARcCzhWD0lhqAzdbHaoMkEKjtLhHR7K-5abo4gtmld_i9X6fvMFm0erydmyRGrUiQUNaPz0fP0-s5Pa07aL1JWLTrf913EELMdwuRhR5vCh-K69i6td)  
**Solutions are defined by a Service Graph**  

- a set of Services and Dependencies between Services
- a Service consists of a Controller and one or more Slices
- a Slice is a logical resource container that includes a set of Instances and a set of Networks

個別的 components 可以單獨的被使用。可以確保與 CORD Vision 一致，但也可以以不同 integrated 的方式代替 CORD Architectures。  
**Today’s CORD Reference Platform includes the following software components: XOS, ONOS, OpenStack, Docker, VTN (an ONOS app that implements overlay virtual networks), and Fabric (an ONOS app that manages the switching fabric).**  

### [XOS](posts/2018-07-29-cord-xos)
