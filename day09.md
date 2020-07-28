---
title: "實現 Kubernetes Service/Ingress 同步設定 DNS 資源紀錄 Part1"
date: 2019-9-24
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
前幾天都在分享關於叢集部署與升級事情，今天來聊聊在地端 Kubernetes 常見的需求功能吧。在生產環境中，我們會將網站或系統放到 Kubernetes 上執行與管理，再利用 Service 機制把服務暴露給外部存取使用，但 Service 在預設情況下僅能支援第四層網路協定(L4，TCP/UDP)功能，故無法設定完整網域名稱(Fully Qualified Domain Name，FQDN)來存取服務，這時大家肯定會想到 Kubernetes 的 Ingress 功能，因為 Ingress 能夠實現第七層網路協定(L7)功能，並以域名(Domain name)形式來對應到 Service 的 Pod 端點。

但是地端環境不像公有雲有各種基礎建設服務(Infrastructure as a Service，IaaS)可以輕易使用與整合(比如:透過 Cloud Provider 讓 Service 整合負載平衡服務、利用 DNS 服務來對應到 Service 的負載平衡 IP 等等)，這時如果想要實現自動同步 Kubernetes Service/Ingress 資源，來設定 FQDN 的話，就會需要建立一套網域名稱系統(Domain Name System，DNS)，並實作同步 Kubernetes 物件設定 DNS 資源紀錄(DNS record)的控制器。慶幸的是，Kubernetes 社區已經有相關元件可以協助我們實踐這些機制，今天就將針對這部分來說明與實現。

<!--more-->

## 架構與元件介紹
在開始實現前，我們先來簡單了解一下會使用到的元件，並說明將會運用在什麼方面。

![](https://i.imgur.com/rZ8PoFC.png)
 
### CoreDNS
[CoreDNS](https://github.com/coredns/coredns) 是經過 CNCF 孵化畢業的開源 DNS 專案，該專案是基於 [Caddy](https://github.com/mholt/caddy) 的一部分開發而來，由於傳統的 DNS 無法很彈性地加入插件，因此非常不靈活，但 CoreDNS 實作了一套中介軟體(Middleware)介面，因此我們能夠很輕易實現插件來完成客制的功能(如 Log、Cache 等等)，也因為這關析，CoreDNS 能夠將資源紀錄儲存至 Redis、etcd 這種 Key/Value 儲存系統上。值得再提的是，CoreDNS 在 v1.11 版本正式取代 KubeDNS，我們能夠透過開始 Kubernetes plugin 來達到叢集服務發現功能。

在實作中，我們將用來給 Kubernetes 同步設定 DNS 資源紀錄使用，並提供解析 Kubernetes Service/Ingress 的 DNS 資源紀錄。

### etcd
[etcd](https://github.com/etcd-io/etcd) 是一套分散式鍵值(Key/Value)儲存系統，類似 [ZooKeeper](https://zookeeper.apache.org/) 與 [Consul](https://www.consul.io/)，而 etcd 共識機制上採用 Raft 演算法來處理多節點共識問題，另外 etcd 支援了 REST API、JSON 格式與 SSL 等等功能。

在這邊主要提供儲存 DNS 資源紀錄，主要作為 CoreDNS 與 ExternalDNS 溝通的中介。

> etcd 也被用於 Kubernetes 與 Cloud Foundry 專案中。

### ExternalDNS
[ExternalDNS](https://github.com/kubernetes-incubator/external-dns) 是 Kubernetes 社區的孵化專案，目的是協助同步 Kubernetes Service/Ingress 的資源，並將內容轉成 DNS 資源紀錄設定的 DNS 供應商與服務上。這邊將利用 ExternalDNS 定期同步 Kubernetes 資源來轉換成 DNS 資源紀錄，並將轉換的紀錄存儲到 etcd 上，接著利用 CoreDNS 的 etcd plugin 來完成資源紀錄查詢功能。

> ExternalDNS 除了支援 CoreDNS 以外，也可以設定各種 DNS 服務。

### NGINX Ingress Controller
[NGINX Ingress](https://github.com/kubernetes/ingress-nginx) 是以 NGINX 引擎為基礎開發的 Kubernetes 控制器與代理系統，主要讓 Kubernetes 能夠透過 L7 協定功能來提供外部存取容器。NGINX Ingress 會監聽 Kubernetes Ingress 資源，並依據內容產生 NGINX 設定，然後熱更新給 NGINX 使用，因此當存取 NGINX Ingress 時，就能依據設定檔的內容轉送給對應的 Kubernetes Service。

![](https://i.imgur.com/aeW5PRF.png)

> Ingress 控制器除了社區提供的專案外，也能夠使用 [Traefik](https://docs.traefik.io/user-guide/kubernetes/)、[Kong](https://github.com/Kong/kubernetes-ingress-controller)、[HAProxy](https://github.com/haproxytech/kubernetes-ingress) 等等。

## 執行流程
本節說明架構上的運作流程。這邊會分成兩種方式實現，分別為 Service 與 Ingress。

* **Ingress**: 當建立一個 Ingress 到叢集時，NGINX Ingress 會接收到 API 資源的事件更新，並在新增與更新事件中，取得 Ingress 資訊來設定 NGINX。而當 Ingress 資源順利完成功能後，ExternalDNS 會以輪詢方式取得所有 Namespace(或指定的)中的 Ingress 資源(Ingress 必須被分派 Host IP 才能進行)，並從 Ingress 資源的`spec.rules`取出 host 資訊，以產生 DNS 資源紀錄(如: A record)，接著將產生的紀錄透過 etcd 儲存，這樣當 CoreDNS 收到 FQDN 查詢請求時，就可以利用 etcd 作為 DNS 資源紀錄後端來來辨識導向。
* **Service**: 當建立一個有`metadata.annotations.external-dns.alpha.kubernetes.io/hostname` 的 Service 時，ExternalDNS 會在輪詢期間取得該欄位的值來產生 DNS 資源紀錄，然後同樣利用 etcd 作為儲存，並在對 CoreDNS 發起 FQDN 查詢請求時，能夠到 etcd 查詢 DNS 資源紀錄以解析結果返回給客戶端。

![](https://i.imgur.com/b4QPkr9.png)

如上圖所示，我們簡單拆解不同步驟來說明。

1. 建立一個有以下範例 Annotations 的 Service/Ingress。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.k8s.local # 將被自動註冊的 domain name.
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: nginx.k8s.local # 將被自動註冊的 domain name.
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
```
> * 當使用 Service 時，需要設定成 Load Balancer 或 NodePort。
> * 當使用 Ingress 時，不需要在 Service 塞入`external-dns.alpha.kubernetes.io/hostname`欄位，且不需要使用 NodePort 與 LoadBalancer。

2. ExternalDNS 透過輪詢取得 Service/Ingress 資源，取出將被用來產生 DNS 資源紀錄的資訊，接著在產生完成後，利用 etcd 進行儲存記錄。
3. 當客戶端存取 `nginx.k8s.local` 時，將對 CoreDNS 發起 FQDN 查詢請求，這時 CoreDNS 會到 etcd 查找 DNS 資源紀錄，以解析指定的 IP 回應給客戶端，
4. 這時客戶端會接受到解析結果，被正確地導向到解析的 IP 位址。

> 若使用 Service 時，因為不是走 HTTP/HTTPS 協定，因此要輸入 TCP/UPD 的 Port。用 Ingress 則以域名存取即可，因為 NGINX Ingress 提供了一個 NGINX 代理後端，它會幫你轉發至 Kubernetes 內部 Service。

## 結語
今天說明如何在地端實現自動更新 Kubernetes Service/Ingress 到 DNS 中的架構與流程，過程中大家可以了解到 Kubernetes 社區利用 [Controller Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/) 機制來擴展功能，以讓供應商整合自家功能與服物到 Kubernetes 中，像今天提到的 ExternalDNS 就是實現自動同步 DNS 資源紀錄到各種 DNS 服務的 Kubernetes 控制器。

今天只是說明元件架構與流程，明天我們將實際的實現該功能。

## Reference
- https://github.com/kubernetes-incubator/external-dns/blob/master/docs/tutorials/cloudflare.md
- https://coredns.io/2018/11/27/cluster-dns-coredns-vs-kube-dns/
- https://zhengyinyong.com/coredns-basis.html
- https://www.hwchiu.com/ingress-1.html