---
title: "動手實作 Kubernetes 自定義控制器 Part1"
date: 2019-10-10
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
昨天了解到 Kubernetes 官方的 Sample Controller，是如何讓一個自定義 API 資源被自定義控制器管理。雖然這個範例僅僅只是管理一個 Deployment 資源，但可以讓人認識到一個自定義控制器是如何運作的。而接下來的文章，我將每天撰寫一小部分程式內容，來重頭慢慢實作一個管理自定義資源`VirtualMachine`的控制器，並隨時間推移新增更多功能(如: LeaseLock、Metrics、Fake client 與 Finalizer、Admission Controller 等等)來完善這個控制器範例。

<!--more-->

本文章的自定義控制器，會實作監聽自定義資源 VirtualMachine 的狀態，並依據 VirtualMachine 的`.spec`內容來操作私有雲中的虛擬機。另外該控制器也會持續同步私有雲虛擬機的狀態，並更新至子資源`.status`中。架構如下圖所示。

![](https://i.imgur.com/WD3qr51.png)

而今天文章先把重點放在自定義 API 資源的資料結構，以及如何透過 [code-generator](https://github.com/kubernetes/code-generator) 產生控制器所需的程式碼。

## 開發環境設置
由於開發中，會使用到 Kubernetes 與 Go 語言，因此需要安裝這些到開發環境中。

* 一座 Kubernetes v1.10+ 叢集。透過 [Minikube](https://github.com/kubernetes/minikube) 建立即可 `minikube start --kubernetes-version=v1.15.4`。
* 安裝 kubectl v1.10+ 工具，安裝請參考 [Install and Set Up kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* 安裝 Go 語言 v1.11+ 開發環境，由於開發中會使用到 Go mod 來管理第三方套件，因此必須符合支援版本。安裝請參考 [Go Getting Started](https://golang.org/doc/install)。

## 產生自定義資源程式碼
在開始定義 API 資源結構與使用 code-generator 前，需要先初始化存放控制器程式的目錄，其結構如下所示:

```sh
controller101
├── LICENSE
├── README.md
├── cmd    # Controller 主程式 main.go 檔
├── deploy # 部署 Controller 的相關檔案，如 Deployment、CRD、RBAC。
├── go.mod # Go mod package 檔案
├── go.sum # Go mod package 檔案
├── hack   # 存放一些常使用到的腳本
└── pkg    # 控制器相關程式碼
```

目錄結構完成後，就能開始撰寫自定義資源，以及 code-generator 所需的內容。而要達到這個目的，我們必須涉及兩個步驟:

* 定義 CRD 內容與資源類型程式結構。
* 透過 code-generator 腳本產生自定義資源的 Clientset、Informers、Listers 程式碼。

### Defining types and scripts
如前言所述，本範例希望透過新增一個自定義 API 資源，用於提供給自定義控制器實現功能。故我們必須在開始撰寫控制器前，先依據 [Kubernetes-style API types](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md) 的規範，來定義 API 群組(Group)與資源類型(Type)，並透過 CRD API 來建立自定義資源。所以假設控制器是所屬組織`Cloud Native Taiwan`想開發，那麼 API 群組就能定義為`cloudnative.tw`，而 API 資源則為`VirtualMachine`，因此 CRD 就會形成如下所示:

```yaml
# File name: deploy/crd.yml
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
  scope: Namespaced
```

當 CRD 定義完後，就能透過 CRD API 來新增自定義資源:

```sh
$ kubectl apply -f deploy/crd.yml
customresourcedefinition.apiextensions.k8s.io/virtualmachines.cloudnative.tw created

$ kubectl get crd
NAME                             CREATED AT
virtualmachines.cloudnative.tw   2019-10-07T14:50:47Z

$ kubectl get vm
No resources found
```

到這裡，會發現我們只是新增 API 而已，這個 VirtualMachine 資源完全沒有規範內容，這使我們無法描述一個 VirtualMachine 的預期內容，以提供給控制器處理。基於此，必須依據實作的功能假設自定義資源結構，如本範例想用 VirtualMachine 來管理虛擬機器，那麼就需要定義如下:

```yaml
apiVersion: cloudnative.tw/v1alpha1
kind: VirtualMachine 
metadata: 
  name: test-vm
spec:
  action: active
  resource:
    cpu: 2
    memory: 4G
    rootDisk: 40G
status:
  phase: synchronized
  server:
    state: active
    usage:
      cpu: 13.3 
      memory: 40.1
  lastUpdateTime: 2019-10-07T14:50:47Z
```

雖然這個 YAML 能夠透過 kubectl 進行各種 API 操作，但如果想在自定義控制器操作的話，那該怎麼辦呢?

有接觸過 Kubernetes clinet-go 的人，可能會想到用 [Dynamic client](https://github.com/kubernetes/client-go/tree/master/dynamic) 來解決。但這不是好做法，因為以下原因:

1. Dynamic client 無法很方便實現 API 資源類型的操作、轉換與驗證等等事情。
2. Dynamic client 不像原生 API 資源類型，提供了很方便的 [Typed client](https://github.com/kubernetes/client-go/tree/master/kubernetes) 可以使用。

因此過去版本(v1.7 或更舊版本)的控制器管理自定義資源較為複雜，但這問題在 v1.8 版中被解決，Kubernetes 官方引入 code-generator 專案，用於產生如同原生 API 資源類型一樣功能的 Typed client 程式碼，這樣當我們在自定義控制器使用時，就如同使用 client-go 一樣。而要達到這樣事情，必須在專案建立產生程式碼所需的所有檔案，其檔案結構如下所示:

```sh
├── hack 
│   └── k8s # Code-generator 腳本
│       ├── boilerplate.go.txt
│       ├── tools.go
│       ├── update-generated.sh
│       └── verify-codegen.sh
│   
└── pkg
    └── apis # APIs 定義
        └── cloudnative # 提供該 Package 的 API Group Name。
            ├── register.go
            └── v1alpha1 # API 各版本結構定義。Kubernetes API 是支援多版本的。
                ├── doc.go
                ├── register.go
                └── types.go
```

#### v1alpha1/doc.go
該檔案用於定義 code-generator 的 Global tags。可標示當前版本 Package 中的每個類型，想要透過 code-generator 產生哪些程式碼(如 Deepcopy, Client)。

```go
// +k8s:deepcopy-gen=package
// +groupName=cloudnative.tw

// Package v1alpha1 is the v1alpha1 version of the API.
package v1alpha1 // import "github.com/cloud-native-taiwan/controller101/pkg/apis/cloudnative/v1alpha1"
```

如上述內容，透過`+k8s:deepcopy-gen=package`標示這個 package 要建立 Deepcopy 方法(Method)。另外`+groupName=cloudnative.tw`定義整個 API Group 名稱，以確保在 code-generator 發生錯誤時，能產生可識別的錯誤代號。

#### v1alpha1/types.go
該檔案用於定義資源類型的資料結構，以及定義 code-generator 的 Local tags。可標示哪些資源類型想透過 code-generator 產生 Client 程式碼。

```go
package v1alpha1

import (
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

type VirtualMachine struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   VirtualMachineSpec   `json:"spec"`
	Status VirtualMachineStatus `json:"status"`
}

type VirtualMachineSpec struct {
	Resource corev1.ResourceList `json:"resource"`
}

type VirtualMachinePhase string

const (
	VirtualMachineNone        VirtualMachinePhase = ""
	VirtualMachineCreating    VirtualMachinePhase = "Creating"
	VirtualMachineActive      VirtualMachinePhase = "Active"
	VirtualMachineFailed      VirtualMachinePhase = "Failed"
	VirtualMachineTerminating VirtualMachinePhase = "Terminating"
	VirtualMachineUnknown     VirtualMachinePhase = "Unknown"
)

type ResourceUsage struct {
	CPU    float64 `json:"cpu"`
	Memory float64 `json:"memory"`
}

type ServerStatus struct {
	ID    string        `json:"id"`
	State string        `json:"state"`
	Usage ResourceUsage `json:"usage"`
}

type VirtualMachineStatus struct {
	Phase          VirtualMachinePhase `json:"phase"`
	Reason         string              `json:"reason,omitempty"`
	Server         ServerStatus        `json:"server,omitempty"`
	LastUpdateTime metav1.Time         `json:"lastUpdateTime"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

type VirtualMachineList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`

	Items []VirtualMachine `json:"items"`
}
```

這邊的`+genclient` tag 是表示在執行 code-generator 時，會對這個類型建立 Client 程式碼。而`+k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object`則表示產生的 Deepcopy 使用 runtime.Object 介面實現。

#### v1alpha1/register.go
用於將剛建立的新 API 版本與新資源類型註冊到 API Group Schema 中，以便 API Server 能夠識別。

> Scheme: 用於 API 資源群組之間的序列化、反序列化與版本轉換。

```go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"

	"github.com/cloud-native-taiwan/controller101/pkg/apis/cloudnative"
)

var SchemeGroupVersion = schema.GroupVersion{Group: cloudnative.GroupName, Version: "v1alpha1"}

func Kind(kind string) schema.GroupKind {
	return SchemeGroupVersion.WithKind(kind).GroupKind()
}

func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

var (
	SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
	AddToScheme = SchemeBuilder.AddToScheme
)

func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&VirtualMachine{},
		&VirtualMachineList{},
	)
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}
```

#### hack/k8s/boilerplate.go.txt
code-generator 自動將 boilerplate.go.txt 的文字，新增到產生的程式碼檔案內容的最上層。通常為 License 樣板。如以下範例:

```txt
/*
Copyright © 2019 The controller101 Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/
```

#### hack/k8s/tools.go
確保`go mod`能夠 code-generator 視為相依套件。

```go
// +build tools

// This package imports things required by build scripts, to force `go mod` to see them as dependencies
// See https://github.com/golang/go/issues/25922

package tools

import (
	_ "k8s.io/code-generator"
)
```

#### hack/k8s/update-generated.sh
用於執行 code-generator 腳本，以產生自定義資源的 Deepcopy、Client、Informer 與 Lister 程式碼。

```sh
#!/usr/bin/env bash

set -o errexit
set -o nounset
set -o pipefail

SCRIPT_ROOT=$(dirname "${BASH_SOURCE[0]}")/../..
CODEGEN_PKG=${CODEGEN_PKG:-$(cd "${SCRIPT_ROOT}"; ls -d -1 ./vendor/k8s.io/code-generator 2>/dev/null || echo ../code-generator)}
bash "${CODEGEN_PKG}"/generate-groups.sh "deepcopy,client,informer,lister" \
  github.com/cloud-native-taiwan/controller101/pkg/generated \
  github.com/cloud-native-taiwan/controller101/pkg/apis \
  "cloudnative:v1alpha1" \
  --output-base "$(dirname ${BASH_SOURCE})/../../../../../" \
  --go-header-file ${SCRIPT_ROOT}/hack/k8s/boilerplate.go.txt
```

#### hack/verify-codegen.sh
透過 diff 檢查當前的程式碼是否已經依據 apis 定義的內容產生。

```bash
#!/usr/bin/env bash

set -o errexit
set -o nounset
set -o pipefail

SCRIPT_ROOT=$(dirname "${BASH_SOURCE[0]}")/../..

DIFFROOT="${SCRIPT_ROOT}/pkg"
TMP_DIFFROOT="${SCRIPT_ROOT}/_tmp/pkg"
_tmp="${SCRIPT_ROOT}/_tmp"

cleanup() {
  rm -rf "${_tmp}"
}
trap "cleanup" EXIT SIGINT

cleanup

mkdir -p "${TMP_DIFFROOT}"
cp -a "${DIFFROOT}"/* "${TMP_DIFFROOT}"

"${SCRIPT_ROOT}/hack/k8s/update-generated.sh"
echo "diffing ${DIFFROOT} against freshly generated codegen"
ret=0
diff -Naupr "${DIFFROOT}" "${TMP_DIFFROOT}" || ret=$?
cp -a "${TMP_DIFFROOT}"/* "${DIFFROOT}"
if [[ $ret -eq 0 ]]
then
  echo "${DIFFROOT} up to date."
else
  echo "${DIFFROOT} is out of date. Please run hack/k8s/update-generated.sh"
  exit 1
fi
```

### Generate codes
當所有用於產生程式碼的檔案都建立後，就能透過 code-generator 依據定義的內容，來產生相關程式碼。由於開發使用 Go mod 管理套件，因此需要執行以下指令來完成程式碼產生:

```sh
$ go mod vendor
$ ./hack/k8s/update-generated.sh
Generating deepcopy funcs
Generating clientset for cloudnative:v1alpha1 at github.com/cloud-native-taiwan/controller101/pkg/generated/clientset
Generating listers for cloudnative:v1alpha1 at github.com/cloud-native-taiwan/controller101/pkg/generated/listers
Generating informers for cloudnative:v1alpha1 at github.com/cloud-native-taiwan/controller101/pkg/generated/informers
```

完成後，即可看到以下檔案:

```
pkg/generated
├── clientset
│   └── versioned
│       ├── fake
│       ├── scheme
│       └── typed
│           └── cloudnative
│               └── v1alpha1
│                   └── fake
├── informers
│   └── externalversions
│       ├── cloudnative
│       │   └── v1alpha1
│       └── internalinterfaces
└── listers
    └── cloudnative
        └── v1alpha1
```

這樣就能夠在自定義控制器中，透過 Client 程式碼直接操作 VirtualMachine API 了。

## 結語
今天主要了解如何自己定義 API 資源類型，並利用 code-generator 產生相關程式碼，以利我們在自定義控制器中使用。明天將透過這些定義與建立的程式碼，實際進行操作自定義 API 資源，並撰寫控制器程式。

## Reference
- https://github.com/kubernetes/sample-controller
- https://github.com/kubernetes/code-generator
- https://itnext.io/how-to-generate-client-codes-for-kubernetes-custom-resource-definitions-crd-b4b9907769ba
- https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/
- http://blog.xbblfz.site/2018/09/19/k8s%E4%BB%A3%E7%A0%81%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E8%BF%87%E7%A8%8B%E7%9A%84%E8%A7%A3%E6%9E%90/
- https://rancher.com/blog/2018/2018-07-09-rancher-management-plane-architecture/