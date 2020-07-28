---
title: "動手實作 Kubernetes 自定義控制器 Part4"
date: 2019-10-13
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
在[動手實作 Kubernetes 自定義控制器 Part3](https://k2r2bai.com/2019/10/12/ironman2020/day27/) 文章中，了解如何實現自定義控制器的高可靠架構，而今天將延續之前位完成的部分，會簡單以 Docker 實作一個虛擬機驅動來提供給自定義控制器使用，控制器會依據自定義資源`VirtualMachine`的內容，來協調完成預期結果的事情。如下架構圖所示。

<!--more-->

![](https://i.imgur.com/3xhW8am.png)

由於為了方便大家在 Minikube 上執行這個自定義控制器，因此這邊實作了一個 [VM Driver](https://github.com/cloud-native-taiwan/controller101/blob/master/pkg/driver/driver.go) 的 Golang 介面，並以該介面實現一個 [Docker Driver](https://github.com/cloud-native-taiwan/controller101/blob/master/pkg/driver/docker.go) 來作為使用。這個 Docker Driver 會以 Docker 預設的系統環境變數來載入 Endpoint、Certs 等等 Docker client 需要的資訊，接著透過這些資訊建立一個 client 與 Docker API 溝通進行各種操作。這邊介面只簡單實現以下函式來完成範例:

```golang
type Interface interface {
	CreateServer(*CreateRequest) (*CreateReponse, error)
	DeleteServer(name string) error
	IsServerExist(name string) (bool, error)
	GetServerStatus(name string) (*GetStatusReponse, error)
}
```

當控制器收到 VirtualMachine 的實例建立時，控制器會在`syncHandler()`函式依據接受到的資源物件資訊來呼叫 VM Driver 進行處理相關事情(如建立虛擬機環境、更新虛擬機使用率等等)，當處理完成後，再依據回應的內容更新到 VirtualMachine 資源實例的`.status`內容。而當控制器收到有個 VirtualMachine 實例被刪除時，就會呼叫 Informer 的`DeleteFunc`來進行處理實際虛擬機移除的事情。

> * 由於這只是為了說明如何開發控制器，因此該範例使用的 Docker Driver 在建立容器時，只會以 NGINX 映像檔為基礎來建立。
> * 原本規劃 Fake Driver 與 KVM 來模擬，但因為時間關析，只能之後再補上。

## 協調資源
本部分將修改`controller.go`程式，以實現自定義資源 VirtualMachine 管理虛擬機的機制。

### 環境準備
由於使用這個功能需要用到 Kubernetes 與 Go 語言，因此需要透過以下來完成條件:

* 一座 Kubernetes v1.10+ 叢集。透過 [Minikube](https://github.com/kubernetes/minikube) 建立即可 `minikube start --kubernetes-version=v1.15.4`。
* 一個 Docker 環境，可以直接 Minikube 執行`eval $(minikube docker-env)`來取的 Docker 參數，並遠端操作。
* 安裝 Go 語言 v1.11+ 開發環境，由於開發中會使用到 Go mod 來管理第三方套件，因此必須符合支援版本。安裝請參考 [Go Getting Started](https://golang.org/doc/install)。

### 管理虛擬機邏輯實現
前幾天我們在實作控制器時，有提到主要處理 API 資源實例的函式是`syncHandler()`，因此大部分邏輯會在這邊實現。但由於該控制器需要透過一些方法管理實際的虛擬機，因此這邊以 [VM Driver](https://github.com/cloud-native-taiwan/controller101/blob/master/pkg/driver/driver.go) 實現 [Docker Driver](https://github.com/cloud-native-taiwan/controller101/blob/master/pkg/driver/docker.go) 方式進行模擬。這時要修改控制器結構與建構子如下。

#### pkg/controller/controller.go
```go
type Controller struct {
	...
	vm        driver.Interface // 管理實際虛擬機的驅動程式
}

func New(clientset cloudnative.Interface, informer cloudnativeinformer.SharedInformerFactory, vm driver.Interface) *Controller {
    ...
	controller := &Controller{
        ...
        vm:        vm,
	}

    ...
	return controller
}
```

> `...` 表示不更改內容。完整程式請參考 [controller.go L44-L72](https://github.com/cloud-native-taiwan/controller101/blob/9669f5e1e744fdc33baf5c0c229c9cb27f095b20/pkg/controller/controller.go#L44-L72)

完成後，就可以透過上面物件，在這個控制器結構的函式操作虛擬機。接著在`syncHandler()`實現協調循環的邏輯:

```go
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

	switch vm.Status.Phase {
	case v1alpha1.VirtualMachineNone:
		if err := c.makeCreatingPhase(vm); err != nil {
			return err
		}
	case v1alpha1.VirtualMachinePending, v1alpha1.VirtualMachineFailed:
		if err := c.createServer(vm); err != nil {
			return err
		}
	case v1alpha1.VirtualMachineActive:
		if err := c.updateUsage(vm); err != nil {
			return err
		}
	}
	return nil
}

// 用於更新 VirtualMachine 資源狀態的通用函式
func (c *Controller) updateStatus(vm *v1alpha1.VirtualMachine, phase v1alpha1.VirtualMachinePhase, reason error) error {
	vm.Status.Reason = ""
	if reason != nil {
		vm.Status.Reason = reason.Error()
	}

	vm.Status.Phase = phase
	vm.Status.LastUpdateTime = metav1.NewTime(time.Now())
	_, err := c.clientset.CloudnativeV1alpha1().VirtualMachines(vm.Namespace).Update(vm)
	return err
}

// 用於將虛擬機狀態新增到 VirtualMachine 資源的通用函式
func (c *Controller) appendServerStatus(vm *v1alpha1.VirtualMachine) error {
	status, err := c.vm.GetServerStatus(vm.Name)
	if err != nil {
		return err
	}

	vm.Status.Server.Usage.CPU = status.CPUPercentage
	vm.Status.Server.Usage.Memory = status.MemoryPercentage
	vm.Status.Server.State = status.State
	return nil
}
```

當收到 Informer 的 Add/Update 事件時，會將資源實例的物件放到 Workqueue，然後控制器的 Workers 會呼叫`processNextWorkItem()`函式來持續消化 Workqueue 中的物件，並在取出物件的 Key 後，將其丟到`syncHandler()`函式處理。而`syncHandler()`函式會透過 Lister 從本地快取中獲取資源實例的內容，這時我們就能透過內容的狀態來處理對應事情。以上面程式為例，我們分成以下幾個狀態來處理。

> 這邊使用不同狀態來處理不同過程，其目的是確保控制器不會因為實例的狀態更新，而一直觸發 Update 事件導致無限循環，因此以狀態來做收斂的點。

* **VirtualMachineNone**: 由於 VirtualMachine 資源實例被建立時，並不會有任何資源狀態，因此該狀態可用於判斷是否是第一次建立，若是的話則將狀態更新為 Creating，這樣可以讓該資源被標示為即將建立虛擬機。程式內容如下:

```go
func (c *Controller) makeCreatingPhase(vm *v1alpha1.VirtualMachine) error {
	vmCopy := vm.DeepCopy()
	return c.updateStatus(vmCopy, v1alpha1.VirtualMachineCreating, nil)
}
```

* **VirtualMachineCreating**: 當處於 Creating 時，控制器會呼叫 VM Driver 以建立虛擬機，若成功的話，則更新 VirtualMachine 資源的`.status`為 Active 狀態;若失敗的話，則標示為 Failed，狀態，並提供失敗原因的訊息。程式內容如下:

```go
func (c *Controller) createServer(vm *v1alpha1.VirtualMachine) error {
	vmCopy := vm.DeepCopy()
	ok, _ := c.vm.IsServerExist(vm.Name)
	if !ok {
		req := &driver.CreateRequest{
			Name:   vm.Name,
			CPU:    vm.Spec.Resource.Cpu().Value(),
			Memory: vm.Spec.Resource.Memory().Value(),
		}
		resp, err := c.vm.CreateServer(req)
		if err != nil {
			if err := c.updateStatus(vmCopy, v1alpha1.VirtualMachineFailed, err); err != nil {
				return err
			}
			return err
		}
		vmCopy.Status.Server.ID = resp.ID

		if err := c.appendServerStatus(vmCopy); err != nil {
			return err
		}

		if err := c.updateStatus(vmCopy, v1alpha1.VirtualMachineActive, nil); err != nil {
			return err
		}
	}
	return nil
}
```

* **VirtualMachineFailed**: 類似 Creating 狀態，當嘗試透過 VM Driver 建立虛擬機失敗時，會讓資源 Requeuing 到 Workqueue 中，並繼續在協調循環中重新嘗試建立虛擬機，直到建立成功或 VirtualMachine 資源被刪除。程式內容同 Creating 狀態。
* **VirtualMachineActive**: 當虛擬機被正確建立，並且能夠取得狀態後，就會進入 Active 狀態。而當資源一直處於 Active 時，就能夠持續透過 VM Driver 獲取當前虛擬機狀態，並更新到 API 資源上。程式內容如下:

```go
func (c *Controller) updateUsage(vm *v1alpha1.VirtualMachine) error {
	vmCopy := vm.DeepCopy()
	t := subtractTime(vmCopy.Status.LastUpdateTime.Time)
	if t.Seconds() > periodSec {
		if err := c.appendServerStatus(vmCopy); err != nil {
			return err
		}

		if err := c.updateStatus(vmCopy, v1alpha1.VirtualMachineActive, nil); err != nil {
			return err
		}
	}
	return nil
}
```

> 這邊`subtractTime()`用於避免控制器一直執行`updateStatus()`，而導致無限循環。

而當 API 資源物件被刪除時，Informer 會呼叫`DeleteFunc`的對應函式`deleteObject()`來刪除虛擬機。程式內容如下:

```go
func (c *Controller) deleteObject(obj interface{}) {
	vm := obj.(*v1alpha1.VirtualMachine)
	if err := c.vm.DeleteServer(vm.Name); err != nil {
		klog.Errorf("Failed to delete the '%s' server: %v", vm.Name, err)
	}
}
```

#### cmd/main.go
當 Controller 與 VM Driver 程式都完成後，就可以修改主程式來反映功能改變:

```go
var (
	...
	driverName         string
)

func parseFlags() {
	...
	flag.StringVarP(&driverName, "vm-driver", "", "", "Driver is one of: [fake docker].")
	...
}

func main() {
    ...
    var vmDriver driver.Interface
	switch driverName {
	case "docker":
		docker, err := driver.NewDockerDriver()
		if err != nil {
			klog.Fatalf("Error to create docker driver: %s", err.Error())
		}
		vmDriver = docker
	default:
		klog.Fatalf("The driver '%s' is not supported.", driverName)
	}

    ...
	controller := controller.New(clientset, informer, vmDriver)
    ...
}
```
> `...` 表示不更改內容。完整程式請參考 [main.co](https://github.com/cloud-native-taiwan/controller101/blob/master/cmd/main.go)

### 執行
當上述功能實現後，且已有新增完 VirtualMachine CRD 的 Kubernetes 環境時，就可以執行以下指令來啟動控制器:

```sh
$ eval $(minikube docker-env)
$ go run cmd/main.go --kubeconfig=$HOME/.kube/config \
    -v=3 --logtostderr \
    --leader-elect=false \
    --vm-driver=docker
...
I1015 16:02:57.180484   62884 controller.go:77] Starting the controller
I1015 16:02:57.180665   62884 controller.go:78] Waiting for the informer caches to sync
I1015 16:02:57.285693   62884 controller.go:86] Started workers
```

接著開啟另一個 Terminal 來建立 VirtualMachine 資源實例。當建立時，會發現控制器更新了 test-vm 資源實例，這時可以利用 kubectl 查看狀態:

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

$ kubectl get vms
NAME      STATUS   CPU   MEMORY                AGE
test-vm   Active   0     0.10977787071142493   44s
```

由於本範例使用 Docker 作為虛擬機驅動程式，因此該資源實際上是建立一個容器。我們可以利用 docker 指令來查看:

```sh
$ docker ps --filter "name=test-vm"
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
b0d7f2be48e5        nginx:1.17.4        "nginx -g 'daemon of…"   8 seconds ago       Up 6 seconds        80/tcp              test-vm

$ docker inspect test-vm -f "{{.HostConfig.Memory}}"
4000000000

$ docker inspect test-vm -f "{{.HostConfig.NanoCpus}}"
2
```

接著我們來增加這個 NIGNX 的工作負載，以查看 CPU 變化:

```sh
$ IP=$(docker inspect test-vm -f "{{.NetworkSettings.IPAddress}}")
$ docker run --rm -it busybox /bin/sh -c "while :; do wget -O- ${IP}; done"
```

開啟新 Terminal 以 kubectl 指令來查看:

```
$ kubectl get vms -w
NAME      STATUS   CPU   MEMORY                AGE
test-vm   Active   0     0.11279374628042013   5m30s
test-vm   Active   15.637706179775282   0.11279374628042013   5m43s
test-vm   Active   15.688157325581395   0.11299480465168647   6m13s
test-vm   Active   15.55665426966292    0.11279374628042013   6m43s
```

> 由於控制器設計關析，CPU 與 Memory 只會每 30s 同步一次。

![](https://i.imgur.com/lWMxYl7.png)

## 結語
今天將控制器管理 VirtualMachine 資源實例的邏輯完成。一但完成，就能利用 Kubernetes-like API 來管理虛擬機的生命週期，或取得虛擬機狀態等等事情。然而今天實作部分，事實上還有一些問題存在，比如說我們先把自定義控制器暫時關閉，然後執行`kubectl delete vm test-vm`指令來將該資源實例從 Kubernetes 中刪除，這時查看虛擬機列表(因為 VM Driver 為 Docker，因此對應查看為 `docker ps`)時，就會發現被管理的虛擬機依然存在，並且當重新啟動控制器時，也會因為該資源實例已經被刪除，因此無法讓控制器來協助刪除，這樣就會形成殭屍虛擬機問題。如下圖所示。

![](https://i.imgur.com/6qecfOu.png)

> 從 Controller 程式碼中也可以從`deleteObject()`看出問題，因為這邊若發生刪除錯誤，就會造成外部資源變成殭屍(或孤兒)。

那麼當遇到這個問題時，該怎麼解決呢?明天我們將針對這部份來實作，以確保 API 資源實例一定要先刪除所管理的虛擬機後，才能從 Kubernetes 叢集中刪除。

## Reference
- https://godoc.org/github.com/docker/docker/client
- https://itnext.io/how-to-create-a-kubernetes-custom-controller-using-client-go-f36a7a7536cc