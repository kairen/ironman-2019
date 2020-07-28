---
title: "動手實作 Kubernetes 自定義控制器 Part5"
date: 2019-10-14
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
在[動手實作 Kubernetes 自定義控制器 Part4](https://k2r2bai.com/2019/10/13/ironman2020/day28/) 文章結語部分，我有提到目前實作的自定義控制器還存在著問題(如下圖)，其中就是自定義資源 VirtualMachine 的實例被刪除前，未正確透過 VM Driver 刪除實際管理的虛擬機，這樣情況下的虛擬機都會變成失去控制器管理的殭屍(或孤兒)。基於此問題，今天將說明該如何修改程式以解決這樣問題。

<!--more-->

在開始實作前，我們先來探討一些概念。在接觸 Kubernetes 時，相信大家都玩過 Deployment、Job 與 DaemonSet 等功能，這些功能有個共同點，那就是都管理著一個或多個 Pod 的生命週期，這表示當一個實例(比如 Deployment)被執行刪除時，其相關聯的 Pod 都會接著被刪除。而這種機制正是 Kubernetes [垃圾收集器](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/)，在 v1.6+ 開始，Kubernetes 會自動對一些 API 資源物件(如 Deployment、ReplicaSet)引入`ownerReferences`欄位，這個欄位用來標示相依 API 資源物件的 Owner 是誰，而自己則為 Owner 的 Dependent，因此當 Owner 被刪除時，所有關聯的 API 資源物件就會被垃圾收集器回收(從叢集中刪除)，而這過程又稱`級聯刪除(Cascading deletion)`。

> 雖然命名為垃圾收集器，但實際上它也是以控制器模式(Controller Pattern)的形式實作。

```sh
$ kubectl run nginx --image nginx --port 80
$ kubectl get po nginx-7c45b84548-gj999 -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2019-10-14T14:43:07Z"
  generateName: nginx-7c45b84548-
  labels:
    pod-template-hash: 7c45b84548
    run: nginx
  name: nginx-7c45b84548-gj999
  namespace: default
  ownerReferences: # Deployment 管理著 ReplicaSet，而 ReplicaSet 管理著 Pod
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: nginx-7c45b84548
    uid: 426961f6-b94d-49d2-a110-db17b3c50008
```

> 值得一提的是，`ownerReferences`是可以手動設定與修改的。比如說上述指令建立了一個 NGINX Pod，這時我們用 kubectl 刪除 Pod 的`ownerReferences`，就可以讓 Kubernetes 垃圾收集器無法處理到該 Pod。只是這個 Pod 也變成殭屍(或孤兒)，不會因為 Deployment 刪除而被殺掉，因此必須手動殺掉才能。

那麼這樣的方式是否可以用來解決我們遇到的問題呢?

不幸的是，Kubernetes 垃圾收集器僅能用於刪除 Kubernetes API 資源，因此無法讓我們達到 VirtualMachine 資源實例被刪除前，確保所關聯的虛擬機已被刪除。那這樣該如何實現呢?

事實上 Kubernetes 也考慮到這樣問題，因此對於 API 資源的級聯刪除提供了兩種模式:

* **Background**:在這模式下，Kubernetes 會直接刪除 Owner 資源物件，然後再由垃圾收集器在後台刪除相關的 API 資源物件。
* **Foreground**:在這模式下，Owner 資源物件會透過設定`metadta.deletionTimestamp`欄位來表示『正在刪除中』。這時 Owner 資源物件依然存在於叢集中，並且能透過 REST API 查看到相關資訊。該資源被刪除條件是當移除了`metadata.finalizers`欄位後，才會真正的從叢集中移除。這樣機制形成了預刪除掛鉤(Pre-delete hook)，因此我們能在正在刪除的期間，開始回收相關的資源(如虛擬機或其他 Kubernetes API 資源等等)，當回收完後，再將該物件刪除。

其中 Foreground 模式能透過 Kubernetes Finalizers 機制與 OwnerReferences 機制完成。不過 OwnerReferences 只能用於內部 API 資源物件，當想要處理外部資源時，就必須利用 Finalizers 來達成。

而 Finalizers 機制只需要在 API 資源物件中的`metadata.finalizers`欄位塞入一個字串值即可，比如說以下範例:

```sh
$ cat <<EOF | kubectl apply -f -
apiVersion: cloudnative.tw/v1alpha1
kind: VirtualMachine
metadata:
  name: test-vm-finalizer
  finalizers:
  - finalizer.cloudnative.tw
spec:
  resource:
    cpu: 2
    memory: 4G
EOF
virtualmachine.cloudnative.tw/test-vm-finalizer created
```

當建立時，接著透過 kubectl 來刪除這個資源實例:

```sh
$ kubectl delete vms test-vm-finalizer
virtualmachine.cloudnative.tw "test-vm-finalizer" deleted
```

這時會發現 kubectl 卡在刪除指令，且不管怎麼執行都無法刪除。因為這樣情況，我們開啟另一個 Terminal 查看後，發現資源物件依然存在，但`metadata.deletionTimestamp`被下了時間，這表示該資源已經處於預刪除階段:

```sh
$ kubectl get vms test-vm-finalizer -o yaml
apiVersion: cloudnative.tw/v1alpha1
kind: VirtualMachine
metadata:
  ...
  deletionGracePeriodSeconds: 0
  deletionTimestamp: "2019-10-14T16:28:58Z"
  finalizers:
  - finalizer.cloudnative.tw
  name: test-vm-finalizer
```

那麼該怎麼讓這個資源物件被刪除呢? 我們只要透過`kubectl edit`把`metadata.finalizers`欄位拔掉即可。不過這樣做法都是透過 kubectl 來達成，那麼我們該如何在自定義控制器程式中實現呢? 接下來將針對這部份進行說明。

## 使用 Finalizer
本部分將修改`controller.go`程式，以加入 Finalizers 機制來確保虛擬機被正確刪除。

### 環境準備
由於使用這個功能需要用到 Kubernetes 與 Go 語言，因此需要透過以下來完成條件:

* 一座 Kubernetes v1.10+ 叢集。透過 [Minikube](https://github.com/kubernetes/minikube) 建立即可 `minikube start --kubernetes-version=v1.15.4`。
* 一個 Docker 環境，可以直接 Minikube 執行`eval $(minikube docker-env)`來取的 Docker 參數，並遠端操作。
* 安裝 Go 語言 v1.11+ 開發環境，由於開發中會使用到 Go mod 來管理第三方套件，因此必須符合支援版本。安裝請參考 [Go Getting Started](https://golang.org/doc/install)。

### Implementation
從前言的過程中，可以發現 Finalizers 能在`metadata.finalizers`欄位手動加入來實現預刪除掛鉤。而在程式的實作中，要加入 Finalizers 機制以避免虛擬機變成殭屍(或孤兒)並不難，只要在 [controller.go](https://github.com/cloud-native-taiwan/controller101/blob/master/pkg/controller/controller.go) 的`createServer()`函式中，對 VirtualMachine 物件設定`metadata.finalizers`即可，只不過加入前提需要確保虛擬機已被正確建立，且 VirtualMachine 一定會進入 Active 狀態下進行。

```go
func (c *Controller) createServer(vm *v1alpha1.VirtualMachine) error {
	vmCopy := vm.DeepCopy()
	ok, _ := c.vm.IsServerExist(vm.Name)
	if !ok {
		...
		addFinalizer(&vmCopy.ObjectMeta, finalizerName)
		if err := c.updateStatus(vmCopy, v1alpha1.VirtualMachineActive, nil); err != nil {
			return err
		}
	}
	return nil
}
```

> * `...` 表示不更改內容。完整程式請參考 [controller.go L187-L214](https://github.com/cloud-native-taiwan/controller101/blob/34bcecb2ae43e3eb9a981da17c76e21f001f06b0/pkg/controller/controller.go#L187-L214)
> * 其中`finalizerName`被定義在成 const 變數，其內容為`finalizer.cloudnative.tw`。
> * 而`addFinalizer()`函式而在 [util.go](https://github.com/cloud-native-taiwan/controller101/blob/master/pkg/controller/util.go) 中被實作。基本上就是傳入 API 物件的 ObjectMeta(metadata) 與 Finalizer 名稱來設定。

修改完成後，當控制器依據 VirtualMachine 資源實例正確建立虛擬機時，就會自動塞入 Finalizers。而當擁有 Finalizers 的資源實例被執行刪除時，Kubernetes API Server 會透過 Update 操作修改`metadata.deletionTimestamp`欄位，但不會執行 Delete 操作，因此 Informer 實際上收到會是 Update 事件。基於此改變，我們必須在 controller.go 中修改一些流程與內容。

```go
func (c *Controller) syncHandler(key string) error {
    ...

    switch vm.Status.Phase {
	...
	case v1alpha1.VirtualMachineActive:
		if !vm.ObjectMeta.DeletionTimestamp.IsZero() {
			if err := c.makeTerminatingPhase(vm); err != nil {
				return err
			}
			return nil
		}

		if err := c.updateUsage(vm); err != nil {
			return err
		}
	case v1alpha1.VirtualMachineTerminating:
		if err := c.deleteServer(vm); err != nil {
			return err
		}
	}
}
```

> `...` 表示不更改內容。完整程式請參考 [controller.go](https://github.com/cloud-native-taiwan/controller101/blob/master/pkg/controller/controller.go)。

當 VirtualMachine 實例 Active，且被執行刪除時，可以透過判斷`metadata.deletionTimestamp`來確認是否進入預刪除階段，若是的話，則將 VirtualMachine 資源實例更新成 Terminating 階段，若不是的話，則持續更新狀態。其中`makeTerminatingPhase()`的程式實現如下:

```go
func (c *Controller) makeTerminatingPhase(vm *v1alpha1.VirtualMachine) error {
	vmCopy := vm.DeepCopy()
	return c.updateStatus(vmCopy, v1alpha1.VirtualMachineTerminating, nil)
}
```

接著當控制器接收到一個處於`Terminating`的資源物件時，就會執行`deleteServer()`來刪除虛擬機，並且直到刪除成功後，才將 Finalizers 從資源實例中移除。一但被 Finalizers 時，Kubernetes 就會在經過`deletionGracePeriodSeconds`設定的秒數後，將該資源實例從叢集中刪除。`deleteServer()`的程式實現如下:

```go
func (c *Controller) deleteServer(vm *v1alpha1.VirtualMachine) error {
	vmCopy := vm.DeepCopy()
	if err := c.vm.DeleteServer(vmCopy.Name); err != nil {
		// Requeuing object to workqueue for retrying
		return err
	}

	removeFinalizer(&vmCopy.ObjectMeta, finalizerName)
	if err := c.updateStatus(vmCopy, v1alpha1.VirtualMachineTerminating, nil); err != nil {
		return err
	}
	return nil
}
```

最後由於採用 Finalizers 機制，因此 Informer 並不會觸發`DeleteFunc`對應的`deleteObject()`函式，因此我們可以在 Controller 的建構子中註解掉。

```go
func New(clientset cloudnative.Interface, informer cloudnativeinformer.SharedInformerFactory, vm driver.Interface) *Controller {
	...
	vmInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: controller.enqueue,
		UpdateFunc: func(old, new interface{}) {
			controller.enqueue(new)
		},
		// DeleteFunc: controller.deleteObject,
	})
	return controller
}
```

> `...` 表示不更改內容。完整程式請參考 [controller.go](https://github.com/cloud-native-taiwan/controller101/blob/master/pkg/controller/controller.go)

### Running
當上述功能實現後，且已有新增完 VirtualMachine CRD 的 Kubernetes 環境時，就能執行以下指令啟動控制器:

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

接著開啟另一個 Terminal 來建立 VirtualMachine 資源實例。當建立時，會發現控制器更新了 test-vm 資源實例，這時可以利用 kubectl 與 docker 查看狀態:

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

$ docker ps --filter "name=test-vm"
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
347f8626f36a        nginx:1.17.4        "nginx -g 'daemon of…"   4 seconds ago       Up 3 seconds        80/tcp              test-vm
```

接著我們利用 kubectl 觀察 test-vm 的`metadata`變化:

```sh
$ kubectl get vms test-vm -o=jsonpath='{.metadata.finalizers}'
[finalizer.cloudnative.tw]
```

而當執行`kubectl delete vm test-vm`時，就會發現這樣變化:

```
$ kubectl get vms -w
NAME      STATUS   CPU   MEMORY                AGE
test-vm   Active   0     0.10595776165736437   4m4s
test-vm   Terminating   0     0.10595776165736437   4m6s
```

這時查看 API 資源與 Container 時，都會被正確移除。另外也可以嘗試把控制器暫時關閉，並執行刪除一個 VirtualMachine 資源實例的操作，這時會看到該操作卡在刪除指令下，並且資源實例還存在於叢集中。而當重新啟動控制器時，才會停止這樣狀況。

## 結語
今天簡單認識 Kubernetes 的垃圾收集器與 Finalizers 的機制，並且在自定義控制器實作 Finalizers 來確保外部資源能夠在 Kubernetes 內部關聯的 API 資源被刪除時，優先被回收。

到這邊一個自定義控制器大致上已完成，而接下來我們將說明如何讓控制器部署到 Kubernetes 叢集中，並且能夠實現哪些功能來加強這個控制器(Expose Metrics、Admission Controller 與 Fake client testing)。

> 由於鐵人賽文章不夠用，因此之後都會以 [KaiRen's Blog](http://k2r2bai.com) 來新增這些內容。

## Reference
- https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/
- https://book.kubebuilder.io/reference/using-finalizers.html
- https://draveness.me/kubernetes-garbage-collector
- https://blog.openshift.com/garbage-collection-custom-resources-available-kubernetes-1-8/