---
title: "探討 Kubernetes 自定義控制器是如何運作 Part2"
date: 2019-10-09
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
在前幾天文章中，認識了開發 Kubernetes 自定義控制器的知識與概念，如: API 函式庫、client-go 函式庫、CRD 與自定義控制器本身等等。但講了這麼多，卻都沒有實際執行一個自定義控制器，因此今天將以 Kubernetes 社區提供的 [sample-controller](https://github.com/kubernetes/sample-controller) 範例為主，來說明如何運行與實作。

<!--more-->
## Sample Controller 架構與運作流程
Sample Controller 是 Kubernetes 官方提供的範例，該範例實現了針對`Foo`自定義資源的控制器，當建立自定義資源 Foo 物件時，該控制器將使用 nginx:latest 容器映像檔與指定的副本數建立 Deployment，換句話說，控制器會確保每個 Foo 資源都有一個對應的 Deployment，其中 Foo 資源的`.spec`內容會與 Deployment 關聯，控制器會在協調循環中依據 Foo 資源的`.spec`內容處理預期的結果。另外該範例也提供了 CRD 的一些功能展示，如: Validation 與 Sub Resources。
 
![](https://i.imgur.com/J9lmgVa.png)

這個控制器流程大致如下:

1. Sample Controller 使用 [client-go](https://github.com/kubernetes/client-go/) 與 [Foo clientset](https://github.com/kubernetes/sample-controller/tree/master/pkg/generated) 函式庫來與 Kubernetes API Server 溝通，並建立一個控制循環(Control Loop)。
2. 在控制循環中，Sample Controller 使用 [Reflector](https://github.com/kubernetes/client-go/blob/master/tools/cache/reflector.go)(實現這功能的是`ListAndWatch()`) 監視 Kubernetes API 中的 Foo 與 Deployment 資源類型(Kind)，以確保兩者持續同步。
3. 當 Reflector 透過 Watch API，收到有關新`Foo`與`Deployment`資源實例存在的事件通知時，它將使用 List API 取得新建立的 API 資源物件，並將其放入`watchHandler`函式內的 DeltaFIFO 佇列中。
4. 接著 Sample Controller 使用 [SharedInformer](https://github.com/kubernetes/client-go/blob/master/tools/cache/controller.go)(`DeploymentInformer` 與 `FooInformer`) 從 DeltaFIFO 佇列中取出 API 資源物件。這邊，SharedInformer 提供了`AddEventHandler()`函式，用於註冊事件處理程式，並將 API 資源物件新增到 Workqueue 中。這邊為 Foo 與 Deployment 傳遞了不同的資源處理函式([L116-L141](https://github.com/kubernetes/sample-controller/blob/master/controller.go#L116-L141))，以處理 API 資源在`add`、`update`與`delete`的事件。如果需要進行某些處理時，這些函式會負責將 Foo 資源物件的鍵(Key)排入 Workqueue。

> * 在本範例中，Workqueue 採用`RateLimitQueue`來進行 API 物件的處理速率限制。
> * 當事件函式被呼叫時，控制器透過`MetaNamespaceKeyFunc()`函式，將當前處理的 API 資源物件轉換為`namespace/name`或`name`(如果沒有 Namespace 的話)的格式作為 Key，然後將這些 Key 添加到 Workqueue。或者透過`DeletionHandlingMetaNamespaceKeyFunc()`來處理刪除事件的 API 資源物件。

5. 這時，Sample Controller 會運行幾個 Worker(完成這個操作的函式是`runWorker()`)，它們會透過不斷的呼叫`processNextWorkItem()`來消耗 Workqueue 中要被處理的 API 物件。這時會進入`syncHandler()`以協調 Foo 當前狀態至預期狀態。而在`syncHandler()`中，控制器會將 Key 恢復成 Namespace 與 Name，並用`Lister`來取得 API 物件內容。

> 在`syncHandler()`中，部分功能會使用 [Indexer](https://github.com/kubernetes/client-go/blob/master/tools/cache/index.go) 引用或者是一個 Listing 封裝器(Wrapper)來檢索對應 Key 的 API 資源物件內容。

6. 在呼叫`syncHandler()`時，控制器會使用指定名稱(Foo 的`.spec.deploymentName`)與副本數(Foo 的`.spec.replicas`)建立一個 Deployment，並同步 Foo 資源的`.status`狀態。

> 在這些過程中，控制器會建立一個 EventRecorder 來紀錄事件的變化過程到 Kubernetes API 中。

而 Sample Controller 整個目錄結構用意如下所示:

```sh
sample-controller
├── Godeps 
├── artifacts
│   └── examples # 存放 Foo 範例，以及 CRD 檔案。
├── docs
│   └── images
├── hack # 存放
└── pkg
    ├── apis
    │   └── samplecontroller
    │       └── v1alpha1 # 自定義資源 Foo 資料結構，會用於 code-generator 產生 client libraries。
    ├── generated # 透過 code-generator 產生的 client libraries。用於跟 API Server 溝通操作 Foo 資源。
    │   ├── clientset
    │   │   └── versioned
    │   │       ├── fake
    │   │       ├── scheme
    │   │       └── typed
    │   │           └── samplecontroller
    │   │               └── v1alpha1
    │   │                   └── fake
    │   ├── informers 
    │   │   └── externalversions
    │   │       ├── internalinterfaces
    │   │       └── samplecontroller
    │   │           └── v1alpha1
    │   └── listers
    │       └── samplecontroller
    │           └── v1alpha1
    └── signals # 實現 Windows 與 POSIX 的 OS shutdown signal
```

## 測試環境部署
由於本次文章將實際執行自定義控制器範例，因此需要建立測試用環境來觀察，這邊請依據需求來完成。

* 需要一座 Kubernetes 叢集。透過 [Minikube](https://github.com/kubernetes/minikube) 建立即可 `minikube start --kubernetes-version=v1.15.4`。
* 安裝 kubectl 工具，請參考 [Install and Set Up kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* 安裝 Go 語言 v1.11+ 開發環境，請參考 [Go Getting Started](https://golang.org/doc/install)。
* 安裝 Git 工具，請參考 [Git](https://git-scm.com/)。

### 設定 Sample Controller
首先透過 Git(或 Go) 取得 sample-controller 原始碼:

```sh
$ git clone https://github.com/kubernetes/sample-controller.git
$ cd sample-controller
```

透過 kubectl 建立該自定義控制器的 CRD 到當前叢集中:

```sh
$ kubectl apply -f artifacts/examples/crd.yaml
customresourcedefinition.apiextensions.k8s.io/foos.samplecontroller.k8s.io created

$ kubectl get crd
NAME                           CREATED AT
foos.samplecontroller.k8s.io   2019-10-06T12:55:18Z
```

### 執行 Sample Controller
當 CRD 建立好後，就可以透過 Go 指令直接執行這個控制器。首先透過 go mod 下載相依函示庫:

```sh
$ export GO111MODULE=on
$ go mod download
```

載完後，即可透過以下指令來執行控制器:

```sh
$ go run $(ls -1 *.go | grep -v _test.go) -kubeconfig=$HOME/.kube/config -v=3 -logtostderr
I1006 21:14:32.364765   73322 controller.go:114] Setting up event handlers
I1006 21:14:32.364892   73322 controller.go:155] Starting Foo controller
I1006 21:14:32.364905   73322 controller.go:158] Waiting for informer caches to sync
I1006 21:14:32.365031   73322 reflector.go:150] Starting reflector *v1.Deployment (30s) from pkg/mod/k8s.io/client-go@v0.0.0-20191005115821-b1fd78950135/tools/cache/reflector.go:105
I1006 21:14:32.365724   73322 reflector.go:185] Listing and watching *v1.Deployment from pkg/mod/k8s.io/client-go@v0.0.0-20191005115821-b1fd78950135/tools/cache/reflector.go:105
I1006 21:14:32.365032   73322 reflector.go:150] Starting reflector *v1alpha1.Foo (30s) from pkg/mod/k8s.io/client-go@v0.0.0-20191005115821-b1fd78950135/tools/cache/reflector.go:105
I1006 21:14:32.365796   73322 reflector.go:185] Listing and watching *v1alpha1.Foo from pkg/mod/k8s.io/client-go@v0.0.0-20191005115821-b1fd78950135/tools/cache/reflector.go:105
I1006 21:14:32.467616   73322 controller.go:163] Starting workers
I1006 21:14:32.467660   73322 controller.go:169] Started workers
```

### 建立 Foo 資源實例
當執行了控制器後，就可以開一個新 Terminal 來建立 Foo 實例，以觀察控制器執行的結果:

```sh
$ cd sample-controller
$ kubectl apply -f artifacts/examples/example-foo.yaml
foo.samplecontroller.k8s.io/example-foo created

$ kubectl get foo
NAME          AGE
example-foo   48s

$ kubectl get deploy,po
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/example-foo   1/1     1            1           5m22s

NAME                              READY   STATUS    RESTARTS   AGE
pod/example-foo-d75d8587c-wlqsz   1/1     Running   0          5m22s
```

從範例中，可以發現 Foo 實際功能是管理著一個 Deployment 的建立，因此會看到上述的結果。當嘗試修改 Foo 中的`.spec.replicas`時，會發現 Foo 管理的 Deployment 會跟著變動副本數。

## 結語
Sample Controller 雖然只是一個非常簡單的自定義控制器範例，但從中卻可以學習到很多觀念，從最基本的 API 資源結構定義、透過 code-generator 產生客戶端函式庫、管理原有的 Kubernetes API 資源等等。雖然範例交代了很多基礎觀念，但是在實際應用上，還是缺乏了一些元素，比如說:多個相同控制器的 HA 實現、如何取得控制器 Metrics、怎麼使用 Finalizer 實作垃圾資源回收、實際部署方式與 RBAC 設定等等問題與功能。

在接下來章節中，我將針對這部份一一說明，我會透過每天實作一小部分來完成一個自定義控制器，在過程中再把一些觀念進一步釐清。

## Reference
- https://engineering.bitnami.com/articles/a-deep-dive-into-kubernetes-controllers.html
- https://engineering.bitnami.com/articles/kubewatch-an-example-of-kubernetes-custom-controller.html
- https://itnext.io/building-an-operator-for-kubernetes-with-the-sample-controller-b4204be9ad56
- https://github.com/kubernetes/sample-controller