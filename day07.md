---
title: "動手嘗試 Kubernetes 叢集更新吧"
date: 2019-9-22
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
許多人都知道，讓應用程式保持較新的狀態，以優化安全性與效能是一種好習慣，這點套用在 Kubernetes 元件升級上也適用，因為 Kubernetes 每三個月左右就會有新的功能發布或改變，且隨著 Kubernetes 盛行，越來越多安全漏洞被揭露後，升級叢集慢慢變成一件重要議題。但 Kubernetes 並不像升級叢集中的應用程式那麼簡單，因為需要考慮各層面問題，就如昨天提到的內容一樣，要針對每一種可能發生狀況先有個認知，以盡可能在發生狀況時，能夠化險為夷。很慶幸的是，這方面越來越受到重視，有許多 Kubernetes 部署工具開始試圖引入一些機制與流程來簡化更新叢集的過程，雖然這並不能解決所有會發生問題，但至少我們可以善用工具來增加更新的正確性與成功率。

<!--more-->

今天我們將實際利用 Kubeadm 工具來更新叢集，這邊沿用之前分享時，部署的 HA 叢集來進行 `v1.15.4 更新至 v1.16.0`。

## 利用 kubeadm 更新叢集
本部分將依據以下步驟進行，並透過 Kubeadm 升級既有 Kubernetes 叢集至新版本:

* **主節點(Masters)**
    * 更新 kube-apiserver, controller manager, scheduler 與 etcd。
    * 更新 Addons。如: kube-proxy, CoreDNS。
    * 更新 kubelet binary file 與組態檔案。
    * (optional)更新 Node bootstrap tokens 的 RBAC 規則。
* **工作節點(Nodes)**
    * 在主節點使用 kubectl drain 來驅趕 Pods 到其他節點，並進入維運模式。另外建議使用 [PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/) 確保應用程式在 Kubernetes 叢集的可用與不可用數。
    * 更新 kubelet 二進制檔與組態檔案
    * 在主節點使用 kubectl uncordon 讓節點能夠被排程。

### 事前準備
在開始更新叢集前，請確保以下條件已達成:

* 用 kubeadm 建立一座 Kubernetes 叢集。可以參考[實現 Kubernetes 高可靠架構部署](https://k2r2bai.com/2019/09/20/ironman2020/day05/)文章進行。
* 確保叢集的所有節點處於 Ready 狀態。
* 確保應用程式利用進階的 Kubernetes API 建立，如 Deployment。並利用多副本機制來避免服務中斷。

> 以下步驟請一台一台來更新，這是為了確保 Kubernetes 功能不會因為更新而中斷。

### 更新主節點(Masters)
依據步驟建議，我們需要先升級主節點，再接著進行工作節點升級。首先需要更新`所有主節點`的 kubeadm 工具版本:

```
$ apt-mark unhold kubeadm
$ apt-get update && apt-get install -y kubeadm=1.16.0-00 && \
  apt-mark hold kubeadm

$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:34:01Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

安裝完成後，在`第一台主節點`執行以下指令來更新控制平面元件:

```sh
$ kubeadm upgrade plan v1.16.0
...
COMPONENT            CURRENT   AVAILABLE
API Server           v1.15.4   v1.16.0
Controller Manager   v1.15.4   v1.16.0
Scheduler            v1.15.4   v1.16.0
Kube Proxy           v1.15.4   v1.16.0
CoreDNS              1.3.1     1.6.2
Etcd                 3.3.10    3.3.15-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.16.0

# 看到上面訊息後，即可執行
$ kubeadm upgrade apply v1.16.0
...
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.16.0". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

接著在`其他主節點`執行以下指令來更新控制平面元件:

```sh
$ kubeadm upgrade node
...
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.16" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
```

當一個主節點完成後，接著需要更新 kubelet 與 kubectl 元件:

```sh
$ apt-mark unhold kubelet kubectl
$ apt-get update && apt-get install -y kubelet=1.16.0-00 kubectl=1.16.0-00 && \
  apt-mark hold kubelet kubectl
```

重新啟動 kubelet 來更新叢集資訊:

```sh
$ systemctl restart kubelet
```

最後利用 kubectl 來檢查叢集狀態:

```sh
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:36:53Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:27:17Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}

$ kubectl get no
NAME     STATUS   ROLES    AGE     VERSION
k8s-g1   Ready    <none>   2d23h   v1.15.4
k8s-g2   Ready    <none>   2d23h   v1.15.4
k8s-m1   Ready    master   3d      v1.16.0
k8s-m2   Ready    master   2d23h   v1.16.0
k8s-m3   Ready    master   2d23h   v1.16.0
k8s-n1   Ready    <none>   2d23h   v1.15.4
k8s-n2   Ready    <none>   2d23h   v1.15.4
```

### 更新節點(Nodes)
當所有主節點都更新完成後，即可進行更新 Nodes。如同主節點一樣，首先需要更新 kubeadm 工具版本:

```
$ apt-mark unhold kubeadm
$ apt-get update && apt-get install -y kubeadm=1.16.0-00 && \
  apt-mark hold kubeadm

$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:34:01Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

接著在任一台`主節點`上執行 kubectl drain 指令，將要更新節點的 Pod 轉移到其他節點，並進入  Unschedulable 狀態:

```sh
$ kubectl drain <node> --ignore-daemonsets
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-clcq6, kube-system/kube-proxy-sgcl2
evicting pod "coredns-bf7759867-jkhf4"
pod/coredns-bf7759867-jkhf4 evicted
node/k8s-n1 evicted
```

在更新元件之前，先更新相關組態檔:

```sh
$ kubeadm upgrade node
...
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
```

一但新版本組態檔案正確更新後，即可更新 kubelet:

```sh
$ apt-mark unhold kubelet
$ apt-get update && apt-get install -y kubelet=1.16.0-00 && \
  apt-mark hold kubelet
```

重新啟動 kubelet 來更新叢集資訊:

```sh
$ systemctl restart kubelet
```

完成後，進入任一台主節點執行以下指令來恢復節點至可排程狀態:

```sh
$ kubectl uncordon <node>
```

### 驗證
最後主節點與節點完成後，即可進入任一台主節點透過 kubectl 來查看狀態:

```sh
$ kubectl get no
NAME     STATUS   ROLES    AGE     VERSION
k8s-g1   Ready    <none>   2d23h   v1.16.0
k8s-g2   Ready    <none>   2d23h   v1.16.0
k8s-m1   Ready    master   3d      v1.16.0
k8s-m2   Ready    master   2d23h   v1.16.0
k8s-m3   Ready    master   2d23h   v1.16.0
k8s-n1   Ready    <none>   2d23h   v1.16.0
k8s-n2   Ready    <none>   2d23h   v1.16.0

$ kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
```

## 結語
今天簡單透過 kubeadm 來實現更新叢集元件。過程中，可以發現 kubeadm 幫助我們簡化了許多流程，讓我們不需要再手動完成太多事情，但是為了徹底知道怎麼運作過程，明天我們將對 kubeadm 的更新步驟進行分析與說明。

## Reference
- https://kubernetes.io/docs/setup/release/version-skew-policy/
- https://v1-15.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade-1-15/
- https://medium.com/@fairwinds/the-reactiveops-bestest-kubernetes-cluster-upgrade-f7a7589b21fb