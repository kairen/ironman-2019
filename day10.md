---
title: "實現 Kubernetes Service/Ingress 同步設定 DNS 資源紀錄 Part2"
date: 2019-9-25
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
有時地端 Kubernetes 會需要提供給內部團隊使用，而團隊人員若希望以域名方式直接存取 Kubernetes 上的服務時，就必須建立一套機制。但這樣需求中，也增加了維運人員的負擔，因為若沒有自動化機制，就要在 Kubernetes Ingress/Service 變動時，手動處理 DNS 資源紀錄，以確保內部團隊能夠解析到位址。

基於此原因，今天延續在[實現 Kubernetes Service/Ingress 同步設定 DNS 資源紀錄 Part1](https://k2r2bai.com/2019/09/24/ironman2020/day09/)的文章提到的架構，實際將該架構部署到一座地端 Kubernetes 叢集上測試，並透過實作過程來了解其功能是如何運作的。

<!--more-->

## 環境建置
本部分將說明如何建立昨天文章提到的架構。

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

### 部署 DNS 系統
首先取得 Kubernetes 部署檔案的 Git 存放庫，並進入 addons 目錄:

```sh
$ git clone https://github.com/cloud-native-taiwan/kourse.git
$ cd kourse/addons
```

接著利用 sed(or perl) 工具修改等下要被部署的檔案(範例原本是 Workshop 使用，內容 IP 打死):

```sh
$ export VIP="172.22.132.10"
$ export DN="k8s.kairen.tw"
$ sed -i "s/192.16.35.12/${VIP}/g" ingress-controller/service.yml
$ sed -i "s/192.16.35.12/${VIP}/g" dns/coredns/service-tcp.yml
$ sed -i "s/192.16.35.12/${VIP}/g" dns/coredns/service-udp.yml
$ sed -i "s/k8s.local/${DN}/g" dns/coredns/configmap.yml
```

> `VIP` 請依據自己環境部署為主。`DN`由於當初是寫死用於教學用，這邊可以自行修改。

#### NGINX Ingress Controller
當取得到檔案，且修改需要的內容後，就可以透過 kubectl 來部署元件到 Kubernetes 叢集中。首先由於測試會用到 Ingress，因此需要一個 Ingress Controller，可以透過以下指令進行:

```sh
$ kubectl apply -f ingress-controller/
namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
deployment.apps/nginx-ingress-controller created
service/ingress-nginx created
```

> 若環境不同，請修改`addons/ingress-controller/`底下 YAML 檔案。

完成後，查看 `ingress-nginx` Namespace 是否有正確啟動 Ingress controller:

```sh
$ kubectl -n ingress-nginx get po,svc
NAME                                            READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-controller-85b6f5f57d-cs4pw   1/1     Running   0          101s

NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
service/ingress-nginx   LoadBalancer   10.109.121.69   172.22.132.10   80:30297/TCP,443:32686/TCP   101s
```

沒問題後，利用 cURL 工具存取服務來驗證:

```sh
$ curl 172.22.132.10
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.15.6</center>
</body>
</html>
```

也可以以瀏覽器開啟`External-IP:80`頁面來查看。

![](https://i.imgur.com/wThG3PC.png)

### CoreDNS + etcd
接著要部署一套 DNS 來提供 FQDN 查詢使用。首先建立一個 Namespace 用來管理這些部署的元件，以確保不會跟 Kubernetes 叢集中的其他服務混肴:

```sh
$ kubectl apply -f dns/
namespace/ddns created
```

Namespace 建立後，即可部署 etcd 與 CoreDNS:

```sh
$ kubectl apply -f dns/etcd/ -f dns/coredns/
deployment.apps/coredns-etcd created
service/coredns-etcd created
configmap/coredns created
deployment.apps/coredns created
service/coredns-tcp created
service/coredns-udp created

$ kubectl -n ddns get po,svc
NAME                               READY   STATUS    RESTARTS   AGE
pod/coredns-7cc8dcc778-9xght       1/1     Running   0          2m47s
pod/coredns-etcd-675b96b65-2kmdb   1/1     Running   0          2m47s

NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                       AGE
service/coredns-etcd   ClusterIP      10.107.159.40    <none>          2379/TCP,2380/TCP             2m47s
service/coredns-tcp    LoadBalancer   10.105.156.110   172.22.132.10   53:32627/TCP,9153:31482/TCP   2m46s
service/coredns-udp    LoadBalancer   10.105.63.16     172.22.132.10   53:30388/UDP                  2m46s
```

> 若環境不同，請修改`dns/coredns/`底下 YAML 檔案。

完成後，利用 dig 工具來查看 DNS SOA(Start Of Authority) 是否正常:

```sh
$ dig @172.22.132.10 SOA k8s.kairen.tw +noall +answer

; <<>> DiG 9.10.6 <<>> @172.22.132.10 SOA k8s.kairen.tw +noall +answer
; (1 server found)
;; global options: +cmd
k8s.kairen.tw.		30	IN	SOA	ns.dns.k8s.kairen.tw. hostmaster.k8s.kairen.tw. 1569412779 7200 1800 86400 30
```

確認沒問題後，即可在測試機器設定 DNS Nameserver，如下圖。

![](https://i.imgur.com/p6vkPPw.png)

### ExternalDNS
當用於查詢的 CoreDNS 與儲存 DNS 資源紀錄的 etcd 完成後，即可部署 ExternalDNS 來提供同步 Kubernetes Ingress/Service 的 DNS 資源紀錄:

```sh
$ kubectl apply -f dns/externaldns/
deployment.apps/external-dns created
clusterrole.rbac.authorization.k8s.io/external-dns created
clusterrolebinding.rbac.authorization.k8s.io/external-dns-viewer created
serviceaccount/external-dns created

$ kubectl -n ddns get po -l k8s-app=external-dns
NAME                           READY   STATUS    RESTARTS   AGE
external-dns-d674c579f-r5xp6   1/1     Running   0          99s
```

> 若環境不同，請修改`addons/dns/external-dns/`底下 YAML 檔案。

完成後，檢查是否正確執行:

```bash
$ kubectl -n ddns logs -f external-dns-d674c579f-r5xp6
...
time="2019-09-25T12:05:57Z" level=debug msg="No endpoints could be generated from service ingress-nginx/ingress-nginx"
time="2019-09-25T12:05:57Z" level=debug msg="No endpoints could be generated from service ddns/coredns-etcd"
time="2019-09-25T12:05:57Z" level=debug msg="No endpoints could be generated from service ddns/coredns-tcp"
```

到這邊就完成所有元件部署了，接下來就能實際測試功能囉。

## 功能驗證
一但該系統在 Kubernetes 建立完成後，就能夠執行一些簡單範例進行驗證。這邊我們使用名為 cheese 的範例來驗證，這個範例會建立三個不同網頁，並利用 Ingress 來導向指定頁面:


* `stilton.k8s.kairen.tw` 將導到`斯蒂爾頓`起司頁面。
* `cheddar.k8s.kairen.tw` 將導到`切達`起司頁面。
* `wensleydale.k8s.kairen.tw` 將導到`文斯勒德起司`起司頁面。

在開始前，先用 dig 工具查看一下 A record 是否能夠解析到:

```sh
$ dig @172.22.132.10 A stilton.k8s.kairen.tw +noall +answer

; <<>> DiG 9.10.6 <<>> @172.22.132.10 A stilton.k8s.kairen.tw +noall +answer
; (1 server found)
;; global options: +cmd
```

若沒有的話，就可以執行以下指令來部署 cheese 範例:

```sh
$ export DN="k8s.kairen.tw"
$ cd kourse/practical-k8s/practical-apps/
$ sed -i "s/example.k8s.local/${DN}/g" lab6-cheese/cheese-ing.yml
$ kubectl apply -f lab6-cheese/
deployment.apps/stilton created
deployment.apps/cheddar created
deployment.apps/wensleydale created
ingress.networking.k8s.io/cheese created
service/stilton created
service/cheddar created
service/wensleydale created

$ kubectl get po,svc,ing
NAME                               READY   STATUS    RESTARTS   AGE
pod/cheddar-59666cdbc4-mzlsl       1/1     Running   0          74s
pod/stilton-d9485c498-g9xp4        1/1     Running   0          74s
pod/wensleydale-79f5fc4c5d-pv9cg   1/1     Running   0          74s

NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/cheddar       ClusterIP   10.108.201.33    <none>        80/TCP    74s
service/stilton       ClusterIP   10.105.135.218   <none>        80/TCP    74s
service/wensleydale   ClusterIP   10.109.92.103    <none>        80/TCP    73s

NAME                        HOSTS                                                                   ADDRESS         PORTS   AGE
ingress.extensions/cheese   stilton.k8s.kairen.tw,cheddar.k8s.kairen.tw,wensleydale.k8s.kairen.tw   172.22.132.10   80      74s
```

建立完成後，當 ExternalDNS 輪詢時，就會將 Ingress/Service 產生成 DNS 資源紀錄，並儲存到 etcd 中。我們可以利用 kubectl logs 來查看:

```sh
$ kubectl -n ddns logs -f external-dns-d674c579f-r5xp6
...
time="2019-09-25T12:37:07Z" level=debug msg="Endpoints generated from ingress: default/cheese: [stilton.k8s.kairen.tw 0 IN A 172.22.132.10 [] cheddar.k8s.kairen.tw 0 IN A 172.22.132.10 [] wensleydale.k8s.kairen.tw 0 IN A 172.22.132.10 []]"
...
```

都沒問題後，透過 dig 工具解析看看 A record 是否正常:

```sh
$ dig @172.22.132.10 A stilton.k8s.kairen.tw +noall +answer

; <<>> DiG 9.10.6 <<>> @172.22.132.10 A stilton.k8s.kairen.tw +noall +answer
; (1 server found)
;; global options: +cmd
stilton.k8s.kairen.tw.	300	IN	A	172.22.132.10
```

這時也能透過瀏覽器查看`stilton.k8s.kairen.tw`、`cheddar.k8s.kairen.tw`與`wensleydale.k8s.kairen.tw`頁面。如下圖所示

![](https://i.imgur.com/Qc7pSK9.jpg)

![](https://i.imgur.com/rhSPAaz.jpg)

## 結語
今天實作了自動化同步 Ingress/Service 的 DNS 資源紀錄的功能。實作中，涉及了 L7 網路協定功能(HTTP, HTTPS, DNS)在地端 Kubernetes 上的實現。我們可以發現 Kubernetes 社區的生態圈，在各種需求上，已經有很多完善的工具或元件可以使用，像是 ExternalDNS 就是很好例子，讓人可以不用手動設定 DNS 資源紀錄，自動從 Kubernetes Ingress/Service 產生。

> 不過今天做法是地端(或內部)部署情境使用，若公司網域是由供應商(如 CloudFlare 或 AWS Route53)提供的話，就必須調整 [ExternalDNS Providers](https://github.com/kubernetes-incubator/external-dns#the-latest-release-v05) 來支援。

## Reference
- https://github.com/kubernetes-incubator/external-dns/blob/master/docs/tutorials/cloudflare.md
- https://coredns.io/2018/11/27/cluster-dns-coredns-vs-kube-dns/
- https://zhengyinyong.com/coredns-basis.html
- https://www.hwchiu.com/ingress-1.html
- https://kubernetes.github.io/ingress-nginx/