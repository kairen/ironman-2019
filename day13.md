---
title: "實作 Kubernetes 裸機 Load Balancer Part3"
date: 2019-9-28
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
在公有雲環境中，負載平衡器建立與外部 IP 位址分配都能由雲平台完成，且 Kubernetes 也能輕易地用 Cloud Provider 來進行整合。但在地端(或裸機)環境中，原生 Kubernetes 就無法達到這樣功能，必須額外開發系統才能達到目的。而慶幸的是前 Google 工程師也看到這樣問題，因此開發了 MetalLB 來協助非雲平台 Kubernetes 能實現網路負載平衡的提供。且 MetalLB 以 Kubernetes 原生方式，直接在 Kubernetes Service 描述`LoadBalancer` 類型來要求分配負載平衡器 IP 位址。雖然 MetalLB 確實帶來了好處，但它使用起來沒問題嗎?另外它究竟是怎麼實作的?會不會影響目前叢集網路環境呢?

基於這些問題，今天想透過深入了解 MetalLB 功能與實作原理，以確保發生問題時，能夠快速解決。

<!--more-->

## 架構
MetalLB 是基於標準[路由協定](https://en.wikipedia.org/wiki/Routing_protocol)實作的 Kubernetes 叢集負載平衡專案。該專案主要以兩個元件實現裸機負載平衡功能，分別為:

* **Controller**:是叢集內的 MetalLB 控制器，主要負責分配 IP 給 Kubernetes Service 資源。該元件會監聽 Kubernetes Service 資源的事件，一但叢集有 LoadBalancer 類型的 Service 被新增時，就依據內容從一個 IP 位址池分配負載平衡 IP 給 Service 使用。
* **Speaker**:利用網路協定(L2: ARP/NDP, L3: BGP)告知負載平衡 IP 的目的位址在何處，並且如何路由。Speaker 是一個被安裝在所有節點上的 Controller，而這些叢集上的 Speaker 只會有一個負責處理事情。

> TODO: 需補架構、流程圖跟程式細節說明。

在 MetalLB 中，實現了 L2(ARP/NDP) 與 L3(BGP) 的模式，使用者可以透過在 MetalLB 組態檔設定。而這種模式差異在哪邊呢?

### Layer 2
在 L2 模式下，MetalLB Speaker 會在叢集中，選出一個節點以標準地址發現協定(IPv4 用 [ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol)、IPv6 用 [NDP](https://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol))讓已分配的負載平衡 IP，透過 ARP/NDP 讓本地網路能夠得知目的位址。

> TODO: 需補架構、流程圖跟程式細節說明。

這種模式好處在於簡單，且不需要外部硬體或配置。但受限於 L2 網路協定。

### BGP
在這種模式下，叢集節點的 MetalLB Speaker 會與外部路由器建立 BGP 對等互連，並告訴路由器如何將流量轉發到Service IP，然後藉由 BGP 策略機制，在多個節點之間實現負載平衡，以及細粒度的流量控制。

> TODO: 需補架構、流程圖與程式細節說明。

這種模式適合生產環境，但需要更多的外部硬體與配置來達成。但要確保實作 BGP 的 CNI 不會衝突。

## 結語
從了解 MetalLB 原理後，可以更清楚知道一個 Kubernetes 裸機負載平衡該如何實現。但是除了 MetalLB 以外，還有其他方法可以實現嗎?當然有!大家可以參考我之前在社群分享的投影片 [How to impletement Kubernetes Bare metal Load Balancer](https://speakerdeck.com/kairen/how-to-impletement-kubernetes-bare-metal-load-balancer)。

> TODO: 需加入 IPVS 實現架構

## Reference
- https://metallb.universe.tf/
- https://github.com/danderson/metallb
- https://blog.cybozu.io/entry/2019/03/25/093000
- https://www.objectif-libre.com/en/blog/2019/06/11/metallb/