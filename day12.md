---
title: "實作 Kubernetes 裸機 Load Balancer Part2"
date: 2019-9-27
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
昨天文章中，我們提到想要讓同一個叢集能夠支援兩個同樣的 TCP/UDP 曝露給外部存取，雖然能夠利用 Service LoadBalancer 或 NodePort 類型來達到需求，但是這兩者依然存在著限制，比如說 NodePort 使用叢集節點 IP:Port 方式來提供存取，這存在著單點故障問題，且建立一個 Port 就會在所有節點綁定;而 LoadBalancer 則不支援地端分配負載平衡 IP 的機制，只能透過手動在`externalIPs`欄位指定，若沒指定的話，其功能只是繼承 NodePort 機制，多了個 Target Port 能夠直接存取而已，而且儘管能夠在`externalIPs`指定 IP，但這些 IP 又該從哪邊來呢?又怎麼分配呢?那該怎麼解決呢?

很慶幸的是有人開發了一個開源專案 [MetalLB](https://metallb.universe.tf/) 來幫助我們解決這些問題，而今天就是要來探討這個專案如何使用。

<!--more-->

## 環境部署  
本部分將說明如何部署與使用 MetalLB，並用於後續架構分使用。

### 節點資訊
部署沿用之前文章建置的 HA 環境進行測試，全部都採用裸機部署，作業系統為`Ubuntu 18.04+`:

| IP Address  | Hostname | CPU | Memory | Role |
|-------------|----------|-----|--------|------|
|172.22.132.11| k8s-m1   | 4   | 16G    |Master|
|172.22.132.12| k8s-m2   | 4   | 16G    |Master|
|172.22.132.13| k8s-m3   | 4   | 16G    |Master|
|172.22.132.21| k8s-n1   | 4   | 16G    |Node  |
|172.22.132.22| k8s-n2   | 4   | 16G    |Node  |
|172.22.132.31| k8s-g1   | 4   | 16G    |Node  |
|172.22.132.32| k8s-g2   | 4   | 16G    |Node  |

> 另外所有 Master 節點將透過 Keepalived 提供一個 Virtual IP `172.22.132.10` 作為使用。

### 事前準備
在開始部署時，請確保滿足以下條件:

* 確保擁有一座版本為 v1.13.0+ 的 Kubernetes 叢集。
* 使用能夠與 MetalLB 共存的 [Network Plugins](https://metallb.universe.tf/installation/network-addons/)。
* 準備一些用於 IPv4 的 IP 位址。必須確保 L2/L3 網路能夠通。這邊將使用`172.22.132.150-172.22.132.200`。
* 若使用到 L3 功能，則還需要 BGP 路由。

### MetalLB 安裝
MetalLB 提供了以容器方式部署到 Kubernetes，且官方也有提供 YAML 讓我們執行:

```sh
$ kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.1/manifests/metallb.yaml
namespace/metallb-system created
podsecuritypolicy.policy/speaker created
serviceaccount/controller created
serviceaccount/speaker created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller created
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created
role.rbac.authorization.k8s.io/config-watcher created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker created
rolebinding.rbac.authorization.k8s.io/config-watcher created
daemonset.apps/speaker created
deployment.apps/controller created
```

> 使用 Helm 也能參考這邊 [Installation With Helm](https://metallb.universe.tf/installation/) 安裝 

只要執行上面指令後，即可完成部署。這時可以透過 kubectl 來查看`metallb-system`的 Namespace:

```sh
$ kubectl -n metallb-system get po
NAME                          READY   STATUS    RESTARTS   AGE
controller-6bcfdfd677-q9fzp   1/1     Running   0          5m
speaker-8648w                 1/1     Running   0          5m
speaker-8h4gs                 1/1     Running   0          5m
speaker-f9zh4                 1/1     Running   0          5m
speaker-dc134                 1/1     Running   0          5m
speaker-xnkt5                 1/1     Running   0          5m
speaker-zczp5                 1/1     Running   0          5m
speaker-zzn5v                 1/1     Running   0          5m
```

若這邊沒問題，就表示已經完成 MetalLB 安裝。接著需要新增 IP Pools 設定，以讓 MetalLB 能夠自動分配 IP 給 Service 的 LoadBalancer 類型使用，下面為一個 L2 的範例:

```sh
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      auto-assign: true
      addresses:
      - 172.22.132.150-172.22.132.200
    # - name: production
    #   auto-assign: false
    #   avoid-buggy-ips: true
    #   addresses:
    #   - 172.22.131.0/24
EOF
```

> * 這邊也可以用 CIDR 來表示。更多的設定可以參考 [MetalLB Configuration](https://metallb.universe.tf/configuration/)
> * 另外由於測試環境限制，僅以 L2 範例為主。

## 功能驗證
當安裝與設定完成後，即可新增一個 Service 來驗證功能:

```sh
$ kubectl run nginx --image nginx --port 80
deployment.apps/nginx created

$ kubectl expose deploy nginx --port 8080 --target-port 80 --type LoadBalancer
service/nginx exposed
```

建立好後，透過 kubectl 來查看 Service:

```sh
$ kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
nginx        LoadBalancer   10.111.63.50   172.22.132.153   8080:31827/TCP   59s
```

這時會看到，MetalLB 在 Service 為 LoadBalancer 時，會自動從前面設定的`default` Pool 中，分配一個 IP 給 Service 使用。當有 IP 時，可以嘗試利用 cURL 來存取看看:

```sh
$ curl 172.22.132.153:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
...
```

接著若再新建一個不同 Port 的 Service 會怎樣呢?

```sh
$ kubectl expose deploy nginx --name nginx-80 --port 80 --target-port 80 --type LoadBalancer
$ kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
nginx        LoadBalancer   10.111.63.50    172.22.132.153   8080:31827/TCP   5m16s
nginx-80     LoadBalancer   10.97.188.162   172.22.132.154   80:30218/TCP     4s
```

大家會發現 MetalLB 又分配了另一個 IP 來使用，這時肯定會覺得這樣是不是每個 IP 只能使用一個 Port，事實上 MetalLB 能夠在 Service 的 Annotation 中，新增`metallb.universe.tf/allow-shared-ip`欄位來達到 IP Sharing 功能，這邊可以參考 [IP Address Sharing](https://metallb.universe.tf/usage/)。

## 結語
今天利用 MetalLB 達成了裸機負載平衡功能，而明天我將針對該專案進行原理分析。

> 最近都沒啥時間好好寫，所以一些缺少內容後續會再慢慢補齊。只能說一天寫一篇真的不簡單...

## Reference
- https://metallb.universe.tf/
- https://medium.com/@JockDaRock/metalloadbalancer-kubernetes-on-prem-baremetal-loadbalancing-101455c3ed48