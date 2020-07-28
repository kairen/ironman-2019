---
title: "開發自定義控制器前，需要先了解的東西 Part2"
date: 2019-10-06
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
昨天提到 Kubernetes GitHub 組織上，有許多豐富的程式函式庫可以使用，除了昨天介紹的一些關於 API 的函式庫外，還有用於跟 Kubernetes API 伺服器溝通的客戶端函式庫，如: client-go，而這些客戶端函式庫在開發 Kubernetes 自定義控制器時，是幾乎避免不了，甚至整個 Kubernetes 控制器架構都是圍繞這些函式庫上實現，而今天就是要針對這些客戶端函式庫做初步認識。

<!--more-->

### client-go
由於 Kubernetes 是以 Go 語言打造的專案，因此相關函式庫都是以 Go 語言來提供使用，因此將 Kubernetes 客戶端函式庫稱之為 [client-go](https://github.com/kubernetes/client-go)。而 client-go 是一套典型的 Web service 函式庫，它支援了所有 Kubernetes 正式的 API 資源類型，並對它進行以下 REST 操作:

* Create
* Get
* List
* Update
* Delete
* Patch

這些 REST 操作動詞都是基於 API 伺服器的 HTTP 介面實現。另外 client-go 也支援了 Watch API，這是透過 HTTP Streaming 機制來監聽資源在叢集中的變化事件。透過 client-go 我們能在程式做到什麼事情呢?這邊列舉幾個:

* 允許操作資源狀態(如:新增 Pod、修改 ConfigMap 或刪除 Persistent Volume)。
* 列出所有資源。
* 獲取有關當前資源狀態的詳細訊息。
* 開發自定義控制器。

而 Kubernetes 的 client-go 背後會使用到上述提到的 api、api-machinery 等等函式庫，因此在導入時，需要注意一下版本相容性，如下圖。

![](https://i.imgur.com/aXjpQfT.png)

![Compatibility matrix](https://i.imgur.com/FHb4taS.png)

client-go 是官方最主要的 API client 函式庫，它在許多地方被使用，如 kubectl、kubeadm 與各種控制器等等。而目前 client-go 整體以目錄分成以下功能:

* **kubernetes**: 提供原生 Kubernetes API 的資源 REST 操作方法與結構，如 Pod。
* **informers**: 提供原生 Kubernetes API 的資源 List/Watch API 機制與功能。經常被用於實現控制器中。
* **listers**: 提供原生 Kubernetes API 的資源從 Local cache 取得 API 等功能。經常被用於實現控制器中。
* **discovery**: 用於發現 Kubernetes API 伺服器支援哪些的 API 群組、版本與資源方法。
* **dynamic**: 提供一個動態的客戶端，能用於操作任何 Kubernetes 上的 API 資源。功能與`kubernetes`套件類似，差別在於`kubernetes`是針對每種 API 資源提供自己的操作方法。
* **transport**: 提供 TCP 授權/連接、Stream(如: exec、logs 與 portforward 等)、Websocket 等等功能。如果沒有明確選擇協定的話，預設會使用 HTTP2 進行溝通。其中 Stream 部分，若不支援 HTTP2 的話，則採用 [SPDY](https://zh.wikipedia.org/wiki/SPDY) 實現。
* **rest**: 提供 REST 客戶端的介面與實現，為 client-go 的 `kubernetes` package 基礎。
* **plugin**: 提供雲端供應商(Cloud Provider)的身份認證插件。
* **tools**: 提供各種方便使用的功能與工具，如 Cache、LeaseLock、Metrics 等等。
* **scale**: 提供 Auto Scaling 相關的客戶端。
* **util**: 提供各種方便使用的程式功能，如 Workqueue、Flow control、Certificate 等等。
* **examples**: 提供各種範例，如 Workqueue、Fake Client 等等操作。

> TODO: 補範例

### apiextensions-apiserver
[apiextensions-apiserver](https://github.com/kubernetes/apiextensions-apiserver) 類似 client-go 功能，但主要為 CRD(CustomResourceDefinitions) API 的資源結構，以及用於操作該資源的 Client 函式庫。如下面範例。

```go
type customResource struct {
	Name       string
	Kind       string
	Group      string
	Plural     string
	Version    string
	Scope      apiextensionsv1beta1.ResourceScope
	ShortNames []string
}

func createCRD(clientset apiextensionsclientset.Interface, resource customResource) error {
	crdName := fmt.Sprintf("%s.%s", resource.Plural, resource.Group)
	crd := &apiextensionsv1beta1.CustomResourceDefinition{
		ObjectMeta: metav1.ObjectMeta{
			Name: crdName,
		},
		Spec: apiextensionsv1beta1.CustomResourceDefinitionSpec{
			Group:   resource.Group,
			Version: resource.Version,
			Scope:   resource.Scope,
			Names: apiextensionsv1beta1.CustomResourceDefinitionNames{
				Singular:   resource.Name,
				Plural:     resource.Plural,
				Kind:       resource.Kind,
				ShortNames: resource.ShortNames,
			},
		},
	}
	_, err := clientset.ApiextensionsV1beta1().CustomResourceDefinitions().Create(crd)
	if err != nil {
		if !errors.IsAlreadyExists(err) {
			return fmt.Errorf("failed to create %s CRD. %+v", resource.Name, err)
		}
	}
	return nil
}

func main() {
    res := customResource{
		{
			Name:    "security",
			Plural:  "securities",
			Kind:    reflect.TypeOf(blendedv1.Security{}).Name(),
			Group:   blendedv1.CustomResourceGroup,
			Version: blendedv1.Version,
			Scope:   apiextensionsv1beta1.NamespaceScoped,
		},
	}
	createCRD(extensionsClient, res)
}
```

> 範例並未提供完整內容，僅擷取部分資訊用以說明。

## 結語
client-go 與 apiextensions-apiserver 是開發一個原生 Kubernetes 控制器的主要函式庫，整個控制器的工作流程與功能，都會利用這兩個客戶端函式庫來完成。而 client-go 除了提供各種 API 資源物件以外，也有各種方便的介面(Interface)、功能(Function)與方法(Method)，如: LeaseLock、Metrics 等等，能讓我們使用。

有了這些函式庫後，就能夠以程式實現各種操作功能，比如說 [Websocket Pod Exec](https://github.com/kairen/websocket-exec) 這個範例，就是利用 client-go transport 實作。

今天主要分享有關客戶端的函式庫，明天將認識擴展 API 時，需要知道的兩個方法。

## Reference
- https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/
- http://www.edwardesire.com/2019/05/14/kubernetesbian-controller-pattern/
- https://kubernetes.io/docs/reference/using-api/client-libraries/
- https://github.com/kubernetes-client/gen
- https://speakerdeck.com/chanyilin/k8s-metacontroller
- https://toutiao.io/posts/4rnwh6/preview