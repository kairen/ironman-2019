---
title: "自建私有容器儲存庫(Container Registry)與實現內容信任(Content Trust)"
date: 2019-9-30
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
在地端的環境中，有許多原本能透過網路取的資源(如: 系統套件、容器映像檔等等)，有可能會基於公司一些考量(如:安全、網路等)，而將這些資源建立在本地端，然後再提供給叢集使用。其中容器儲存庫(Container Registry)是最常見的需求，因為有些團隊會要求公司測試與服務的容器映像檔，都必須從公司內部取得，這時自建一套私有容器儲存庫就非常重要。尤其是基於安全考量，還需要對映像檔進行安全掃描，或對映像檔內容進行加密等等。

而今天將說明如何自建一套容器儲存庫，並實現映像檔內容信任功能，以確保叢集使用的映像檔處於安全受信任的。在開始前，先來了解一下今天要使用到的開源軟體吧。

<!--more-->

### Harbor
[Harbor](https://github.com/goharbor/harbor) 是 CNCF Incubating 專案，該專案是基於 Docker Distribution 擴展功能的 Container Registry，提供映像檔儲存、簽署、漏洞掃描等功能。另外增加了安全、身份認證與 Web-based 管理介面等功能。

* 整合 LDAP/Active Directory、OIDC 進行使用者認證
* 整合 Clair 以實現容器映像檔安全掃描
* 整合 Notary 以實現容器映像檔簽署(Content trust)
* 支援 S3、Cloud Storage 等儲存後端
* 支援映像檔副本機制
* 提供使用者管理(User managment)UI
* 提供基於角色存取控制(Role-based access control)和活動稽核(Activity auditing)機制

### Clair
[Clair](https://github.com/coreos/clair) 是 CoreOS 開源的容器映像檔安全掃描專案，其提供 API 式的分析服務，透過比對公開漏洞資料庫 CVE（Common Vulnerabilities and Exposures）的漏洞資料，並發送關於容器潛藏漏洞的有用和可操作資訊給管理者。

![](https://i.imgur.com/bW4smc9.png)

### Notary
[Notary](https://github.com/theupdateframework/notary) 是 CNCF Incubating 專案，該專案是 Docker 對安全模組重構時，抽離的獨立專案。Notary 是用於建立內容信任的平台，目標是確保 Server 與 Client 之間交互使用已經相互信任的連線，並保證在 Internet 上的內容發佈安全性，該專案在容器應用時，能夠對映像檔、映像檔完整性等安全需求提供內容信任支援。

> TODO: 補 Portieris。

## 部署環境
本部分將說明如何部署 Harbor，並設定啟用 Clair 與 Notary。

### 節點資訊
部署沿用之前文章建置的 HA 環境進行測試，全部都採用裸機部署，作業系統為`Ubuntu 18.04+`:

| IP Address  | Hostname | CPU | Memory | Role |
|-------------|--------------|-----|--------|------|
|172.22.132.11| k8s-m1       | 4   | 16G    |Master|
|172.22.132.12| k8s-m2       | 4   | 16G    |Master|
|172.22.132.13| k8s-m3       | 4   | 16G    |Master|
|172.22.132.21| k8s-n1       | 4   | 16G    |Node  |
|172.22.132.22| k8s-n2       | 4   | 16G    |Node  |
|172.22.132.31| k8s-g1       | 4   | 16G    |Node  |
|172.22.132.32| k8s-g2       | 4   | 16G    |Node  |
|172.22.132.253| deploy-node | 4   | 16G    |Harbor|

> k8s 節點不需要這麼多，這邊只是沿用。

### 事前準備
在開始部署時，請確保滿足以下條件:

* `deploy-node`節點需要安裝容器引擎:

```sh
$ curl -fsSL "https://get.docker.com/" | sh
```

* `deploy-node`節點需要安裝 docker-compose 工具:

```sh
$ curl -L https://github.com/docker/compose/releases/download/1.24.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```

* 部署一座 Kubernetes v1.10+ 叢集。可參考[用 kubeadm 部署 Kubernetes 叢集](https://kairen.github.io/2016/09/29/kubernetes/deploy/kubeadm/)。

### Harbor 部署
Harbor 會由多個容器部署而成，因此我們需要在部署前取得安裝檔案。這邊可以利用 wget 下載 Offline 安裝的檔案:

```sh
$ wget https://storage.googleapis.com/harbor-releases/release-1.9.0/harbor-offline-installer-v1.9.0.tgz
$ tar xvf harbor-offline-installer-v1.9.0.tgz && \
    rm harbor-offline-installer-v1.9.0.tgz
$ cd harbor
```

這邊安裝透過 Offline 進行，因此在部署前，需要透過 Docker 載入映像檔:

```sh
$ docker load < harbor.v1.9.0.tar.gz

# 完成後，透過 images 指令查看
$ docker images
```

接著由於部署的 Harbor 使用 HTTPS，因此需要提供憑證。這邊由於測試用，因此以自簽(Self signed)來處理:

```sh
# 產生 CA crt 
$ openssl req \
    -newkey rsa:4096 -nodes -sha256 -keyout ca.key \
    -x509 -days 365 -out ca.crt \
    -subj "/C=TW/ST=New Taipei/L=New Taipei/O=test_company/OU=IT/CN=test"

# 產生 Harbor csr 
$ openssl req \
    -newkey rsa:4096 -nodes -sha256 -keyout harbor-registry.key \
    -out harbor-registry.csr \
    -subj "/C=TW/ST=New Taipei/L=New Taipei/O=test_company/OU=IT/CN=172.22.132.253"

$ cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth 
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor-server
IP.1=172.22.132.253
EOF

$ openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in harbor-registry.csr \
    -out harbor-registry.crt
```

複製檔案至`/data/cert/`目錄底下:

```sh
$ mkdir -p /data/cert/
$ cp -rp ./{ca.key,ca.crt,harbor-registry.key,harbor-registry.crt} /data/cert/ 
```

修改 Harbor 組態檔案`harbor.yml`內容，如以下所示:

```yaml
hostname: 172.22.132.253
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /data/cert/harbor-registry.crt
  private_key: /data/cert/harbor-registry.key
harbor_admin_password: p@ssw0rd
```

> 這邊僅修改需要欄位，其餘則保持不變。

完成後，即可執行腳本進行部署 Harbor:

```sh
$ ./prepare
...
Clean up the input dir

$ ./install.sh --with-notary --with-clair
```

### 在 Docker 存取映像檔
首先複製 ca 憑證到 Docker certs 目錄，以確保 HTTPs 能夠授權:

```sh
$ mkdir -p /etc/docker/certs.d/172.22.132.253
$ cp /data/cert/ca.crt /etc/docker/certs.d/172.22.132.253/
```

取得測試用映像檔，並推送映像檔到 Harbor 中:

```sh
$ docker pull alpine:3.7

# 輸入帳密登入
$ docker login 172.22.132.253
$ docker tag alpine:3.7 172.22.132.253/library/alpine:3.7
$ docker push 172.22.132.253/library/alpine:3.7
```

完成後，可以在 UI 上查看，如同下圖所示:

![](https://i.imgur.com/nR9kkGE.png)

### 在 Kubernetes 上存取映像檔
首先在所有 K8s 節點上，複製 ca 憑證到 Docker certs 目錄，以確保 HTTPS 能夠授權:

```sh
$ mkdir -p /etc/docker/certs.d/172.22.132.253
$ scp /data/cert/ca.crt <HOST>:/etc/docker/certs.d/172.22.132.253/
```

> 這邊建議用 Ansible 這種工具複製。

在任一能操作叢集的節點上，執行以下指令建立 Pull Secret:

```
$ kubectl create secret docker-registry regcred \
    --docker-server="172.22.132.253" \
    --docker-username=admin \
    --docker-password=p@ssw0rd \
    --docker-email=admin@example.com


$ kubectl apply -f /vagrant/harbor
$ kubectl get po
```

接著建立一個測試用 Pod:

```sh
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: alpine
    image: 172.22.132.253/library/alpine:3.7
    command: ["/bin/sh", "-c"]
    args:
    - "while :; do sleep 1; done"
EOF

$ kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
test                     1/1     Running   0          8s
```

### 映像檔 Content trust
首先透過 UI 建立新的 Project 來測試內容信任功能。

![New Project](https://i.imgur.com/m2ARRFp.png)

![Enable content trust](https://i.imgur.com/z0AIluw.png)

在 Harbor 節點上，複製 ca 憑證到 Notary certs 目錄，以確保 HTTPS 能夠授權:

```sh
$ mkdir -p $HOME/.docker/tls/172.22.132.253:4443/
$ cp /data/cert/ca.crt $HOME/.docker/tls/172.22.132.253:4443/
```

在 Docker 客戶端啟用 Content Trust，並推送一個簽署的映像檔到 Harbor:

```sh
$ export DOCKER_CONTENT_TRUST=1
$ export DOCKER_CONTENT_TRUST_SERVER=https://172.22.132.253:4443
$ docker tag alpine:3.7 172.22.132.253/trust/alpine:3.7

# 這邊會需要輸入密碼短語資訊
$ docker push 172.22.132.253/trust/alpine:3.7
...
Enter passphrase for new root key with ID 93f1593:
Repeat passphrase for new root key with ID 93f1593:
Enter passphrase for new repository key with ID 224d9cd:
Repeat passphrase for new repository key with ID 224d9cd:
Finished initializing "172.22.132.253/trust/alpine"
Successfully signed 172.22.132.253/trust/alpine:3.7
```

上傳完成後，即可以查看 UI。結果如下圖所示:

![](https://i.imgur.com/rDHeDiw.png)

當 Docker 啟用 Content Trust 時，也可測試 pull 未簽署的映像檔:

```sh
$ docker rmi 172.22.132.253/library/alpine:3.7
$ docker pull 172.22.132.253/library/alpine:3.7
Error: remote trust data does not exist for 172.22.132.253/library/alpine: 172.22.132.253:4443 does not have trust data for 172.22.132.253/library/alpine
```

## 結語
今天簡單部署 Harbor 作為私有容器儲存庫使用，可以看到 Harbor 整合了許多有用的系統與工具，如:掃描映像檔 CVE、提供 Notary 映像檔內容信任、Web-based UI 等等功能。且 Harbor 也提供身份認證系統與後端儲存的整合，這讓我們擁有 Enterpise 級的功能。

## Reference
- https://docs.docker.com/notary/getting_started/
- https://goharbor.io/