---
title: "實作 Kubernetes 裸機 Load Balancer Part1"
date: 2019-9-26
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
在生產環境中，通常都是以 Ingress 方式來曝露 HTTP/HTTPS 存取服務，而前幾天分享如何透過 NGINX Ingress、ExternalDNS 與 CoreDNS 等，就是在自建 Kubernetes 上實現這樣功能，讓我們以 L7 網路協定功能來達到服務存取目的。但在實際應用中，還是有很多需要以 TCP/UDP 方式存取或連接服務，這樣該如何進行呢?相信有研究過 Kubernetes 朋友都知道 NGINX Ingress 也支援了 TCP/UDP 的反向代理，這表示 Ingress 也能支援 TCP/UDP。但另一個問題來了，如果叢集有兩個服務需要用到同一個 TCP/UDP Port 時，這該怎麼辦呢?這時 Ingress 就無法很好地達到該需求，那我們該怎麼做呢?事實上，Kubernetes Service 的 LoadBalancer 類型就能達到，但是`僅限於公有雲服務`上才能完成，在地端的 Kubernetes 存在著一些限制，使得無法滿足需求。而這次主題就是要針對該議題進行說明與嘗試實作。

<!--more-->

在開始實作前，我們先來聊聊目前 Kubernetes 支援的存取方式吧。

### Host Network
在生產環境中，有些容器應用程式會需要用到主機層面的網路資源，這時我們能夠在 Pod 設定`hostNetwork: true`來讓該 Pod 能夠使用主機的 Network namespace。而這種方法有幾個優點:

1. 沒有 NAT 轉換，因此效能佳。
2. 故障排除較簡單。
3. 設計單純。

雖然這在某些需求上使用很合理，，但用於 Pod 曝露給外部存取的話，就會面臨一些問題，比如說:由於 Pod 相依於主機，因此 Pod 使用的埠口(Port)會佔用主機、每次啟動會被排程道不同節點上，因此導致存取 IP 改變。如下圖建立了兩個應用程式，並分別使用 Host Network 功能。

![](https://i.imgur.com/dnRwziA.png)

從上圖中，可以看到使用 Host Network 的 Pod 能夠以主機 IP 位址來存取容器中的應用。假設有個 A 服務寫死要跟這模式的 Pod 溝通的 IP 位址，然後突然發生 HostA 節點故障狀況，這時 A 服務的 Pod 可能被搬移至其他節點上，因此存取 Pod 端點也跟著改變了，這時 A 服務就會發生連不到的狀況。

![](https://i.imgur.com/wisAlI0.png)

從上圖中，可以看到這種模式無法透過一個虛擬 IP 來提供外部存取，因此該節點故障時，就會影響到該 Pod。

### Host Port 
除了 Host network 以外，Kubernetes 也支援在 Pod 指定`hostPort`與`containerPort`欄位來建立 Port 映射功能，讓某個 Pod 的 Port 能夠以主機的`IP:Port`來存取服務。該模式與 Host network 好處差不多，但相對 Host network 來的安全一點，因為不需使用到主機的 Network namespace，但要注意此功能的支援，在目前版本中需要依賴 CNI 來完成。

另外使用 Host port 的容器只能被排程到沒有指定 Port 衝突的節點上，而這種方式也不適合提供給應用程式級的外部存取使用，比較適用於系統級背景服務。

> 圖片待補。

### Service
Kubernetes Service 是一種抽象資源，目的是希望把外部存取 Pod 機制進行分層處理，因為 Kubernetes Pod 監聽的 IP 位址不能作為外部存取端點來使用，這是由於 Pod 在叢集中，會隨著一些狀況而動態改變，或者重新建立，這使得原本的 Pod IP 位址也跟著變更，那如果有個服務寫死存取某個 IP 時，就會發生問題。因此 Kubernetes 利用 Service、EndPoint 與 Pod 三者關析來達到 Pod IP 改變時，也能透過統一的 IP 端點存取到 Pod。

預設情況下，Kubernetes Service 使用 ClusterIP 類型讓使用者可以在叢集內部存取 Pod。當將 Service 設定為 ClusterIP 時，Kubernetes 會自動從設定的網段分配一個名為 Cluster IP 的虛擬 IP 給 Service，接著利用 Linux 網路技術將指定的 Cluster IP 以 NAT 方式，轉發到 Pod 上來達成連接，因此大家會發現 Cluster IP 並不會真的存在實體網卡上。而這背後實現者就是 kube-proxy 這個元件，它利用 IPTables 或 IPVS 等技術，來實現 Service 運作機制，如下圖所示。

![](https://i.imgur.com/wCpjwrh.png)

而使用過 Kubernetes ClusterIP 類型的朋友會發現，該類型僅能在 Kubernetes 叢集內部存取，那如果要曝露服務給外部存取的話呢?這就要用到另兩個 Kubernetes Service 的類型了。下面我們也簡單的介紹一下。

#### NodePort
Service 的 NodePort 是一種很常用到的方式。由於 Kubernetes Service 預設都是使用 Cluster IP 來提供存取，但是 Cluster IP 只能在叢集內部存取用，當要給外部時，就無法達到需求，這時就能利用 NodePort 來達成。一但設定成 NodePort 時，叢集中裝有 kube-proxy 的節點，就會幫你把一個亂數 Port 綁定到主機上，接著利用 IPtables/IPVS 設定 NAT 轉發到關聯的 Pod 上，這樣只要存取任一個 Kubernetes 節點的 IP:Port 就能夠存取到 Pod。

> TODO: 內容待補。

![](https://i.imgur.com/5JQGIlj.png)

#### LoadBalancer
LoadBalancer 是提供給公有雲使用的類型，如果一個 Service 設定為這個類型時，就會利用叢集中的 Cloud Provider 元件去呼叫公有雲的服務 API 來建立負載平衡器，接著跟這個 Pod 進行綁定。但在地端部署時，該類型會實現類似 NodePort 機制，但可以額外設定 `externalIPs` 來直接存取指定的 Port。

> TODO: 內容待補。

![](https://i.imgur.com/FmNbr7N.png)

### Ingress
Ingress 雖然提供了許多負載平衡器的特性，如 HTTP/HTTPs 路由、SSL Termination、TCP/UDP 負載平衡等等。

> TODO: 內容待補。

![](https://i.imgur.com/aeW5PRF.png)

## 結語
今天了解了目前 Kubernetes 支援的外部存取服務方式。從中，我們知道在地端 Kubernetes 叢集中，並不支援網路負載平衡器的實現(Service 類型為 LoadBalancer)，因為 Kubernetes 內建的 LB 功能，幾乎都用於公有雲(GCP、AWS、Azure 等)服務上。但在自建(地端)叢集中，若沒有 IaaS 平台或環境能夠使用的話，就會在建立 LoadBalancer Service 時，一直處於`pending`狀態。

那麼我們該如何解決呢?下一篇將分享一些使用過的專案與做法。

> 今天由於一些原因沒有太多時間寫完整，這部分會在之後慢慢調整上來。

## Reference
- https://www.hwchiu.com/kubernetes-service-i.html
- https://www.hwchiu.com/kubernetes-service-ii.html
- https://www.hwchiu.com/kubernetes-service-iii.html
- https://medium.com/@tao_66792/how-does-the-kubernetes-networking-work-part-1-5e2da2696701
- https://medium.com/practo-engineering/networking-with-kubernetes-1-3db116ad3c98
- https://collabnix.com/3-node-kubernetes-cluster-on-bare-metal-system-in-5-minutes/
- https://medium.com/@maniankara/kubernetes-tcp-load-balancer-service-on-premise-non-cloud-f85c9fd8f43c
- https://jimmysong.io/posts/accessing-kubernetes-pods-from-outside-of-the-cluster/
- https://www.asykim.com/blog/deep-dive-into-kubernetes-external-traffic-policies
- https://thinkit.co.jp/article/13739