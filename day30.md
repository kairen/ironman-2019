---
title: "動手實作 Kubernetes 自定義控制器 Part6"
date: 2019-10-15
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
在[動手實作 Kubernetes 自定義控制器 Part5](https://k2r2bai.com/2019/10/14/ironman2020/day29/) 文章結束後，基本上已經完成了這個自定義控制器範例的功能，這時若我們想要部署這個控制器的話，該怎麼辦呢?因為過去文章中，我們都是以 Go 語言指令直接建構程式進行測試，且使用 client-go 與 API Server 溝通時，都是以`cluster-admin`使用者來達成，這種作法如果是正式上線環境，必然會有很多疑慮，比如說控制器環境有安全問題，如果這些狀況被取得 Kubernetes cluster-admin 權限的話，就可能會危害到整個 Kubernetes 環境，因為 cluster-admin 可以操作任何 Kubernetes API 資源。基於這些問題，今天就是要來說明如何讓控制器正確的部署到 Kubernetes 叢集中執行。

<!--more-->

## Deploy in the cluster
由於自定義控制器部署到 Kubernetes 叢集時，需要擁有特定使用者與權限操作 Kubernetes APIs，才能正常的執行控制器功能，因此必須利用 [Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) 與 [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) 來讓控制器能在 Pod 中與 API Server 溝通。

> 雖然 Kubernetes 在建立 Namespace 時，預設也會自動建立一個名稱為`default`的 Service Account，但這個 Service Account 通常會被用於該 Namespace 下的所有 Pod，因此不建議將 RBAC 權限賦予給這個 Service Account。

而要提供部署到 Kubernetes 叢集中，通常需要準備以下幾個檔案:

```sh
├── Dockerfile # 將控制器建構成容器映像檔
└── deploy # 提供部署控制器的目錄
    ├── crd.yml # 自定義控制器的 CRDs
    ├── deployment.yml # 用於部署自定義控制器程式本身
    ├── rbac.yml # 自定義控制器的 API 存取權限
    └── sa.yml # 自定義控制器的服務帳戶，會與 RBAC 結合以限制 Pod 存取 API 的權限
```

### 環境準備
由於今天的實作內容需要用到 Kubernetes 與 Docker，因此須完成以下需求與條件:

* 一座 Kubernetes v1.10+ 叢集。透過 [Minikube](https://github.com/kubernetes/minikube) 建立即可 `minikube start --kubernetes-version=v1.15.4`。
* 一個 Docker 環境，可以直接 Minikube 執行`eval $(minikube docker-env)`來取的 Docker 參數，並遠端操作。

### Implementation
本部分將建立這些檔案，並執行所需的操作。首先由於要將控制器部署到 Kubernetes 中，因此必須將控制器程式建構成容器映像檔，這樣才能透過 Pod 的形式來部署。而建構映像檔可以透過 Dockerfile 來達到，如以下內容:

```bash
FROM kairen/golang-dep:1.12-alpine AS build

ENV GOPATH "/go"
ENV PROJECT_PATH "$GOPATH/src/github.com/cloud-native-taiwan/controller101"
ENV GO111MODULE "on"

COPY . $PROJECT_PATH
RUN cd $PROJECT_PATH && \
  make && mv out/controller /tmp/controller

FROM alpine:3.7
COPY --from=build /tmp/controller /bin/controller
ENTRYPOINT ["controller"]
```

完成後，即可透過 Docker 指令來建構與推送到公有的 Container registry 上以便後續部署使用:

```sh
$ eval $(minikube docker-env)
$ docker build -t <owner>/controller101:v0.1.0 .
$ docker push <owner>/controller101:v0.1.0
```

接著我們將新增以下檔案用於部署控制器到 Kubernetes 上執行。

#### deploy/crd.yml
用於以 CRD API 來新增自定義資源的檔案。在這檔案中，通常會定義一個或多個自定義資源內容。

```yml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: virtualmachines.cloudnative.tw
spec:
  group: cloudnative.tw
  version: v1alpha1
  names:
    kind: VirtualMachine
    singular: virtualmachine
    plural: virtualmachines
    shortNames:
    - vm
    - vms
  scope: Namespaced
  additionalPrinterColumns:
  - name: Status
    type: string
    JSONPath: .status.phase
  - name: CPU
    type: number
    JSONPath: .status.server.usage.cpu
  - name: Memory
    type: number
    JSONPath: .status.server.usage.memory
  - name: Age
    type: date
    JSONPath: .metadata.creationTimestamp
```

> 有些自定義控制器會直接透過 [apiextensions-apiserver](https://github.com/kubernetes/apiextensions-apiserver) 在程式啟動時，將自定義資源自動新增到當前 Kubernetes 叢集中。

#### deploy/sa.yml
該檔案會定義一個 Service Account 用於提供給控制器程式在 Pod 中與 API Server 溝通的帳戶，這個帳戶會透過 RBAC 賦予特定的權限，以便控制器程式獲取操作所需 API 資源的權限。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: controller101
  namespace: kube-system
```

#### deploy/rbac.yml
該檔案會賦予指定 Service Account 相對應的 API 權限，由於這個自定義控制器需要對 VirtualMachines 資源做各種操作(如 Create、Update、Delete、Get 與 Watch 等等)，因此需要設定相對應的 API Group 與 Resources 來確保控制器程式能獲取權限。

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: controller101-role
rules:
- apiGroups:
  - cloudnative.tw
  resources:
  - "virtualmachines"
  verbs:
  - "*"
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: controller101-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: controller101-role
subjects:
- kind: ServiceAccount
  namespace: kube-system
  name: controller101
```

> 如果控制器需要存取其他 API 時，就必須在`rules`欄位額外新增。請務必確保給予最小可運作的權限。

#### deploy/deployment.yml
該檔案定義自定義控制器的容器如何在 Kubernetes 部署，我們會在以 Deployment 方式進行部署，因為能利用 Deployment 機制來確保控制器在叢集中的可靠性。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: controller101
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: controller101
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        k8s-app: controller101
    spec:
      priorityClassName: system-cluster-critical # 由於控制器有可能是重要元件，因此要確保節點資源不足時，不會優先被驅逐
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      serviceAccountName: controller101
      containers:
      - name: controller
        image: kairen/controller101:v0.1.0
        env:
        - name: DOCKER_HOST
          value: "tcp://192.168.99.159:2376"
        - name: DOCKER_TLS_VERIFY
          value: "1"
        - name: DOCKER_CERT_PATH
          value: "/etc/docker-certs"
        args:
        - --v=2
        - --logtostderr=true
        - --vm-driver=docker
        - --leader-elect=false
        volumeMounts:
        - name: docker-certs
          mountPath: "/etc/docker-certs"
          readOnly: true
      volumes:
      - name: docker-certs
        secret:
          secretName: docker-certs
```

> * 其中`env`部分，需要依據環境差異來改變。
> * 若有多個副本時，需要透過`--leader-elect`來啟用 Leader Election 機制。

### Deployment
當檔案都建立好後，就可以透過 kubectl 來部署到 Kubernetes 叢集。首先由於範例使用 Docker Driver 來測試，因此需要在控制器程式啟動時載入 Docker Certs，故這邊要先將這些檔案以 Secrets 方式存到 Kubernetes 中，之後再提供給 Deployment 掛載使用。

```sh
$ minikube docker-env
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.159:2376"
export DOCKER_CERT_PATH="/Users/test/.minikube/certs"

$ kubectl -n kube-system create secret generic docker-certs \
    --from-file=$HOME/.minikube/certs/ca.pem \
    --from-file=$HOME/.minikube/certs/cert.pem \
    --from-file=$HOME/.minikube/certs/key.pem
```

建立好 Docker certs 後，就可以執行以下指令進行部署:

```sh
$ kubectl apply -f deploy/
customresourcedefinition.apiextensions.k8s.io/virtualmachines.cloudnative.tw created
deployment.apps/controller101 created
clusterrole.rbac.authorization.k8s.io/controller101-role created
clusterrolebinding.rbac.authorization.k8s.io/controller101-rolebinding created
serviceaccount/controller101 created

$ kubectl -n kube-system logs -f controller101-7858db7484-5bjvf
I1015 11:14:45.159556       1 controller.go:77] Starting the controller
I1015 11:14:45.159640       1 controller.go:78] Waiting for the informer caches to sync
I1015 11:14:45.262042       1 controller.go:86] Started workers
```

> 若 Minikube 的 Docker envs 資訊不同時，需要修改`deploy/deployment.yml`裡面 envs。

接著嘗試建立一個 VirtualMachine 資源實例，並用 kubectl get 與 docker ps 查看狀態:

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
test-vm   Active   0     0.11359797976548552   22s


$ docker ps --filter "name=test-vm"
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
5ac009171998        nginx:1.17.4        "nginx -g 'daemon of…"   46 seconds ago      Up 45 seconds       80/tcp              test-vm
```

## 結語
從今天實作中，可以了解到部署自定義控制器到 Kubernetes 叢集並非難事，一但控制器容器化後，並以 Kubernetes API 資源形式部署時，就能夠增加控制器的可攜帶性與維護性，往後有版本更新時，也可以利用 Kubernetes 的一些機制(如 Rolling Upgrade)來安全地更新控制器程式。

最後由於鐵人賽文章不夠用，因此關於 CKA/CKAD 認證、 Admission Controller 等文章，都會在 [KaiRen's Blog](http://k2r2bai.com) 上持續新增。

## Reference
- https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/