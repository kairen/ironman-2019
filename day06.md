---
title: "淺談 Kubernetes 叢集元件更新"
date: 2019-9-21
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
在 Kubernetes 的快速發展下，每隔三個月左右，我們就會看到新版本的推出，以及舊版本進入 EOL(End of Life) 狀態，這時一但想要嘗試新功能與特性，或者是由於安全漏洞關析需要進行補丁時，就必須面對升級 Kubernetes 叢集元件問題。但是若在生產環境中，隨意升級 Kubernetes 叢集元件會不會發生什麼問題?

是的，若對於 Kubernetes 不熟悉的話，很容易掉入陷阱，比如說:已被棄用的 API、不相容的 Add-ons 版本、已棄用的元件參數、不正確的升級方式等等，這些都是有可能在升級後，導致 Kubernetes 叢集功能無法正常工作的原因。

<!--more-->

而今天就是要來跟大家分享一下，如何更安全地升級 Kubernetes 叢集，並且遇到問題時，該如何避免與解決。在開始前，我們先來了解一座 Kubernetes 叢集要更新時，需要關注哪些元件。

* **Kubernetes** 
    * **主節點元件**: kube-apiserver, kube-scheduler, kube-controller-manager 等等。
    * **節點**: kubelet。
    * **Add-ons**: kube-proxy、CoreDNS 等等。
* **容器 Runtime**: Docker、CRI-O 等等。
* **叢集資料儲存**: etcd。
* **叢集網路(CNI plugins)**: Ｃalico, Flannel 等等。
* **作業系統**: 系統軟體與 Kernel 版本。

### 更新叢集前須知
為了預防在更新 Kubernetes 叢集時發生問題，因此不需先注意幾件事情，以確保升級順利。

1. 在更新叢集時，請務必`備份 etcd 資料`。因為 etcd 儲存 Kubernetes 叢集的狀態，若發生不一致時，將會影響叢集運行。這邊可以透過以下工具來完成。
    * [etcd Operator](https://github.com/coreos/etcd-operator)
    * 透過 [etcdctl](https://github.com/coreos/etcd-operator) 的 snapshot + cron + restore 指令。

![R.I.P](https://i.imgur.com/zMJoesc.png)

2. 叢集更新時，務必以一個`次要版本(Minor)`為間隔進行。
    * Kubernetes 約每三個月會發布一個次要版本，其中每個次要版本號都會提供相容功能與 APIs 來轉移將被棄用的。
    * **Good**: My cluster is of v1.10, I want to upgrade to v1.11.
    * **Bad**: My cluster is of v1.10, I want to upgrade to v.1.13.
    * **Good**: My cluster is of v1.10, I upgrade to v1.11 and then upgrade to v1.12.

3. 閱讀 [Release Notes](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG.md) 來了解每個版本的變化。
    * 了解已知問題
    * 需要採取的措施
    * 被棄用與移除的功能
4. `善用工具`或`公有雲服務`來完成叢集更新過程。
    * Kubeadm, Kops, Kubespray, [Cluster API](https://github.com/kubernetes-sigs/cluster-api), ....
    * GKE, EKS, AKS, ....

> 公有雲通常提供了更新機制，這也是公有雲 Kubernetes 服務的優勢之一，因為自建的環境往往很容易因為更新發生問題。

5. 了解要更新的 Kubernetes `目標版本 API 變化`。
    * API 會隨版本演進而改變，如 [v1.16 要移除 extensions/v1beta1](https://kubernetes.io/blog/2019/07/18/api-deprecations-in-1-16/)。

![](https://i.imgur.com/Ho4qVuT.png)

6. 確保應用程式使用進階或穩定的 API 建置實例，如 Deployment。
    * 確保應用程式有多個實例(Pod)支撐。
    * 利用探針確保應用狀態，以攔阻流量的分發。
    * 使用 Pod 的 PreStop hook 來加強生命週期管理。
7. 更新 Node 以前，優先更新 Master。

> TODO: 補充更多細節

### 可預見問題與解法

1. etcd 中過舊的資料。
    * 不要刪除 API 版本(#52185)。
    * 使用 [Storage migration 系統](https://kubernetes.io/docs/reference/using-api/deprecation-policy)[1]。
    * 不要部署 EOL 版本 API[2]。
2. Clients 使用的版本已過舊(過時)
    * 應用與服務使用的 API 依然是相依於 extensions/v1beta1。
        * 需在叢集更新以前，優先將應用與服務的 API object 轉移到新版本
    * 開發的 Custom Controller 涉及過舊版本的 API 與函式庫。
        * 需要更新程式碼使用新版 API，並且將相依 Libraries 也更新至相容的新版本。
3. Policy breaks after upgrade(Webhook, RBAC)
    * 有新版本 batch/v2 與 test_batch/v1
        * 在叢集更新前，先更新 Policies 使用目前所有支援的版本。

![](https://i.imgur.com/Q8pmOSC.png)

> TODO: 補充更多細節

### 不可預見問題與解法

* 在升級前盡可能地在實際環境測試要升級的版本。
    * 使用舊版本的 [Sonobuoy](https://github.com/heptio/sonobuoy) 來測試新版本叢集。
* 確保組態檔案保持一致，並確保設定是否因為版本改變而出現錯誤。
* 檢視 Addons 是否因為新版本的特性而崩潰。
* 利用多階段升級方式來確保 API 的相容。

> TODO: 補充更多細節

## 結語
今天簡單的分享了升級 Kubernetes 叢集時，需要優先了解的知識，目的是希望幫助正在煩惱升級的人，能夠更有信心的執行。這邊在總結一下更新叢集前、更新叢集中與更新叢集後需要瞭解的事情。

* **更新叢集前**
    * 備份 etcd 資料!!備份 etcd 資料!!備份 etcd 資料!!
    * 閱讀新版本的 Release notes
    * 更新客戶端相關軟體與程式，以支援新舊版本的相容 API 與函式庫。
    * 更新相關的組態檔案，以支援新舊版本的相容 API 與函式庫。
* **更新叢集中**
    * 先更新主節點(Master node)，在更新工作節點(Node)。
    * 若主節點為 HA 架構，請確保所有節點都更新完，在使用新的 APIs 與功能。
    * 更新工作節點前，透過`kubectl drain <node>`將節點設定為維運模式。
    * 確認 Add-ons 支援新版本的 Kubernetes 元件、APIs 與函式庫等等後，再進行更新。
    * 網路插件同上。
* **更新叢集後**
    * 檢查 Kubernetes 叢集控制平面元件。
    * 檢查 Add-ons 與網路插件。
    * 確保所有節點處於 Ready 狀態。
    * 平衡因為升級而集中的 Pod。

當有了基本的叢集更新知識後，明天我們將實際實踐更新一座叢集的過程。

## Reference
- https://static.sched.com/hosted_files/kccncchina2018english/1c/Safely%20upgrading%20Kubernetes%20clusters.pdf
- https://static.sched.com/hosted_files/kccna18/8b/Highly%20Available%20Kubernetes%20Clusters%20-%20Best%20Practices%20-%20Kubecon%20NA%202018.pdf
- https://v1-15.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade-1-15/
- https://medium.com/@fairwinds/the-reactiveops-bestest-kubernetes-cluster-upgrade-f7a7589b21fb
- https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-upgrading-your-clusters-with-zero-downtime

[1]: https://github.com/kubernetes/community/pull/2524
[2]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/