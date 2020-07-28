---
title: "實現 Kubernetes 高可靠架構部署"
date: 2019-9-20
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
隨著團隊越來越多地在生產環境使用 Kubernetes 管理雲原生應用程式，我們必須考量在各種故障下，Kubernetes 能正常運行的情況，比如說:在流量高峰期間，將工作負載分散到更多節點或轉往公有雲、跨多個 Availability Zones/Regions 部署、建構高可靠(Highly Available，HA)架構等等要求。其中高可靠架構在昨天的[淺談 Kubernetes 高可靠架構](https://k2r2bai.com/2019/09/19/ironman2020/day04/)文章中，簡單地複習了高可靠架構。而今天將說明如何實現與利用 kubeadm 建立一座架構大致如下圖所示的 HA 叢集。

![](https://i.imgur.com/Bj2a2MW.png)

<!--more-->

在開始建立前，我們先簡單瞭解每個主節點要執行元件，以及這些元件如何完成高可靠架構。

* **etcd**: 透過多節點的 etcd 實例組成叢集，並利用 [Raft](https://raft.github.io/) 演算法，來選取一個領導者(Leader)處理需要叢集共識的所有客戶端的請求(Request)，如下圖所示。另外由於 Raft 演算法關析，還需要注意叢集的故障容許度(Failure Tolerance)。

| Cluster Size | Majority | Failure Tolerance |
|:-:|:-:|:-:|
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |
| 6 | 4 | 2 |
| 7 | 4 | 3 |

> 計算故障容許節點數為`(N/2)+1`，其中 N 為叢集大小。

![Quorum with etcd](https://i.imgur.com/LHjRbcx.png)
(圖片擷取自：[ Kubecon NA 2018 - Highly Available Kubernetes Clusters - Best Practices](https://static.sched.com/hosted_files/kccna18/8b/Highly%20Available%20Kubernetes%20Clusters%20-%20Best%20Practices%20-%20Kubecon%20NA%202018.pdf))

* **API server:**: 每個 API server 會與本地端的 etcd 溝通，並接收來至客戶端與其他元件的 API 請求。由於 API server 屬於 Active-Active 架構，因此每個 API server 在叢集中都處於可用狀態。

![Active-Active for API server](https://i.imgur.com/SbhdAd3.png)
(圖片擷取自：[ Kubecon NA 2018 - Highly Available Kubernetes Clusters - Best Practices](https://static.sched.com/hosted_files/kccna18/8b/Highly%20Available%20Kubernetes%20Clusters%20-%20Best%20Practices%20-%20Kubecon%20NA%202018.pdf))

* **controllers, scheduler**: 這些元件採用 [Lease](https://en.wikipedia.org/wiki/Lease_(computer_science)) 機制來從所有實例中選取一個作為領導者，並由領導者處理監聽對應的 API 資源來完成功能，因此整個叢集只會有一個擁有完整功能，除非原本領導的節點發生故障，才會尤其它接手。

![Active-Passive for controllers and scheduler](https://i.imgur.com/KCaiB8o.png)
(圖片擷取自：[ Kubecon NA 2018 - Highly Available Kubernetes Clusters - Best Practices](https://static.sched.com/hosted_files/kccna18/8b/Highly%20Available%20Kubernetes%20Clusters%20-%20Best%20Practices%20-%20Kubecon%20NA%202018.pdf))

* **kubelet**: 由於 kubelet 只能設定跟一個 API server 的端點(Endpoint)，但為了達到某個 API server 故障時，還能夠繼續正常執行的需求，我們需要提供一個虛擬 IP(VIP, Virtual IP)，以及負載平衡器(Load Balancer)來讓 kubelet 能夠存取多個 API server，一方面利用負載平衡器的機制來分散工作負載到所有 API server 上。

另外由於 API servers 需要提供 VIP 與負載平衡器，因此必須在所有主節點上額外安裝以下元件來達到需求。

* **Keepalived**: 基於 [VRRP](https://en.wikipedia.org/wiki/Virtual_Router_Redundancy_Protocol) 協定來實現高可靠架構，所有主節點會基於此元件舉出一個 VIP 來作為存取 API 的端點，這主要是確保其他元件連接 API 時，不會因為某個主節點中斷而無法存取。
* **HAProxy**: 與 API server 一樣，為 Active-Active 架構，因此每個主節點都可以存取作為 Proxy 的 IP 與 Port。

簡單了解完實現方式後，下一小節將說明如何利用 kubeadm 來建構 Kubernetes HA 叢集。

> 選用 kubeadm 是因為方便手動做測試，且 kubeadm HA 功能在 v1.15 版本進入了 Beta 階段，因此值得大家嘗試看看。當然過程中，若節點數過多的話，建議搭配 Ansible(or Puppet, SaltStack) 這類工具進行。

## Set up HA cluster using kubeadm
本部分將透過 [Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/) 來部署 Kubernetes v1.15 版本的 High Availability 叢集，而本安裝主要是參考官方文件中的 [Creating Highly Available Clusters with kubeadm](https://kubernetes.io/docs/setup/independent/high-availability/) 內容來進行，這邊將透過 HAProxy 與 Keepalived 的結合來實現控制面的 Load Balancer 與 VIP。

Kubernetes 部署的版本資訊：

* kubeadm: v1.15.4
* Kubernetes: v1.15.4
* CNI: v0.7.5
* etcd: v3.2.18
* Docker CE: 19.03.2
* Calico: v3.8

Kubernetes 部署的網路資訊：

* **Cluster IP CIDR**: 10.244.0.0/16
* **Service Cluster IP CIDR**: 10.96.0.0/12
* **Service DNS IP**: 10.96.0.10
* **DNS DN**: cluster.local
* **Kubernetes API Virtual IP**: 172.22.132.10

### 節點資訊
本文採用以下節點數進行裸機部署，作業系統採用`Ubuntu 18.04+`進行測試:

| IP Address  | Hostname | CPU | Memory | Role |
|-------------|----------|-----|--------|------|
|172.22.132.11| k8s-m1   | 4   | 16G    |Master|
|172.22.132.12| k8s-m2   | 4   | 16G    |Master|
|172.22.132.13| k8s-m3   | 4   | 16G    |Master|
|172.22.132.21| k8s-n1   | 4   | 16G    |Node  |
|172.22.132.22| k8s-n2   | 4   | 16G    |Node  |
|172.22.132.31| k8s-g1   | 4   | 16G    |Node  |
|172.22.132.32| k8s-g2   | 4   | 16G    |Node  |

另外所有 Master 節點將透過 Keepalived 提供一個 Virtual IP `172.22.132.10` 作為使用。

> * 所有操作全部用`root`使用者進行，主要方便部署用。

### 事前準備
開始部署叢集前需先確保以下條件已達成：
* `所有節點`彼此網路互通，並且`k8s-m1` SSH 登入其他節點為 passwdless，由於過程中很多會在某台節點(`k8s-m1`)上以 SSH 複製與操作其他節點。
* 確認所有防火牆與 SELinux 已關閉。如 CentOS：

```sh
$ systemctl stop firewalld && systemctl disable firewalld
$ setenforce 0
$ vim /etc/selinux/config
SELINUX=disabled
```
> 關閉是為了方便安裝使用，若有需要防火牆可以參考 [Required ports](https://kubernetes.io/docs/tasks/tools/install-kubeadm/#check-required-ports) 來設定。

* `所有節點`需要安裝 Docker CE 版本的容器引擎：

```sh
$ curl -fsSL https://get.docker.com/ | sh
```

* 所有節點需要加入 APT Kubernetes package 來源：

```shell=
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
$ echo 'deb http://apt.kubernetes.io/ kubernetes-xenial main' | tee /etc/apt/sources.list.d/kubernetes.list
```

* `所有節點`需要設定以下系統參數。

```sh
$ cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

$ sysctl -p /etc/sysctl.d/k8s.conf
```
> 關於`bridge-nf-call-iptables`的啟用，主要取決於是否將容器連接到`Linux bridge`或使用其他一些機制(如 SDN vSwitch)。

* Kubernetes v1.8+ 要求關閉系統 Swap，請在`所有節點`利用以下指令關閉：

```sh
$ swapoff -a && sysctl -w vm.swappiness=0

# 不同機器會有差異
$ sed '/swap.img/d' -i /etc/fstab
```
> * 記得`/etc/fstab`也要註解掉`SWAP`掛載。
> * 關閉 Swap 是避免 Kubernetes Pod 不會因為使用到 Swap 而影響效能，另一方面可以讓 OOM Killer 正常運作。

### Kubernetes Master 建立
本節將說明如何部署與設定 Kubernetes Master 節點中的各元件。

在開始部署`master`節點元件前，請先安裝好 kubeadm、kubelet 等套件，並建立`/etc/kubernetes/manifests/`目錄存放 Static Pod 的 YAML 檔：
```sh
$ export KUBE_VERSION="1.15.4"
$ apt-get update && apt-get install -y kubelet=${KUBE_VERSION}-00 kubeadm=${KUBE_VERSION}-00 kubectl=${KUBE_VERSION}-00
$ apt-mark hold kubeadm kubectl kubelet
$ mkdir -p /etc/kubernetes/manifests/
```

完成後，依照下面小節完成部署。

#### HAProxy
本節將說明如何建立 HAProxy 來提供 Kubernetes API Server 的負載平衡。在所有`master`節點的`/etc/haproxy/`目錄：
```sh
$ mkdir -p /etc/haproxy/
```

接著在所有`master`節點新增`/etc/haproxy/haproxy.cfg`設定檔，並加入以下內容：
```sh
$ cat <<EOF > /etc/haproxy/haproxy.cfg
global
  log 127.0.0.1 local0
  log 127.0.0.1 local1 notice
  tune.ssl.default-dh-param 2048

defaults
  log global
  mode http
  option dontlognull
  timeout connect 5000ms
  timeout client 600000ms
  timeout server 600000ms

listen stats
    bind :9090
    mode http
    balance
    stats uri /haproxy_stats
    stats auth admin:admin123
    stats admin if TRUE

frontend kube-apiserver-https
   mode tcp
   bind :8443
   default_backend kube-apiserver-backend

backend kube-apiserver-backend
    mode tcp
    balance roundrobin
    stick-table type ip size 200k expire 30m
    stick on src
    server apiserver1 172.22.132.11:6443 check
    server apiserver2 172.22.132.12:6443 check
    server apiserver3 172.22.132.13:6443 check
EOF
```
> 這邊會綁定`8443`作為 API Server 的 Proxy。

接著在新增一個路徑為`/etc/kubernetes/manifests/haproxy.yaml`的 YAML 檔來提供 HAProxy 的 Static Pod 部署，其內容如下：
```sh
$ cat <<EOF > /etc/kubernetes/manifests/haproxy.yaml
kind: Pod
apiVersion: v1
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  labels:
    component: haproxy
    tier: control-plane
  name: kube-haproxy
  namespace: kube-system
spec:
  hostNetwork: true
  priorityClassName: system-cluster-critical
  containers:
  - name: kube-haproxy
    image: docker.io/haproxy:1.7-alpine
    resources:
      requests:
        cpu: 100m
    volumeMounts:
    - name: haproxy-cfg
      readOnly: true
      mountPath: /usr/local/etc/haproxy/haproxy.cfg
  volumes:
  - name: haproxy-cfg
    hostPath:
      path: /etc/haproxy/haproxy.cfg
      type: FileOrCreate
EOF
```

接下來將新增另一個 YAML 來提供部署 Keepalived。

#### Keepalived
本節將說明如何建立 Keepalived 來提供 Kubernetes API Server 的 VIP。在所有`master`節點新增一個路徑為`/etc/kubernetes/manifests/keepalived.yaml`的 YAML 檔來提供 HAProxy 的 Static Pod 部署，其內容如下：
```sh
$ cat <<EOF > /etc/kubernetes/manifests/keepalived.yaml
kind: Pod
apiVersion: v1
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  labels:
    component: keepalived
    tier: control-plane
  name: kube-keepalived
  namespace: kube-system
spec:
  hostNetwork: true
  priorityClassName: system-cluster-critical
  containers:
  - name: kube-keepalived
    image: docker.io/osixia/keepalived:2.0.17
    env:
    - name: KEEPALIVED_VIRTUAL_IPS
      value: 172.22.132.10
    - name: KEEPALIVED_INTERFACE
      value: enp3s0
    - name: KEEPALIVED_UNICAST_PEERS
      value: "#PYTHON2BASH:['172.22.132.11', '172.22.132.12', '172.22.132.13']"
    - name: KEEPALIVED_PASSWORD
      value: d0cker
    - name: KEEPALIVED_PRIORITY
      value: "100"
    - name: KEEPALIVED_ROUTER_ID
      value: "51"
    resources:
      requests:
        cpu: 100m
    securityContext:
      privileged: true
      capabilities:
        add:
        - NET_ADMIN
EOF
```
> * `KEEPALIVED_VIRTUAL_IPS`：Keepalived 提供的 VIPs。
> * `KEEPALIVED_INTERFACE`：VIPs 綁定的網卡。
> * `KEEPALIVED_UNICAST_PEERS`：其他 Keepalived 節點的單點傳播 IP。
> * `KEEPALIVED_PASSWORD`： Keepalived auth_type 的 Password。
> * `KEEPALIVED_PRIORITY`：指定了備援發生時，接手的介面之順序，數字越小，優先順序越高。這邊`k8s-m1`設為 100，其餘為`150`。
> * `KEEPALIVED_ROUTER_ID`：一組 Keepalived instance 的數字識別子。

#### First control plane node
首先在`k8s-m1`節點建立`kubeadm-config.yaml`的 Kubeadm Master Configuration 檔：
```sh
$ cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.15.4
controlPlaneEndpoint: "172.22.132.10:8443"
networking:
  podSubnet: "10.244.0.0/16"
EOF
```
> `controlPlaneEndpoint`填入 VIPs 與 bind port。

新增完後，透過 kubeadm 來初始化 control plane：
```sh
$ kubeadm init --config=kubeadm-config.yaml --upload-certs

...
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 172.22.132.10:8443 --token qawtjn.l0bpc3o12fef33t5 \
    --discovery-token-ca-cert-hash sha256:7310e2e34b47214eba2be7a44375ea588a1d59d3126ac11759853d59fa76fadc \
    --control-plane --certificate-key 6b9fbbac56a7af8576d8c7f98e44d5d78984c7331ca6d41a066d05c3d3795cc7

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.22.132.10:8443 --token qawtjn.l0bpc3o12fef33t5 \
    --discovery-token-ca-cert-hash sha256:7310e2e34b47214eba2be7a44375ea588a1d59d3126ac11759853d59fa76fadc
```
> 請記下來 join 節點資訊，方便後面使用。若忘記的話，可以用 `kubeadm token` 指令重新取得。

經過一段時間完成後，接著透過 netstat 檢查是否正常啟動服務：
```sh
$ netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 172.22.132.10:8443      0.0.0.0:*               LISTEN      11218/haproxy
tcp        0      0 0.0.0.0:9090            0.0.0.0:*               LISTEN      11218/haproxy
tcp        0      0 127.0.0.1:44551         0.0.0.0:*               LISTEN      9237/kubelet
tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      9237/kubelet
tcp        0      0 127.0.0.1:10249         0.0.0.0:*               LISTEN      11669/kube-proxy
tcp        0      0 172.22.132.11:2379      0.0.0.0:*               LISTEN      10367/etcd
tcp        0      0 172.22.132.11:2380      0.0.0.0:*               LISTEN      10367/etcd
tcp        0      0 127.0.0.1:10257         0.0.0.0:*               LISTEN      10460/kube-controll
tcp        0      0 127.0.0.1:10259         0.0.0.0:*               LISTEN      10615/kube-schedule
```

經過一段時間完成後，執行以下指令來使用 kubeconfig：
```sh
$ mkdir -p $HOME/.kube
$ cp -rp /etc/kubernetes/admin.conf $HOME/.kube/config
$ chown $(id -u):$(id -g) $HOME/.kube/config
```

透過 kubectl 檢查 Kubernetes 叢集狀況：
```sh
$ kubectl get no
NAME     STATUS     ROLES    AGE   VERSION
k8s-m1   NotReady   master   31s   v1.15.4

$ kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
```

接著部署 Calico CNI plugin:

```sh
$ wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml
$ sed -i 's/192.168.0.0\/16/10.244.0.0\/16/g' calico.yaml
$ kubectl apply -f calico.yaml
```

完成後，透過 kubectl 來查看 kube-system 的 Pod 建立狀況:

```sh
$ kubectl -n kube-system get po
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-65b8787765-ckd7b   1/1     Running   0          8m8s
calico-node-l9wh9                          1/1     Running   0          8m8s
coredns-5c98db65d4-89wq5                   1/1     Running   0          9m14s
coredns-5c98db65d4-lmvvn                   1/1     Running   0          9m14s
etcd-k8s-m1                                1/1     Running   0          8m24s
kube-apiserver-k8s-m1                      1/1     Running   0          8m17s
kube-controller-manager-k8s-m1             1/1     Running   0          8m26s
kube-haproxy-k8s-m1                        1/1     Running   0          9m30s
kube-keepalived-k8s-m1                     1/1     Running   0          8m13s
kube-proxy-g7clj                           1/1     Running   0          9m14s
kube-scheduler-k8s-m1                      1/1     Running   0          8m41s
```

到這邊`k8s-m1`就完成部署了，接著我們要將`k8s-m2`與`k8s-m3`以控制平面節點加入現有叢集。

#### Other control plane nodes
在 kubeadm v1.15 版本中，提供了自動配置 HA 的機制，因此只需要在其他主節點執行以下指令即可:

```sh
$ kubeadm join 172.22.132.10:8443 --token qawtjn.l0bpc3o12fef33t5 \
  --discovery-token-ca-cert-hash sha256:7310e2e34b47214eba2be7a44375ea588a1d59d3126ac11759853d59fa76fadc \
  --control-plane \
  --certificate-key 6b9fbbac56a7af8576d8c7f98e44d5d78984c7331ca6d41a066d05c3d3795cc7 \
  --ignore-preflight-errors=DirAvailable--etc-kubernetes-manifests
```

經過一段時間完成後，執行以下指令來使用 kubeconfig：
```sh
$ mkdir -p $HOME/.kube
$ cp -rp /etc/kubernetes/admin.conf $HOME/.kube/config
$ chown $(id -u):$(id -g) $HOME/.kube/config
```

透過 kubectl 檢查 Kubernetes 叢集狀況：
```sh
$ kubectl get node
NAME     STATUS   ROLES    AGE   VERSION
k8s-m1   Ready    master   10m   v1.15.4
k8s-m2   Ready    master   3m    v1.15.4
k8s-m3   Ready    master   74s   v1.15.4
```

> 由於其他主節點加入方式一樣，所以`k8s-m3`節點請重複執行本節過程來加入。

### Kubernetes Nodes 建立
本節將說明如何部署與設定 Kubernetes Node 節點中。在開始部署`node`節點元件前，請先安裝好 kubeadm、kubelet 等套件：
```sh
$ export KUBE_VERSION="1.15.4"
$ apt-get update && apt-get install -y kubelet=${KUBE_VERSION}-00 kubeadm=${KUBE_VERSION}-00
$ apt-mark hold kubeadm kubelet
```

安裝好後，在所有`node`節點透過 kubeadm 來加入節點：
```sh
$ kubeadm join 172.22.132.10:8443 \
  --token qawtjn.l0bpc3o12fef33t5 \
  --discovery-token-ca-cert-hash sha256:7310e2e34b47214eba2be7a44375ea588a1d59d3126ac11759853d59fa76fadc

...
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

### 測試部署結果
當節點都完成後，進入任一台`master`節點透過 kubectl 來檢查：
```sh
$ kubectl get no
NAME     STATUS   ROLES    AGE     VERSION
k8s-g1   Ready    <none>   92s     v1.15.4
k8s-g2   Ready    <none>   3m33s   v1.15.4
k8s-m1   Ready    master   20m     v1.15.4
k8s-m2   Ready    master   13m     v1.15.4
k8s-m3   Ready    master   8m32s   v1.15.4
k8s-n1   Ready    <none>   3m30s   v1.15.4
k8s-n2   Ready    <none>   3m35s   v1.15.4

$  kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
```
> kubeadm 的方式會讓狀態只顯示 etcd-0。


接著進入`k8s-m1`節點測試叢集 HA 功能，這邊先關閉該節點：
```sh
$ sudo poweroff
```

接著進入到`k8s-m2`節點，透過 kubectl 來檢查叢集是否能夠正常執行：
```sh
# 先檢查元件狀態
$ kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}

# 檢查 nodes 狀態
$ kubectl get no
NAME     STATUS     ROLES    AGE     VERSION
k8s-g1   Ready      <none>   4m54s   v1.15.4
k8s-g2   Ready      <none>   6m55s   v1.15.4
k8s-m1   NotReady   master   55m     v1.15.4
k8s-m2   Ready      master   33m     v1.15.4
k8s-m3   Ready      master   11m     v1.15.4
k8s-n1   Ready      <none>   6m52s   v1.15.4
k8s-n2   Ready      <none>   6m57s   v1.15.4

# 測試是否可以建立 Pod
$ kubectl run nginx --image nginx --restart=Never --port 80
$ kubectl expose pod nginx --port 80 --type NodePort
$ kubectl get po,svc
NAME        READY   STATUS    RESTARTS   AGE
pod/nginx   1/1     Running   0          27s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        56m
service/nginx        NodePort    10.106.25.118   <none>        80:30461/TCP   21s
```

透過 cURL 檢查 NGINX 服務是否正常：
```sh
$ curl 172.22.132.10:30461
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

## 結語
今天簡單透過 kubeadm 來實現 Kubernetes HA 架構，但大家肯定會發現這樣手動操作非常累，若要建立大量裸機節點時，將會有很多重複指令需要執行，因此這邊推薦結合 Ansible 這類工具進行，或者直接使用 [Kubespray](https://github.com/kubernetes-sigs/kubespray) 來自動化部署 Kubernetes HA 環境。

但是這樣就能確保服務執行在 Kubernetes 上後，就都不會出事了嗎?

![Karan Goel, Meaghan Kjelland @ Google](https://i.imgur.com/WXkQuqj.png)

事實上，不光要針對 Kubernetes 做 HA 架構，我們還要對許多層面進行處理。

![](https://i.imgur.com/PqMd0Iz.png)

## Reference
- https://kubernetes.io/docs/tasks/administer-cluster/highly-available-master/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
- https://medium.com/velotio-perspectives/demystifying-high-availability-in-kubernetes-using-kubeadm-3d83ed8c458b
- http://www.haproxy.org/
- https://www.keepalived.org/
- https://github.com/etcd-io/etcd/blob/master/Documentation/faq.md
- https://kccna18.sched.com/event/GrWQ
- https://kubernetes.io/blog/2019/06/24/automated-high-availability-in-kubeadm-v1.15-batteries-included-but-swappable/