---
title: "淺談 Kubernetes 自定義資源(Custom Resource)與自定義控制器(Custom Controller)"
date: 2019-10-04
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
使用 Kubernetes 時，大家都能感受到其容器編配能力，當有一個容器發生異常時，Kubernetes 會透過自身機制幫你把容器遷移或重新啟動，或者能利用副本機制讓容器同時存在於叢集的不同節點上，甚至提供滾動升級(Rolling Update)容器機制。這些酷炫的功能，大家肯定都知道如何去使用，因為 Kubernetes 透過一些方式，將複雜的功能進行了抽象化與封裝，因此使用者只需要了解如何操作 API 物件，就能完成需要的功能，比如:Deployment 修改參數就會進行滾動升級。然而這些『抽象化』與『封裝』的過程究竟是如何實現呢?今天文章就是要針對這個部分進行探討。

<!--more-->

Kubernetes 是個非常容易擴展的系統。Kubernetes 提供了多種方法讓我們能夠自定義 API，或擴展功能，比如:

* **Cloud providers**: 提供自定義雲平台整合的控制器。比如說跟 IaaS 進行整合，當建立一個 LoadBalancer Service 時，自動呼叫 IaaS 負載平衡 API 進行建立。

> 過去 Cloud providers 屬於 kube-controller-manager 的部分控制器，現在已從核心程式碼移出。

* **Admission control webhooks**: 提供 kube-apiserver 存取擴展的 Webhook API。
* **kubelet plugins**: 提供容器 Runtime(CRI)、網路(CNI)、儲存(CSI)與裝置(Device Plugins)等等介面。
* **kubectl plugins**: 提供擴展 kubectl。如 [krew](https://github.com/kubernetes-sigs/krew)。
* **Custom resources 與 Custom controllers**: 提供自定義 API 資源(物件)，以及執行這些客製化 API 資源(物件)的邏輯程式。
* **Custom API servers**: 提供透過 [apiserver](https://github.com/kubernetes/apiserver) 函式庫開發用於 API Aggregation 的 API servers。
* **Custom schedulers**: 提供能實現自定義的排程演算法與機制，並在 Pod 指定使用。
* **Authentication webhooks**: 提供擴展 Kubernetes 身份認證機制，可以與外部系統整合。如: LDAP、OCID。

![](https://i.imgur.com/iz2YEOb.png)

而今天我們重點就是要放在探討`自定義資源(Custom Resource)`與`自定義控制器(Custom Controller)`上。

### 自定義資源(Custom Resource)
在 Kubernetes API 中，一個端點(Endpoint)就是一個資源，這個資源是被用於儲存某個類型的 API 物件的集合。比如說 Pod 有 /api/v1/pods API 端點。 

![Kubernetes API Resources](https://i.imgur.com/IdC8aKc.png)

一個 API 端點的組成如下所示:

* **API Group**: 是邏輯上相關的種類集合，如 Job 與 CronJob 都屬於批次處理功能相關。
* **Version**: 每個 API Group 存在多個版本，這些版本區分不同穩定度層級，一般功能會從 v1alpha1 升級到 v1beta1，然後在 v1 成為穩定版本。
* **Resource**: 資源是透過 HTTP 發送與檢索的 API 物件實體，其以 JSON 來表示。可以是單一或者多個資源。

其中每個 API 都可能存在著不同版本，其意味著不同層級穩定度與支援度:

* **Alpha Level**: 在預設下是大多情況禁止使用狀態，這些功能有可能隨時在下一版本被遺棄，因此只適用於測試用，如: v1alpha1。
* **Beta Level**: 在這級別一般預設會啟用，這表示該功能已經過很好的測試項目，但是物件內容可能會在後續版本或穩定版本發生變化。如: v1beta2。
* **Stable Level**: 在這級別表示該功能已經穩定，會很長的時間一直存在。如: v1。

![](https://i.imgur.com/znt3I3m.png)

而自定義資源就是預設不存在於 Kubernetes 原生的額外 API 資源，這包含了從當前叢集擴展新的資源物件(如:原本沒有 DaemonJob，我透過一些機制新增了)，以及其他系統元件本身使用的(如: KubeadmConfig)。目前 Kubernetes 提供了兩種方式來新增自定義資源:

* [CRD(CustomResourceDefinitions)](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/)
* [API Aggregation](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)

> TODO: 補充更多細節

### 自定義控制器(Controller)
當利用 kubectl 建立一個 Pod 時，客戶端會透過 kubectl 與 kube-apiserver 進行溝通呼叫 Pod APIs，這時 API 物件經過驗證後，被成功建立到 Kubernetes 上，最後儲存到 etcd 中，然後過不久後，就會發現這個 Pod 在叢集中的某個節點上被執行了。到這邊一定會疑惑中間的過程，是怎麼判定建立到哪個節點上的，又是怎麼在該節點建立的呢?實際上，這需要由 Kubernetes 的`kube-scheduler`與`kubelet`元件完成的，其流程如下圖所示。

![Pod workflow](https://i.imgur.com/kOPShpV.png)

1. 使用者透過客戶端工具與 kube-apiserver 以 REST API 方式進行溝通建立 Pod 物件，然後 kube-apiserver 進行各種驗證通過後，將其寫入 etcd 中。
2. 這時 kube-scheduler 會透過監聽 kube-apiserver 的 Pod 物件變化事件，獲取到 Pod 物件的內容，而當觀察到該 Pod 的`.spec.nodeName`欄位沒有被分配節點名稱時，kube-scheduler就會透過`過濾(Filter)`與`排名(Rank)`演算法來計算所有節點的權重，並從中找出一個最佳的節點，接著在 Pod 的`.spec.nodeName`更新被選取的名稱，然後該狀態會被儲存到 etcd 中。
3. 這時 kubelet 也一直監聽著 kube-apiserver 的 Pod 物件變化事件，當發現有一個 Pod 的`.spec.nodeName`欄位是這個節點時，kubelet 就會呼叫容器 Runtime 來啟動這個 Pod 所定義的相關功能，如:掛載儲存、透過 system call 寫入環境變數等等。
4. 一方面 kubelet 會監控容器 Runtime 執行的 Pod 狀態。並隨著情況的變化，同步將內容更新到 Pod API 物件上，以讓使用者能夠了解當前狀態。

從這點了解 kube-scheduler 與 kubelet，才是實際上負責執行 Pod 邏輯的角色之一，而這些`邏輯`就是所謂的`控制器(Controller)`。因此儘管 Kubernetes 原生提供了許多的 API 資源可以使用(如下圖)，如果當前 Kubernetes 叢集並沒有啟動或安裝相關的控制器，以執行實際的邏輯的話，這些 API 資源就形同空殼般存在於叢集中。這邊再舉幾個例子，比如:實現 Service 功能的就是 kube-proxy 與 kube-controller-manager 的 [Endpoint 控制器](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller/endpoint)(或 [Endpoint Slice 控制器](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller/endpointslice))，這兩者分別監聽 Service 設定 NAT rules 與同步綁定 Service 與 Pod 的 IP。

![Kubernetes API Resources](https://i.imgur.com/y5KxOlT.png)

那麼自定義控制器又跟自定義資源有什麼關析呢?就如同上面提到範例的 API 資源一樣，自定義資源本身只提供儲存與檢索結構化內容，因此當擴展時，並不會有實際功能，而這時就需要結合自定義控制器來完成功能的邏輯事情，並持續同步更新自定義資源。一般來說自定義控制器會有一個 Control Loop 邏輯，會持續監聽自定義資源在 API server 的事件變化(Create、Update 與 Delete)，一但收到變化後，取出 API 物件內容，並執行預期結果。

![The Controller Loop](https://i.imgur.com/vTzezjA.png)
(圖片擷取自：[Programming Kubernetes](https://github.com/programming-kubernetes))

> TODO: 補充更多細節

## 結語
在 Kubernetes 生態中，幾乎所有 API 物件功能都是以這樣形式來完成，讓使用者以宣告式 API(Declarative API)方式先定義該物件`預期執行的需求`，最後再由控制器想辦法`執行到預期的結果`。而在過去，這種模式並沒有盛行於開發者上，是直到 v1.7 版本 CRD(CustomResourceDefinitions) 的出現(當然 Kubernetes 成功也是原因)，才出現越來越多基於此概念的各種控制器出現，甚至出現了新的名詞『Operator』。

自定義控制器除了能夠讓開發人員擴展與添加新功能以外，事實上也能替換現有的功能來優化(如利用 kbue-router 取代 kube-proxy)。當然也能用於執行一些自動化管理任務。接下來我將用一系列文章說明如何實作自定義控制器，並了解一些技巧。

## Reference
- https://github.com/kubernetes/kubernetes/tree/master/pkg/controller
- https://kubernetes.io/docs/concepts/extend-kubernetes/
- https://speakerdeck.com/thockin/kubernetes-what-is-reconciliation
- https://medium.com/speechmatics/how-to-write-kubernetes-custom-controllers-in-go-8014c4a04235
- https://itnext.io/how-to-create-a-kubernetes-custom-controller-using-client-go-f36a7a7536cc
- https://github.com/kubeflow/tf-operator/issues/300
- https://admiralty.io/blog/kubernetes-custom-resource-controller-and-operator-development-tools/
- https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
- https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/
- https://zhuanlan.zhihu.com/p/59660536