---
title: "探討 Kubernetes 自定義控制器是如何運作 Part1"
date: 2019-10-08
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
Kubernetes 控制器主要目的是協調(Reconcile)某個 API 資源的狀態，從實際狀態轉換為期望狀態，換句話說，在 Kubernetes API 資源上的說法，就是讓 API 資源的`.status`狀態，達到`.spec`所定義內容。而為了達到這點，控制器會透過 client-go 一直監視這兩種狀態的變化，並在發現變化時，觸發協調邏輯的循環，以更改當前狀態所需的任何操作，並使其往資源的預期狀態進展。Kubernetes 為了實現這樣機制，在 API 提供了一組 List 與 Watch 方法，用於監視任何 API 資源的事件。而自定義控制器就是以 client-go 操作這些 API 方法，監視其主要的自定義資源與其他任何相關的資源。

<!--more-->

舉例，想要實現管理 TensorFlow 分散式訓練的控制器。這個控制器不僅要監視 TensorFlowJob(這是自定義資源) 物件的變化，而必須響應 Pod 事件，以確保 Pod 能再發生狀況時，處理對應狀況。當然這種追蹤 API 資源之間關聯的機制，也能利用 Kubernetes 的 Owner references 機制達成。這機制允許控制器在任何 API 資源上，設定資源的父子關析(如 Deployment 與 Pod 關聯這樣)，而當子資源事件發生時，就能反應給控制器，以知道哪個 TensorFlowJob 物件已受到影響，這時再由控制器的檢查與協調循環，來解決狀態的變化。

但講這麼多，一個控制器內部究竟是如何運作呢?今天就是要來聊聊這個內容。

## 控制器如何運作?
這部分將解釋 Kubernetes 自定義控制器運作流程，如下圖所示。一個自定義控制器是由 client-go 中的幾個主要功能實現，並結合自己實現的邏輯來達成。因此在開發前，必須先理解這些名詞與功能是做什麼用的。

![Kubernetes controller diagram](https://i.imgur.com/NYo9CsO.png) 
(圖片擷取自：[Kubernetes sample-controller](https://github.com/kubernetes/sample-controller/blob/master/docs/images/client-go-controller-interaction.jpeg))

從架構圖來看，主要的 client-go 會有以下三者處理:

* **Reflector**: 會透過 List/Watch API 監視著 Kubernetes 中指定的資源類型(Kind)，而這些資源可以是既有的資源(如 Pod、Deployment)，也可以是自定義資源。當 Reflector 透過 Watch API 收到新資源實例建立通知時，會將透過該資源的 List API 取得新建立的物件，並將物件放到 Delta Fifo 佇列中。
* **Informer**: 是控制器機制的基礎，它會在協調循環中，從 Delta Fifo 佇列取出 API 資源，將其儲存成物件提供給 Indexer 快取，並提供物件事件的處理介面(Add、Update 與 Delete)。
* **Indexer**: 為 API 資源物件提供檢索功能。利用執行緒安全的資料儲存中，將物件儲存成鍵(Key)/值(Value)形式，其 Key 的格式為 `namespace/name`。

而除了上面 client-go 元件外，在開發一個自定義控制器時，還會有以下幾個元件會被使用到:

* **Informer reference**: 在自定義控制器使用的 Informer 引用。在開發自定義控制器時，通常會宣告一個 Informer 實例用於多個 API 資源監聽使用，但自定義控制器有可能會需要監聽不同 API 資源，因此可能有多個子控制器，這時就可以用工廠模式(Factory Pattern)傳遞進去，並設定該控制器監聽的 API 資源。
* **Indexer reference**: 同 Informer 概念。用於處理不同 API 資源的檢索。
* **Resource Event Handlers**: 這些 Informer 的回呼函式(callback functions)，分別有`onAdd()`、`onUpdate()`與`onDelete()`。當 Informer 收到 API 資源物件事件時，就會呼叫這些函式，這時自定義控制器就能在函式中處理接下來事情。
* **Work queue**: 當 Resource Event Handlers 被呼叫時，會將 Key 寫到這個佇列中，以確保 API 資源物件能被依序處理。在 client-go 支援不同的 Workqueue 類型，而這邊通常會以 RateLimiting 為主。
* **Process Item**: 從 Workqueue 中取出物件 Key 的過程。
* **Handle Object**: 處理物件的實際邏輯，在開發自定義控制器時，通常會在這邊撰寫功能邏輯。一般來說會利用一個 Indexer reference 從取得的 Key 來檢索指定 API 資源物件的內容，並依據內容當前狀態與預期狀態處理。

> TODO: 補充更多細節

## 結語
今天理解了自定義控制器使用到的元件，以及其運作方式，從中可以發現 Kubernetes client-go 幾乎是整個控制器的核心，許多功能實現都是圍繞著該函式庫。明天將部署 [sample-controller](https://github.com/kubernetes/sample-controller) 到 Minikube 上，以釐清一些功能運作流程，這些知識與觀念將用於後續範例開發中。

雖然 Kubernetes 控制器看起來似乎很複雜，事實上簡化來看，大概也就長這個樣子:

![](https://i.imgur.com/RZAuO0K.png)

## Reference
- https://itnext.io/how-to-create-a-kubernetes-custom-controller-using-client-go-f36a7a7536cc
- https://github.com/kubernetes/sample-controller
- https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/
- http://www.edwardesire.com/2019/05/14/kubernetesbian-controller-pattern/
- https://kubernetes.io/docs/reference/using-api/client-libraries/
- https://speakerdeck.com/chanyilin/k8s-metacontroller
- https://toutiao.io/posts/4rnwh6/preview
- https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources
- https://www.oreilly.com/library/view/cloud-native-infrastructure/9781491984291/ch04.html
- https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/
- https://github.com/opsnull/kubernetes-dev-docs/tree/master/client-go
- https://blog.csdn.net/weixin_42663840/article/details/81482553