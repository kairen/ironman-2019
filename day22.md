---
title: "開發自定義控制器前，需要先了解的東西 Part3"
date: 2019-10-07
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
前面文章中，了解到 Kubernetes 的架構一直以擴展性與靈活性為主，從 [Extension Points](https://kubernetes.io/docs/concepts/extend-kubernetes/extend-cluster/#extension-points) 可以看到上至 API 層，下至基礎建設層都有各種擴展的介面、標準與 API，其中 API 的擴展，我們在介紹自定義資源(Custom Resource)時，有提及能透過`CustomResourceDefinition(CRD)`與`API Aggregation`來達成，一方面在開發自定義控制器時，很多情況下，會透過增加新的 API 資源來讓控制器實現功能，因此我們必須先瞭解這兩者擴展 API 的作法與選擇。

<!--more-->

在 Kubernetes API 擴展中，不管是使用 CRD 或 API Aggregation，旨都是希望在不修改 Kubernetes 核心程式碼的情況下，讓新的 API 能夠被註冊或新增到 Kubernetes 叢集中。但兩者在用意上有一點小差異:

* **API Aggregation**: 用於將第三方服務資源註冊到 Kubernetes API 中，以統一透過 Kubernetes API 來存取外部服務。
* **CRD**: 在當前叢集中新增 API 資源，並沿用 Kubernetes 原有的 API(如 Pod) 操作方式來管理這些新 API 資源。

從描述中，可以看到差別在於『把既有註冊』跟『直接增加新的』。而這兩者又是如何運作呢?

### CRD(CustomResourceDefinitions)
CRD 是 Kubernetes 在 v1.7 版本新增的 API 資源，被用於新增自定義 API 資源上，一但 CRD 物件建立時，Kubernetes API Server 就會幫你處理自定義資源的 API 請求、狀態儲存等。而這過程中，完全不需要撰寫任何程式碼。

> 事實上，在 CRD 之前還有個 ThirdPartyResources(TPR) 被用於擴展 API，但因為一些限制關析，TPR 被 CRD 取代了，並在 v1.8 就被棄用。

* 不用撰寫任何程式碼，就能輕易擴展新 API。
* 能夠沿用熟悉的 Kubernetes UX 工具來管理。如 kubectl。
* 支援 SubResources、Multiple versions、Defaulting、Additional Properties 與 Validation 等等功能。
* 擴展 API 方式簡單，但相對靈活性差。

那要如何新增呢?在跑一個範例前，先來看一下 CRD 是如何定義 API URL。假設想實現在 Kubernetes 上管理 KVM 虛擬機時，我們需要先定義一個用於管理虛擬機的 API 類型 - VM，並且這個 API 是 kairen (或公司與組織)開發，這時 CRD 會透過資源類型名稱+域名來定義，如:域名為`kairen.io`，那麼 CRD 名稱就會是`vms.kairen.io`，而完整 API 資源路徑則是`/apis/kairen.io/<version>/namespaces/<namespace>/vms/..`。

![](https://i.imgur.com/gRkisOJ.png)

基於上述，我們會這樣定義 CRD 內容。當這個範例被建立時，Kubernetes API 伺服器會增加新的 API 資源端點`/apis/kairen.io/v1alpha1/namespaces/<namespace>/vms/..`提供給客戶端存取。而當我們新增一個 VM 實例時，Kubernetes API 伺服器就會幫你管理整個 VMs API 的資源狀態儲存。

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: vms.kairen.io
spec:
  group: kairen.io
  version: v1alpha1
  names:
    kind: VM
    plural: vms 
  scope: Namespaced 
  additionalPrinterColumns:
  - name: Status
    type: string
    description: The VM Phase
    JSONPath: .status.phase
  - name: CPU
    type: integer
    description: CPU Usage
    JSONPath: .status.cpuUtilization
  - name: MEMORY
    type: integer
    description: Memory Usage
    JSONPath: .status.memoryUtilization
  - name: Age
    type: date
    JSONPath: .metadata.creationTimestamp
  validation:
    openAPIV3Schema:
      properties:
        spec:
          properties:
            vmName:
              type: string
              pattern: '^[-_a-zA-Z0-9]+$'
            cpu:
              type: integer
              minimum: 1
            memory:
              type: integer
              minimum: 128
            diskSize:
              type: integer
              minimum: 1
```

> * **scope**: 分為 Cluster 與 Namespaced。前者為叢集面資源(如 PV)，這種 API 通常只有管理員才能操作。
> **additionalPrinterColumns**: 在 kubectl get 時，額外顯示的資訊。能夠以 Json Path 來取得 API 資源內容。
> **validation**: 在呼叫 API 時，能基於 [OpenAPI v3 schema](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#schemaObject) 來驗證 VM 這個物件內容是否符合要求。

當我們利用這個範例在 Kubernetes 建立時，會如下所示:

```sh
$ kubectl get crd
NAME            CREATED AT
vms.kairen.io   2019-10-03T12:30:55Z

$ cat <<EOF | kubectl apply -f -
apiVersion: kairen.io/v1alpha1
kind: VM
metadata:
  name: test-1
spec:
  cpu: 1
  memory: 2048 # MB
  diskSize: 10 # GiB
EOF

$ kubectl get vms
NAME     STATUS   CPU   MEMORY   AGE
test-1                           2m31s
```

這邊會看到 Status 跟 CPU/Memory 使用率都沒更新，因為 CRD 只幫你管理與儲存 API 狀態，若背後沒有一個機制去處理這個 API 時，就不會有實際作用。當然要模擬也是可以，利用 kubectl edit 嘗試修改`.status`內容即可。

### API Aggregation
在 Kubernetes 架構下，每個資源都是由 API 伺服器處理 REST 操作請求，然後管理每個資源物件的狀態儲存。但有些情況下，擴展 API 的開發者，希望自行實現處理 REST API 的所有請求時，就無法透過 CRD 機制來達成。在這種需求下，就要用 API Aggregation 來解決，因為 API Aggregation 能利用一些機制，讓 Kubernetes API 伺服器知道如何委託自定義 API 的請求給第三方 API 伺服器處理。

雖然這種方式能處理更多 API 相關的事情，但相對的程式開發要求較高。那怎麼開發呢?我們能會利用 [k8s.io/apiserver](https://github.com/kubernetes/apiserver) 函式庫來實現。

* 需要程式開發能力，通常建構在 k8s.io/apiserver 函式庫之上。
* 客製化程度非常高。如新增 HTTP verb、實現 Hooks。
* 能完成 CRD 所有能做到的事情。
* 支援 protobuf 與 OpenAPI schema 等等。

> TODO: 補充更多細節

### CRD vs API Aggregation

| CRD         | API Aggregation |
|-------------|-----------------|
|不用寫程式| 要用 Go 語言來開發 API 伺服器| 
|不需要額外服務處理 API 請求與儲存，但還是需要一個控制器來實現資源功能邏輯|需要獨立的第三方服務處理 API 各種事情|
|任何問題都是由 Kubernetes 社區處理與修復|需要同步 Kubernetes 社區問題修復方法，並重新建構 API 伺服器| 
|無需額外處理 API 多版本機制|需要自行處理 API 多版本機制|

> * 詳細英文描述，可以參考 [Comparing ease of use](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#comparing-ease-of-use)。
> * 更多詳細的功能支援比較，可以參考 [Advanced features and flexibility](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#advanced-features-and-flexibility)。

看完這邊大家會發現 CRD 能達到的功能，API Aggregation 也都能達到。但 CRD 上手簡單又快速，API Aggregation 相對要處理事情更多。因此後續開發我會以 CRD 為主。


> TODO: 補充更多比較

## 結語
Kubernetes 提供了非常彈性的方式來擴展 API 功能，開發者能夠依據需求自行選擇擴展方式。也因為這些機制的完善，漸漸地越來越多在 Kubernetes 上新增自定義資源，並開發自家的控制器來管理這些資源，以實現各種在 Kubernetes 的新功能。

到這邊大致上了解一些開發自定義控制器前相關的知識，明天將說明一個 Kubernetes 自定義控制器是如何運作。

## Reference
- https://hackmd.io/@onyiny-ang/HyxVWsS6z?type=view
- https://programming-kubernetes.info/
- https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/