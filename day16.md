---
title: "利用 Device Plugins 提供硬體加速"
date: 2019-10-01
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
[Device Plugins](https://kubernetes.io/docs/concepts/cluster-administration/device-plugins/) 是 Kubernetes v1.8 版本開始加入的 Alpha 功能，目標是結合 Extended Resource 來支援 GPU、FPGA、高效能 NIC、InfiniBand 等硬體設備介接的插件，這樣好處在於硬體供應商不需要修改 Kubernetes 核心程式，只需要依據 [Device Plugins 介面](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-management/device-plugin.md)來實作特定硬體設備插件，就能夠提供給 Kubernetes Pod 使用。而本篇會稍微提及 Device Plugin 原理，並說明如何使用 NVIDIA device plugin。

P.S. 傳統的`alpha.kubernetes.io/nvidia-gpu`將於 1.11 版本移除，因此與 GPU 相關的排程與部署原始碼都將從 Kubernetes 核心移除。

<!--more-->

## Device Plugins 原理
Device  Plugins 主要提供了一個 [gRPC 介面](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-management/device-plugin.md)來給廠商實現`ListAndWatch()`與`Allocate()`等 gRPC 方法，並監聽節點的`/var/lib/kubelet/device-plugins/`目錄中的 gRPC Server Unix Socket，這邊可以參考官方文件 [Device Plugins](https://kubernetes.io/docs/concepts/cluster-administration/device-plugins/)。一旦啟動 Device Plugins 時，透過 Kubelet Unix Socket 註冊，並提供該 plugin 的 Unix Socket 名稱、API 版本號與插件資源名稱(vendor-domain/resource，例如 nvidia.com/gpu)，接著 Kubelet 會將這些曝露到 Node 狀態以便 Scheduler 使用。

Unix Socket 範例：
```sh
$ ls /var/lib/kubelet/device-plugins/
kubelet_internal_checkpoint  kubelet.sock  nvidia.sock
```

一些 Device Plugins 列表：
- [NVIDIA GPU](https://github.com/NVIDIA/k8s-device-plugin)
- [RDMA](https://github.com/hustcat/k8s-rdma-device-plugin)
- [Kubevirt](https://github.com/kubevirt/kubernetes-device-plugins)
- [SFC](https://github.com/vikaschoudhary16/sfc-device-plugin)
- [Intel Device Plugins](https://github.com/intel/intel-device-plugins-for-kubernetes)
- [SR-IOV](https://github.com/intel/sriov-network-device-plugin)

## 節點資訊
部署沿用之前文章建置的 HA 環境進行測試，全部都採用裸機部署，作業系統為`Ubuntu 18.04+`:

| IP Address  | Hostname     | CPU | Memory | Role | Extra Device |
|-------------|--------------|-----|--------|------|--------------|
|172.22.132.11| k8s-m1       | 4   | 16G    |Master| None         |
|172.22.132.12| k8s-m2       | 4   | 16G    |Master| None         |
|172.22.132.13| k8s-m3       | 4   | 16G    |Master| None         |
|172.22.132.21| k8s-n1       | 4   | 16G    |Node  | None         |
|172.22.132.22| k8s-n2       | 4   | 16G    |Node  | None         |
|172.22.132.32| k8s-g2       | 4   | 16G    |Node  | GTX 1060 3G *2|

## 事前準備
安裝 Device Plugin 前，需要確保以下條件達成：

* 所有節點需要安裝 [Docker](https://docs.docker.com/v17.09/engine/installation/#cloud)。

```sh
$ curl -fsSL "https://get.docker.com/" | sh
```

* GPU 節點需正確安裝指定版本的 NVIDIA Driver 與 CUDA。

```sh
$ wget http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_9.1.85-1_amd64.deb
$ sudo dpkg -i cuda-repo-ubuntu1604_9.1.85-1_amd64.deb
$ sudo apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub
$ sudo apt-get update 
$ sudo apt-get install -y linux-headers-$(uname -r)
$ sudo apt-get -o Dpkg::Options::="--force-overwrite" install -y cuda-10-0 cuda-drivers
```

* GPU 節點需正確安裝指定版本的 [NVIDIA Docker 2](https://github.com/NVIDIA/nvidia-docker)。

```sh
$ distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
$ curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
$ curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

$ sudo apt-get update && sudo apt-get install -y nvidia-docker2
$ sudo systemctl restart docker
```

* 部署一座 Kubernetes v1.10+ 叢集。請參考 [kubeadm 部署 Kubernetes 叢集](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)。

## 安裝 NVIDIA Device Plugin
若上述要求以符合，再開始前需要在`每台 GPU worker 節點`修改`/lib/systemd/system/docker.service`檔案，將 Docker default runtime 改成 nvidia，依照以下內容來修改:
```sh
...
ExecStart=/usr/bin/dockerd -H fd:// --default-runtime=nvidia
...
```
> 這邊也可以修改`/etc/docker/daemon.json`檔案，請參考 [Configure and troubleshoot the Docker daemon](https://docs.docker.com/config/daemon/)。

完成後儲存，並重新啟動 Docker：
```sh
$ sudo systemctl daemon-reload && sudo systemctl restart docker
```

確認上述完成，接著在主節點透過 kubectl 來部署 NVIDIA Device Plugins:
```sh
$ kubectl create -f https://gist.githubusercontent.com/kairen/bf967d566d35edda381edb9ba8659f7b/raw/ccc18711bf016d5b836280226785c1ad0282c035/nvidia-device-plugin.yml
daemonset "nvidia-device-plugin-daemonset" created

$ kubectl -n kube-system get po -o wide
NAME                                       READY     STATUS    RESTARTS   AGE       IP               NODE
...
nvidia-device-plugin-daemonset-nwx2s       1/1     Running   0          49s    10.244.255.80   k8s-g2   <none>           <none>
```

> 由於目前 NVIDIA Device Plugin 的 beta3 有問題，因此以 beat1 為主。

## 測試 GPU
首先執行以下指令確認是否可被分配資源:

```sh
$ kubectl get nodes "-o=custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu"
NAME     GPU
k8s-g2   2
...
```

當 NVIDIA Device Plugins 部署完成後，即可建立一個簡單範例來進行測試:

```sh
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  restartPolicy: Never
  containers:
  - image: nvidia/cuda
    name: cuda
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1
EOF
pod "gpu-pod" created

$ kubectl get po -o wide
NAME                     READY   STATUS      RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
gpu-pod                  0/1     Completed   0          21m     10.244.255.81   k8s-g2   <none>           <none>

$ kubectl logs gpu-pod
Sat Oct 01 15:28:38 2019
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 418.87.01    Driver Version: 418.87.01    CUDA Version: 10.1     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 106...  Off  | 00000000:05:00.0 Off |                  N/A |
|  0%   38C    P8     6W / 120W |      0MiB /  3019MiB |      1%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

從上面結果可以看到 Kubernetes Pod 正確的使用到 NVIDIA GPU。

## 結語
Kubernetes 提供了 Device Plugin Interface 來讓硬體供應商實現自家硬體與 Kubernetes 整合的功能，這使 Kubernetes 社區不在需要維護各種廠商的硬體整合程式，以減少核心程式碼的複雜性，一方面能更加專注在規範 Device Plugin 標準的事情。

## Reference
- https://medium.com/@maniac.tw/ubuntu-18-04-%E5%AE%89%E8%A3%9D-nvidia-driver-418-cuda-10-tensorflow-1-13-a4f1c71dd8e5
- https://www.mvps.net/docs/install-nvidia-drivers-ubuntu-18-04-lts-bionic-beaver-linux/
- https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-management/device-plugin.md