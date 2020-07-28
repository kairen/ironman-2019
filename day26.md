---
title: "動手實作 Kubernetes 自定義控制器 Part2"
date: 2019-10-11
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
在[動手實作 Kubernetes 自定義控制器 Part1](https://k2r2bai.com/2019/10/10/ironman2020/day25/)文章中，我們透過定義 API 資源結構，以及使用 code-generator 產生了用於開發自定義控制器的程式函式庫。今天將延續範例，利用昨天產生的函式庫(apis, clientsets)建立一個控制器程式，以監聽自定義資源`VirtualMachine`的 API 事件。

<!--more-->

## 實現控制器程式
當有了自定義資源的 api 與 client 的函式庫後，我們就能利用這些來撰寫控制器程式。延續 [Controller101](https://github.com/cloud-native-taiwan/controller101)，我們將新增一些檔案來完成，如下所示:

```sh
├── cmd
│   └── main.go
├── example
│   └── test-vm.yml 
└── pkg
    ├── controller
    │   └── controller.go
    └── version
        └── version.go
```

* **cmd/main.go**: 為控制器的主程式。
* **example/test-vm.yml**: 用於測試控制器的 VirtualMachine 資源的範例檔。(optional)
* **pkg/controller/controller.go**: VirtualMachine 控制器核心程式。
* **pkg/version/version.go**: 用於 Go build 時加入版本號。(optional)

> 目前 GitHub 範例已經新增這些程式，若不想看這累死人沒排版文章，可以直接透過 git 抓下來跑。

### pkg/controller/controller.go
該檔案會利用 Kubernetes client-go 函式庫，以及 code-generator 產生的程式函式庫來實現控制器核心功能。通常撰寫一個控制器時，會建立一個 Controller struct，並包含以下元素:

* **Clientset**: 擁有 VirtualMachine 的客戶端介面，讓控制器與 Kubernetes API Server 進行互動，以操作 VirtualMachine 資源。
* **Informer**: 控制器的 SharedInformer，用於接收 API 事件，並呼叫回呼函式。
* **InformerSynced**: 確認 SharedInformer 的儲存是否以獲得至少一次完整 LIST 通知。
* **Lister**: 用於列出或獲取快取中的 VirtualMachine 資源。
* **Workqueue**: 控制器的資源處理佇列，都 Informer 收到事件時，會將物件推到這個佇列，並在協調程式取出處理。當發生錯誤時，可以用於 Requeue 當前物件。

```go
package controller

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	cloudnative "github.com/cloud-native-taiwan/controller101/pkg/generated/clientset/versioned"
	cloudnativeinformer "github.com/cloud-native-taiwan/controller101/pkg/generated/informers/externalversions"
	listerv1alpha1 "github.com/cloud-native-taiwan/controller101/pkg/generated/listers/cloudnative/v1alpha1"
	"github.com/golang/glog"
	"k8s.io/apimachinery/pkg/api/errors"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/util/workqueue"
	"k8s.io/klog"
)

const (
	resouceName = "VirtualMachine"
)

type Controller struct {
	clientset cloudnative.Interface
	informer  cloudnativeinformer.SharedInformerFactory
	lister    listerv1alpha1.VirtualMachineLister
	synced    cache.InformerSynced
	queue     workqueue.RateLimitingInterface
}

func New(clientset cloudnative.Interface, informer cloudnativeinformer.SharedInformerFactory) *Controller {
	vmInformer := informer.Cloudnative().V1alpha1().VirtualMachines()
	controller := &Controller{
		clientset: clientset,
		informer:  informer,
		lister:    vmInformer.Lister(),
		synced:    vmInformer.Informer().HasSynced,
		queue:     workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), resouceName),
	}

	vmInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: controller.enqueue,
		UpdateFunc: func(old, new interface{}) {
			controller.enqueue(new)
		},
	})
	return controller
}

func (c *Controller) Run(ctx context.Context, threadiness int) error {
	go c.informer.Start(ctx.Done())
	klog.Info("Starting the controller")
	klog.Info("Waiting for the informer caches to sync")
	if ok := cache.WaitForCacheSync(ctx.Done(), c.synced); !ok {
		return fmt.Errorf("failed to wait for caches to sync")
	}

	for i := 0; i < threadiness; i++ {
		go wait.Until(c.runWorker, time.Second, ctx.Done())
	}
	klog.Info("Started workers")
	return nil
}

func (c *Controller) Stop() {
	glog.Info("Stopping the controller")
	c.queue.ShutDown()
}

func (c *Controller) runWorker() {
	defer utilruntime.HandleCrash()
	for c.processNextWorkItem() {
	}
}

func (c *Controller) processNextWorkItem() bool {
	obj, shutdown := c.queue.Get()
	if shutdown {
		return false
	}

	err := func(obj interface{}) error {
		defer c.queue.Done(obj)
		key, ok := obj.(string)
		if !ok {
			c.queue.Forget(obj)
			utilruntime.HandleError(fmt.Errorf("Controller expected string in workqueue but got %#v", obj))
			return nil
		}

		if err := c.syncHandler(key); err != nil {
			c.queue.AddRateLimited(key)
			return fmt.Errorf("Controller error syncing '%s': %s, requeuing", key, err.Error())
		}

		c.queue.Forget(obj)
		glog.Infof("Controller successfully synced '%s'", key)
		return nil
	}(obj)

	if err != nil {
		utilruntime.HandleError(err)
		return true
	}
	return true
}

func (c *Controller) enqueue(obj interface{}) {
	key, err := cache.MetaNamespaceKeyFunc(obj)
	if err != nil {
		utilruntime.HandleError(err)
		return
	}
	c.queue.Add(key)
}

func (c *Controller) syncHandler(key string) error {
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("invalid resource key: %s", key))
		return err
	}

	vm, err := c.lister.VirtualMachines(namespace).Get(name)
	if err != nil {
		if errors.IsNotFound(err) {
			utilruntime.HandleError(fmt.Errorf("virtualmachine '%s' in work queue no longer exists", key))
			return err
		}
		return err
	}

	data, err := json.Marshal(vm)
	if err != nil {
		return err
	}

	klog.Infof("Controller get %s/%s object: %s", namespace, name, string(data))
	return nil
}
```

### cmd/main.go
該檔案為控制器主程式，主要提供 Flags 來設定控制器參數、初始化所有必要的程式功能(如 REST Client、K8s Clientset、K8s Informer 等等)，以及執行控制器核心程式。

```go
package main

import (
	"context"
	goflag "flag"
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/cloud-native-taiwan/controller101/pkg/controller"
	cloudnative "github.com/cloud-native-taiwan/controller101/pkg/generated/clientset/versioned"
	cloudnativeinformer "github.com/cloud-native-taiwan/controller101/pkg/generated/informers/externalversions"
	"github.com/cloud-native-taiwan/controller101/pkg/version"
	flag "github.com/spf13/pflag"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/klog"
)

const defaultSyncTime = time.Second * 30

var (
	kubeconfig  string
	threads     int
)

func parseFlags() {
	flag.StringVarP(&kubeconfig, "kubeconfig", "", "", "Absolute path to the kubeconfig file.")
	flag.IntVarP(&threads, "threads", "", 2, "Number of worker threads used by the controller.")
	flag.BoolVarP(&showVersion, "version", "", false, "Display the version.")
	flag.CommandLine.AddGoFlagSet(goflag.CommandLine)
	flag.Parse()
}

func restConfig(kubeconfig string) (*rest.Config, error) {
	if kubeconfig != "" {
		cfg, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
		if err != nil {
			return nil, err
		}
		return cfg, nil
	}

	cfg, err := rest.InClusterConfig()
	if err != nil {
		return nil, err
	}
	return cfg, nil
}

func main() {
	parseFlags()

	k8scfg, err := restConfig(kubeconfig)
	if err != nil {
		klog.Fatalf("Error to build rest config: %s", err.Error())
	}

	clientset, err := cloudnative.NewForConfig(k8scfg)
	if err != nil {
		klog.Fatalf("Error to build cloudnative clientset: %s", err.Error())
	}

	informer := cloudnativeinformer.NewSharedInformerFactory(clientset, defaultSyncTime)
	controller := controller.New(clientset, informer)
	ctx, cancel := context.WithCancel(context.Background())
	signalChan := make(chan os.Signal, 1)
	signal.Notify(signalChan, syscall.SIGINT, syscall.SIGTERM)

	if err := controller.Run(ctx, threads); err != nil {
		klog.Fatalf("Error to run the controller instance: %s.", err)
	}

	<-signalChan
	cancel()
	controller.Stop()
}
```

其中`restConfig()`函式用於建立 RESTClient Config，如果有指定 Kubeconfig 檔案時，會透過`client-go/tools/clientcmd`解析 Kubeconfig 內容以產生 Config 內容;若沒有的話，則表示該控制器可能被透過 Pod 部署在 Kubernetes 中，因此使用 [InClusterConfig](https://github.com/kubernetes/client-go/blob/master/rest/config.go#L406) 方式建立 Config。

### 執行
當控制器程式實現完成，且已經擁有一座安裝好 VirtualMachine CRD 的 Kubernetes 時，就能透過以下指令來執行:

```sh
$ go run cmd/main.go --kubeconfig=$HOME/.kube/config -v=2 --logtostderr
I1008 15:38:30.350446   52017 controller.go:68] Starting the controller
I1008 15:38:30.350543   52017 controller.go:69] Waiting for the informer caches to sync
I1008 15:38:30.454799   52017 controller.go:77] Started workers
```

接著開啟另一個 Terminal 來建立 VirtualMachine 實例:

```sh
$ cat <<EOF | kubectl apply -f -
apiVersion: cloudnative.tw/v1alpha1
kind: VirtualMachine
metadata:
  name: test-vm
spec:
  resource:
    cpu: 2
    memory: 4G
EOF
virtualmachine.cloudnative.tw/test-vm created
```

這時觀察控制器，會看到以下資訊:

```sh
$ go run cmd/main.go --kubeconfig=$HOME/.kube/config -v=3 --logtostderr
...
I1008 17:28:18.775656   56945 controller.go:156] Controller get default/test-vm object: {"metadata":{"name":"test-vm","namespace":"default","selfLink":"/apis/cloudnative.tw/v1alpha1/namespaces/default/virtualmachines/test-vm","uid":"a1acb111-c71e-4d2b-a2f4-62605e616dfc","resourceVersion":"52295","generation":1,"creationTimestamp":"2019-10-08T09:28:18Z","annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"cloudnative.tw/v1alpha1\",\"kind\":\"VirtualMachine\",\"metadata\":{\"annotations\":{},\"name\":\"test-vm\",\"namespace\":\"default\"},\"spec\":{\"action\":\"active\",\"resource\":{\"cpu\":2,\"memory\":\"4G\",\"rootDisk\":\"40G\"}}}\n"}},"spec":{"action":"active","resource":{"cpu":"2","memory":"4G","rootDisk":"40G"}},"status":{"phase":"","server":{"state":"","usage":{"cpu":0,"memory":0}},"lastUpdateTime":null}}
I1008 17:28:18.775687   56945 controller.go:115] Controller successfully synced 'default/test-vm'
```

## 結語
透過今天的實作，可以發現使用 code-generator 產生的相關程式碼操作自定義資源，就如同 Kubernetes client-go 的原生 API clientsets 一樣簡單，只要根據 [sample-controller](https://github.com/kubernetes/sample-controller) 內容做些調整，就能實現特定 API 資源的控制器程式。

## Reference
- https://github.com/kubernetes/sample-controller
- https://itnext.io/how-to-create-a-kubernetes-custom-controller-using-client-go-f36a7a7536cc
