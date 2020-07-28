---
title: "實作 Kubernetes 外部認證系統整合: 以 LDAP 為例"
date: 2019-9-29
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
在一座 Kubernetes 叢集中，通常都會透過不同的使用者來給予不同的存取權限，因為若讓任何人擁有叢集最高權限的話，有可能帶來一些風險。而在 Kubernetes 中都會有兩種類型的使用者:

* 由 Kubernetes 管理的服務帳號(Service Account)。
* 普通使用者。

假設普通使用者是由外部獨立系統進行管理(如 LDAP)，那麼管理員分散私鑰、儲存使用者資訊等等功能，都必須由外部系統處理，因為在這方面，Kubernetes 並沒有普通使用者的 API 物件可以使用，因此無法透過 API 將普通使用者資訊添加到叢集中。

<!--more-->

但在 Kubernetes 生產環境中，管理普通使用者需求是很常見的需求，假設公司又希望讓管理使用者事情，由既有的帳戶系統管理的話，就會面臨問題。好在 Kubernetes 在這方面也都考慮到了，Kubernetes 提供了 [Webhook Token Authentication](https://kubernetes.io/docs/admin/authentication/#webhook-token-authentication) 與 [Authenticating Proxy](https://kubernetes.io/docs/admin/authentication/#authenticating-proxy) 機制讓我們可以跟既有系統整合。

> TODO: 補 Webhook 細節。

## 以 LDAP 作為 Kubernetes 身份認證
本節以 LDAP 為例來實現身份認證整合。由於 Kubernetes 官方並沒有針對 LDAP/AD 的整合，因此需要藉由 Webhook Token 方式來達成。這邊概念上會開發一個 HTTP Server 提供認證 APIs，當 Kubernetes API Server 收到認證請求時，會轉發至認證用的 HTTP Server 上，這時 HTTP Server 會利用 LDAP client 檢索符合認證的 User 資訊，並將該 User 的 Group 回傳給 API Server，最後 API Server 以該資訊來進行認證授權。

![](https://i.imgur.com/7Fb4saO.png)

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
|172.22.132.150| deploy-node | 4   | 16G    |LDAP Server|

> 節點不需要這麼多，這邊只是沿用。

### 事前準備
在開始部署時，請確保滿足以下條件:

* `LDAP Server`節點需要安裝 Docker 容器引擎:

```sh
$ curl -fsSL "https://get.docker.com/" | sh
```

* 部署一座 Kubernetes v1.10+ 叢集。可參考[用 kubeadm 部署 Kubernetes 叢集](https://kairen.github.io/2016/09/29/kubernetes/deploy/kubeadm/)。

### OpenLDAP 與 phpLDAPadmin 部署
本部分說明如何部署、設定與操作 OpenLDAP。首先進入`ldap-server`節點，接著利用容器部署 OpenLDAP 與 phpLDAPadmin:

```sh
$ docker run -d \
    -p 389:389 -p 636:636 \
    --env LDAP_ORGANISATION="Kubernetes LDAP" \
    --env LDAP_DOMAIN="k8s.com" \
    --env LDAP_ADMIN_PASSWORD="password" \
    --env LDAP_CONFIG_PASSWORD="password" \
    --name openldap-server \
    osixia/openldap:1.2.0

$ docker run -d \
    -p 443:443 \
    --env PHPLDAPADMIN_LDAP_HOSTS=172.22.132.150 \
    --name phpldapadmin \
    osixia/phpldapadmin:0.7.1
```

> * 這邊的`cn=admin,dc=k8s,dc=com`為`admin` DN，而`cn=admin,cn=config`為`config` DN。
> * 另外這邊僅做測試用，故沒有使用 Persistent Volumes，若需要的話，可以參考 [Docker OpenLDAP](https://github.com/osixia/docker-openldap) 來設定。

執行完成後，就可以透過瀏覽器來 [phpLDAPadmin](https://172.22.132.150/)。這邊點選`Login`輸入 DN 與 Password。成功登入後畫面，就可以自行新增其他資訊。

![](https://i.imgur.com/JBJ86LQ.png)

雖然可以直接利用 phpLDAPadmin 來新增跟 Kubernetes 整合的資訊，但為了操作快速，這邊以指令方式進行。

#### 建立 Kubenretes Token Schema
在`ldap-server`節點透過 Docker 進入`openldap-server`容器，然後執行以下指令建立 Kubernetes token schema 設定:

```sh
$ docker exec -ti openldap-server sh
$ mkdir ~/kubernetes_tokens
$ cat <<EOF > ~/kubernetes_tokens/kubernetesToken.schema
attributeType ( 1.3.6.1.4.1.18171.2.1.8
        NAME 'kubernetesToken'
        DESC 'Kubernetes authentication token'
        EQUALITY caseExactIA5Match
        SUBSTR caseExactIA5SubstringsMatch
        SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 SINGLE-VALUE )

objectClass ( 1.3.6.1.4.1.18171.2.3
        NAME 'kubernetesAuthenticationObject'
        DESC 'Object that may authenticate to a Kubernetes cluster'
        AUXILIARY
        MUST kubernetesToken )
EOF

$ echo "include /root/kubernetes_tokens/kubernetesToken.schema" > ~/kubernetes_tokens/schema_convert.conf
$ slaptest -f ~/kubernetes_tokens/schema_convert.conf -F ~/kubernetes_tokens
config file testing succeeded
```

然後執行以下指令來修改內容:
```sh
$ vim ~/kubernetes_tokens/cn=config/cn=schema/cn\=\{0\}kubernetestoken.ldif
# AUTO-GENERATED FILE - DO NOT EDIT!! Use ldapmodify.
# CRC32 e502306e
dn: cn=kubernetestoken,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: kubernetestoken
olcAttributeTypes: {0}( 1.3.6.1.4.1.18171.2.1.8 NAME 'kubernetesToken' DESC
 'Kubernetes authentication token' EQUALITY caseExactIA5Match SUBSTR caseExa
 ctIA5SubstringsMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 SINGLE-VALUE )
olcObjectClasses: {0}( 1.3.6.1.4.1.18171.2.3 NAME 'kubernetesAuthenticationO
 bject' DESC 'Object that may authenticate to a Kubernetes cluster' AUXILIAR
 Y MUST kubernetesToken )
```

接著利用 ldapadd 指令將 Kubernetes token schema 物件新增到當前 LDAP 伺服器中:

```sh
$ cd ~/kubernetes_tokens/cn=config/cn=schema
$ ldapadd -c -Y EXTERNAL -H ldapi:/// -f cn\=\{0\}kubernetestoken.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
adding new entry "cn=kubernetestoken,cn=schema,cn=config"
```
 
完成後，透過 ldapsearch 指令查詢是否有正確新增 Entry:

```sh
$ ldapsearch -x -H ldap:/// -LLL -D "cn=admin,cn=config" -w password -b "cn=schema,cn=config" "(objectClass=olcSchemaConfig)" dn -Z
Enter LDAP Password:
dn: cn=schema,cn=config
...
dn: cn={14}kubernetestoken,cn=schema,cn=config
```

#### 新增測試用 LDAP Groups 與 Users
一但 Kubernetes token schema 建立完成後，就能夠新增一些測試用 Groups 來模擬。這邊一樣在`openldap-server`容器中執行:

```sh
$ cat <<EOF > groups.ldif
dn: ou=People,dc=k8s,dc=com
ou: People
objectClass: top
objectClass: organizationalUnit
description: Parent object of all UNIX accounts

dn: ou=Groups,dc=k8s,dc=com
ou: Groups
objectClass: top
objectClass: organizationalUnit
description: Parent object of all UNIX groups

dn: cn=kubernetes,ou=Groups,dc=k8s,dc=com
cn: kubernetes
gidnumber: 100
memberuid: user1
memberuid: user2
objectclass: posixGroup
objectclass: top
EOF

$ ldapmodify -x -a -H ldap:// -D "cn=admin,dc=k8s,dc=com" -w password -f groups.ldif
adding new entry "ou=People,dc=k8s,dc=com"

adding new entry "ou=Groups,dc=k8s,dc=com"

adding new entry "cn=kubernetes,ou=Groups,dc=k8s,dc=com"
```

當 Group 建立完成後，再接著建立 Users 資訊:

```sh
$ cat <<EOF > users.ldif
dn: uid=user1,ou=People,dc=k8s,dc=com
cn: user1
gidnumber: 100
givenname: user1
homedirectory: /home/users/user1
loginshell: /bin/sh
objectclass: inetOrgPerson
objectclass: posixAccount
objectclass: top
objectClass: shadowAccount
objectClass: organizationalPerson
sn: user1
uid: user1
uidnumber: 1000
userpassword: user1

dn: uid=user2,ou=People,dc=k8s,dc=com
homedirectory: /home/users/user2
loginshell: /bin/sh
objectclass: inetOrgPerson
objectclass: posixAccount
objectclass: top
objectClass: shadowAccount
objectClass: organizationalPerson
cn: user2
givenname: user2
sn: user2
uid: user2
uidnumber: 1001
gidnumber: 100
userpassword: user2
EOF

$ ldapmodify -x -a -H ldap:// -D "cn=admin,dc=k8s,dc=com" -w password -f users.ldif
adding new entry "uid=user1,ou=People,dc=k8s,dc=com"

adding new entry "uid=user2,ou=People,dc=k8s,dc=com"
```

完成後，就可以透過指令或是登入 phpLDAPadmin 頁面查看資訊。如下圖所示。

![](https://i.imgur.com/mRqk5F8.png)


#### 新增 Kubernetes Token 至 Users
當 Users 建立完成後，就可以透過執行以下指令來新增每個 User 的 Kubernetes Token:

```sh
$ cat <<EOF > users.txt
dn: uid=user1,ou=People,dc=k8s,dc=com
dn: uid=user2,ou=People,dc=k8s,dc=com
EOF

# 新增 token 腳本指令
$ while read -r user; do
fname=$(echo $user | grep -E -o "uid=[a-z0-9]+" | cut -d"=" -f2)
token=$(dd if=/dev/urandom bs=128 count=1 2>/dev/null | base64 | tr -d "=+/" | dd bs=32 count=1 2>/dev/null)
cat << EOF > "${fname}.ldif"
$user
changetype: modify
add: objectClass
objectclass: kubernetesAuthenticationObject
-
add: kubernetesToken
kubernetesToken: $token
EOF

ldapmodify -a -H ldapi:/// -D "cn=admin,dc=k8s,dc=com" -w password  -f "${fname}.ldif"
done < users.txt

# output
Enter LDAP Password:
modifying entry "uid=user1,ou=Users,dc=k8s,dc=com"

Enter LDAP Password:
modifying entry "uid=user2,ou=Users,dc=k8s,dc=com"
```

### 部署 LDAP Webhook
當 OpenLDAP 都完成，且 Kubernetes 叢集也建立完成後，就可以進入任一 Kubernetes 主節點部署 LDAP Webhook，這邊透過 Git 取得:

```sh
$ git clone https://github.com/kairen/kube-ldap-authn.git
$ cd kube-ldap-authn
```

> Golang 版本可以參考 [kube-ldap-webhook](https://github.com/kairen/kube-ldap-webhook)。

新增一個`config.py`檔案，並設定查詢時需要的相關內容：
```sh
LDAP_URL='ldap://172.22.132.150/ ldap://172.22.132.150'
LDAP_START_TLS = False
LDAP_BIND_DN = 'cn=admin,dc=k8s,dc=com'
LDAP_BIND_PASSWORD = 'password'
LDAP_USER_NAME_ATTRIBUTE = 'uid'
LDAP_USER_UID_ATTRIBUTE = 'uidNumber'
LDAP_USER_SEARCH_BASE = 'ou=People,dc=k8s,dc=com'
LDAP_USER_SEARCH_FILTER = "(&(kubernetesToken={token}))"
LDAP_GROUP_NAME_ATTRIBUTE = 'cn'
LDAP_GROUP_SEARCH_BASE = 'ou=Groups,dc=k8s,dc=com'
LDAP_GROUP_SEARCH_FILTER = '(|(&(objectClass=posixGroup)(memberUid={username}))(&(member={dn})(objectClass=groupOfNames)))'
```

> 可以參考 [Config example](https://github.com/kairen/kube-ldap-authn/blob/master/config.py.example) 查看詳細變數說明。

接著將上述的設定檔以 Secret 方式上傳至 Kubernetes 叢集中，然後部署 LDAP webhook 的 DaemonSet 到所有主節點上:

```sh
$ kubectl -n kube-system create secret generic ldap-authn-config --from-file=config.py=config.py
$ kubectl create -f daemonset.yaml
$ kubectl -n kube-system get po -l app=kube-ldap-authn -o wide
NAME                    READY     STATUS    RESTARTS   AGE       IP             NODE
kube-ldap-authn-sx994   1/1       Running   0          13s       192.16.35.11   k8s-m1
...
```

> 部署到所有主節點是在 HA 架構中進行，因為呼叫 API 時，有可能會因為負載平衡關析，而導到不同節點上，這時若沒有在每個節點設定 Webhook 的話，就會認證失敗。

部署成功後，就可以透過 cURL 工具來測試:

```sh
$ curl -X POST -H "Content-Type: application/json" \
    -d '{"apiVersion": "authentication.k8s.io/v1beta1", "kind": "TokenReview",  "spec": {"token": "<LDAP_K8S_TOKEN>"}}' \
    http://localhost:8087/authn

# output
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "status": {
    "authenticated": true,
    "user": {
      "groups": [
        "kubernetes"
      ],
      "uid": "1000",
      "username": "user1"
    }
  }
}
```

確認沒問題後，接著在所有主節點上，新增`/srv/kubernetes/webhook-authn`檔案，並加入以下內容:

```sh
$ mkdir /srv/kubernetes
$ cat <<EOF > /srv/kubernetes/webhook-authn
clusters:
  - name: ldap-authn
    cluster:
      server: http://localhost:8087/authn
users:
  - name: apiserver
current-context: webhook
contexts:
- context:
    cluster: ldap-authn
    user: apiserver
  name: webhook
EOF
```

完成後，修改所有主節點的`/etc/kubernetes/manifests`目錄底下的`kube-apiserver.yaml`檔案，其內容修改成如下:

```yaml
...
spec:
  containers:
  - command:
    ...
    - --runtime-config=authentication.k8s.io/v1beta1=true
    - --authentication-token-webhook-config-file=/srv/kubernetes/webhook-authn
    - --authentication-token-webhook-cache-ttl=5m
    volumeMounts:
      ...
    - mountPath: /srv/kubernetes/webhook-authn
      name: webhook-authn
      readOnly: true
  volumes:
    ...
  - hostPath:
      path: /srv/kubernetes/webhook-authn
      type: File
    name: webhook-authn
```

> 這邊`...`表示已存在的內容，請不要刪除與變更。

### 測試功能
都完成部署後，就可以進入任一主節點進行測試。這邊建立一個綁定在 user1 Namespace 的 Role 與 RoleBinding 來提供權限測試:

```sh
$ kubectl create ns user1

# 建立 Role
$ cat <<EOF | kubectl create -f -
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: readonly-role
  namespace: user1
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
EOF

# 建立 RoleBinding
$ cat <<EOF | kubectl create -f -
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: readonly-role-binding
  namespace: user1
subjects:
- kind: Group
  name: kubernetes
  apiGroup: ""
roleRef:
  kind: Role
  name: readonly-role
  apiGroup: ""
EOF
```

> 在 RoleBinding 中的 subjects 需要對應於 LDAP 中的 Group 資訊。

接著在任一台操作端設定 Kubeconfig 來以 user1 使用者，存取叢集:

```sh
$ cd
$ kubectl config set-credentials user1 --kubeconfig=.kube/config --token=<user-ldap-token>
$ kubectl config set-context user1-context \
    --kubeconfig=.kube/config \
    --cluster=kubernetes \
    --namespace=user1 --user=user1
```

透過 kubectl 來測試權限是否正確設定:

```sh
$ kubectl --context=user1-context get po
No resources found

$ kubectl --context=user1-context run nginx --image nginx --port 80
Error from server (Forbidden): deployments.extensions is forbidden: User "user1" cannot create deployments.extensions in the namespace "user1"

$ kubectl --context=user1-context get po -n default
Error from server (Forbidden): pods is forbidden: User "user1" cannot list pods in the namespace "default"
```

## 結語
今天簡單實作了 Kubernetes 整合外部認證系統的功能，讓我們能夠以 LDAP 方式來管理 Kubernetes 的普通使用者。可以發現 Kubernetes 在各種方面都考慮了許多擴充方式，不只是網路、儲存等等，在認證與授權部分也提供了一些 API 與機制來實現。現在也有很多開源專案實作了 Auth Webhook 來整合認證，如以下:

- [Dex](https://github.com/dexidp/dex)
- [Kubehook](https://github.com/planetlabs/kubehook)
- [OpenStack Keystone](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-keystone-webhook-authenticator-and-authorizer.md)
- [Guard](https://github.com/appscode/guard)


## Reference
- https://github.com/osixia/docker-openldap
- https://icicimov.github.io/blog/virtualization/Kubernetes-LDAP-Authentication/
- https://github.com/torchbox/kube-ldap-authn
- https://superuser.openstack.org/articles/strengthening-open-infrastructure-integrating-openstack-and-kubernetes/
- https://kubernetes.io/docs/reference/access-authn-authz/webhook/