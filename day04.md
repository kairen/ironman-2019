---
title: "淺談 Kubernetes 高可靠架構"
date: 2019-9-19
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
近幾年來容器的興起，重塑了我們對於開發、部署與維運軟體的方式。容器允許我們透過打包應用程式成容器映像檔，並透過容器引擎部署到一組虛擬或實體的機器上執行。也正因為這樣過程與需求，產生了所謂的容器編排系統(Container Orchestration)，以自動化、基於容器應用程式的方式部署、管理與擴展。其中 Kubernetes 就是近年來的一套容器編排系統標準，它能允許大規模部署與管理基於容器的應用程式，而 Kubernetes 的特性還能實現分散式的應用程式，以帶來更高的可靠性與穩定性。但是，當 Kubernetes 重要元件或是其主節點(Master Node)發生故障時，那 Kubernetes 會發生什麼事呢?又該如何確保 Kubernetes 本身保持正常運行呢?

<!--more-->

前幾天分享部署工具章節，可以了解在生產環境中，部署一座高可靠(Highly Available，HA)架構的 Kubernetes 是非常重要的一件事。因為當服務在一座非高可靠的 Kubernetes 叢集上執行時，發生了節點故障的話，將會造成正在執行的服務受影響，嚴重甚至中斷整個服務的運行，除此之外還可能發生 Kubernetes 叢集狀態不一致、叢集功能全失效等等問題，既而造成維運人員負擔，以及造成公司損失。

也正因此，今天想針對 Kubernetes 高可靠架構進行探討，希望帶大家了解 Kubernetes 高可靠架構如何規劃與實現。

## 高可靠架構
在開始談 Kubernetes 高可靠架構時，我們先來複習 Kubernetes 的節點角色，以及其元件:

* **主節點(Master node)**: 該節點主要運行 Kubernetes 的控制平面元件，如: kube-apiserver、kube-controller-manager、kube-scheduler、kubelet 與容器 runtime 等等元件，目的是用於維護整個 Kubernetes 的運作，負責接收 API 請求、容器排程、執行容器等等事情。另外根據架構不同還有可能運行 etcd 作為整個叢集狀態儲存用。

> * [etcd](https://github.com/etcd-io/etcd) 是一套分散式 key-value 儲存系統，在 Kubernetes 中，它被用於儲存叢集狀態，如: API resources。
> * 在目前常見架構中，主節點也被視為工作節點，但預設會透過一些機制來確保容器不會被排程到這些節點。

![Kubernetes Master](https://i.imgur.com/yUXpNWu.png)

* **節點(Node)**: 又稱 Worker node 或 Minion node，該節點主要運行 kubelet、容器 runtime 等等。主要被用於執行應用程式容器，會定期與主節點回報目前節點資訊。

![Kubernetes Node](https://i.imgur.com/xGcL34D.png)

而 Kubernetes 的高可靠旨在使用一種沒有單點故障(SPOF, Single Point of Failure)的方式，設定與建構 Kubernetes 元件與支援的元件(etcd)。當使用單一主節點叢集時，很容易因為節點或元件發生錯誤，導致叢集故障; 而多主節點叢集則可以利用每個主節點元件來存取相同的工作節點，因此能更提升叢集的穩定性與可靠性。

在單個主叢集中，kube-apiserver 與 kube-controller-manager 等等重要元件都僅在單一節點上，如果故障的話，則無法建立或執行 Kubernetes 的功能。但在 Kubernetes HA 架構中，這些重要元件會在多個節點各執行一組(通常最小為 3 個主節點)，因此有主節點故障時，就會有正常的主節點來確保叢集的運作。其架構如下圖所示:

![Kubernetes HA](https://i.imgur.com/9hVtWy9.png)
(圖片擷取自：[kubernetes.io](https://kubernetes.io/docs))

而在 Kubernetes HA 的架構實現中，etcd 會根據需求有不同的拓樸架構，分別為:

* **Stacked etcd topology**: 該拓樸是將 etcd 連同 Kubernetes 控制平面放同一個節點上。這時 etcd 可能會透過 kubelet 以 Static Pod 方式來管理，或者以作業系統的背景行程管理機制進行(如 Linux systemd)。雖然這種架構只需要最少三台就能達到 HA，但由於放一起，所以故障域不會分離。

![](https://i.imgur.com/g0Y8Ffn.png)
(圖片擷取自：[kubernetes.io](https://kubernetes.io/docs))

* **External etcd topology**: 該拓樸是讓 kube-apiserver 存取外部的 etcd 叢集，這種架構將叢集資料儲存系統與叢集功能系統做分離，因此可靠性比 Stacked etcd 架構來的好些，不會因為一個節點故障，而同時影響到 Kubernetes 控制平面與 etcd。相反的，這種架構需要更多的機器來支撐。

![](https://i.imgur.com/m3ex3yv.png)
(圖片擷取自：[kubernetes.io](https://kubernetes.io/docs))

> TODO: 補充更多細節

## 結語
今天主要再次複習 Kubernetes HA 架構是如何實現的，因為在 On-Premise(Custom) Kubernetes 中，不像公有雲服務會幫你管理主節點的元件，因此適當地了解對於部署 On-Premise(Custom) Kubernetes 有很多幫助，也可以再發生問題時，更快的知道問題點在哪裡。

而除了今天提到的事情外，究竟建立 HA 架構還有什麼好處呢?

* 透過負載平衡器來分散 Kubernetes API server 的負載。
* Kubernetes 狀態資料的故障轉移(etcd)。
* 在 Multi-zones 中執行，確保跨不同故障域(Failure domains)。
* 實現安全地更新 Kubernetes 叢集元件。

> 更新 Kubernetes 叢集元件部分，在之後文章我也會說明如何進行，並需要注意哪些事情，以確保更新版本時不會出太大錯誤。

最後明天我們將實際利用工具來建立一套 Kubernetes HA 架構的環境，並在過程中說明如何做最佳實踐，一但完成後，也將嘗試驗證功能。

## Reference
- https://www.kubeclusters.com/docs/How-to-Deploy-a-Highly-Available-kubernetes-Cluster-with-Kubeadm-on-CentOS7
- https://platform9.com/blog/create-highly-available-kubernetes-cluster/
- https://medium.com/@bambash/ha-kubernetes-cluster-via-kubeadm-b2133360b198
- https://techbeacon.com/enterprise-it/6-best-practices-highly-available-kubernetes-clusters
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/
- https://kubernetes.io/docs/tasks/administer-cluster/highly-available-master/
- https://kccna18.sched.com/event/GrWQ