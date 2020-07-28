---
title: "淺談 Kubernetes 的部署工具選擇 Part3"
date: 2019-9-18
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
在昨天文章中，我針對`開發環境(Development)`與`測試環境(Testing)`使用情境下，選擇 Kubernetes 工具的看法，而今天將延續 [淺談 Kubernetes 的部署工具選擇 Part2](https://k2r2bai.com/2019/09/17/ironman2020/day02/) 未講完的部分繼續分享`預備環境(Staging)`與`生產環境(Production)`看法。

<!--more-->

在開始談預備環境與生產環境上，我如何選擇部署工具前，先來回顧一下我在公司聽到關於這兩者的幾句經典話:

* 敏感與重要的系統與程式，就直接在生產環境測試就好。
* 先蓋好生產環境再去考慮預備環境，趕緊上線比較重要。
* 東西自己測完，就可以上生產環境了阿，不用等人拉。

然後這幾句話換來的結果就是。

![](http://giphygifs.s3.amazonaws.com/media/IVIRMpSAlSnHq/giphy.gif)

回想過去參與某些專案時，經歷的幾個駭人狀況:

* 有人在生產環境開發 API，不小心把所有使用者的物件儲存 Buckets 砍掉了。
* 又或者不小心在開發時，誤刪生產環境的 DB 虛擬機。
* 有人被要求在生產環境上，開發與測試要整合的外部防火牆，結果一個不小心就把防火牆的 API Request 上限打爆了，導致整個卡住無法正常運作。
* 有人把測試用與正式用的資源都放在生產環境上，所以測試結束後，刪除時，發現只剩下測試用。

諸如此類問題，三不五時就會聽聞與遭遇，這也讓我對於這種情況越來越重視，或許不是每間公司都有資源可以分多個環境階段來預防服務上線前的問題，但適當地切分還是能有效避免一些不該發生問題。不過上面這些只是題外話，總之我們開始進入主題吧。

## 預備環境(Staging)
當我們開發的程式經過測試環境的單元測試、煙霧測試、E2E 測試(或整合測試)，並將程式轉為發布階段時，就會利用 CI/CD 平台(工具)部署至預備環境(或者 QA 自行部署)，然後由內部 QA 或軟體測試人員進行各種上線時，可能會發生的情境或者模擬客戶使用的狀況。那在這樣的環境下，該如何選擇 Kubernetes 部署工具呢?

> 有時候我們會疑惑測試環境與預備環境的差異，因為這兩者都被用來做測試的環境。我自己的理解大概是這樣:
> * **Testing**:是`開發人員`驗證程式的各種測試的環境，開發人員可以在程式轉到發行版本前，測試任何程式的變更或錯誤修復。
> * **Staging**:比較偏向專給 `QA` 或`軟體測試人員`對開發人員發布的程式使用。在這環境中，會盡可能去模擬與實踐 Production 環境的各種事情，以確保上線的品質。也可以想像預備環境就是生產環境的`副本`。

事實上，選擇條件類似先前一些提到的，但是針對某幾點特別再去思考:

* **支援高可靠(Highly Available)部署**: 在生產環境中，高可靠架構是非常重要的一件事情，因為任何機器或 Kubernetes 節點在發生故障時，而使 Kubernetes 整個停止運作，繼而影響執行的應用程式。然而，由於預備環境必須從基礎建設到應用程式，都要盡可能模擬生產環境狀況，因此建立一座最小高可靠叢集是必須的。若部署工具對於高可靠架構支援程度太差的話，將有可能增加維運人員的負擔與風險。
* **支援更新叢集元件**: 由於預備環境跟生產環境不可能隨便就打掉重練，大多數情況下，會以升級當前叢集方式來進行，但升級 Kubernetes 叢集需要考慮到很多層面問題。因此若工具本身不會幫忙檢查，並實踐安全更新機制的話，可能在未來新版本釋出，舊版本進入 EoL 時，會有一些風險存在(如當前版本有重大資安問題，但卻無法更新)。
* **方便新增/刪除節點**: 同上類似狀況。但有時候節點發生異常時，可能需要將該節點轉成維運模式(或從叢集中移除)，並進行修復動作，直到確認正常後，再重新加入原本的 Kubernetes 叢集。當遇到這種情境時，若工具本身不支援這樣需求的話，就會發生如前面幾點一樣問題。
* **自動化**: 由於生產環境有可能需要部署大量機器，所以使用純 kubeadm 這種方式的話，維運人員就可能會面臨執行重複指令的時間成本增長問題，也可能發生執行錯誤指令狀況，因此一個部署工具是否能夠對多節點進行操作，並盡可能保持一致性是很重要的。
* **可客製化**: 在自建(或地端)部署時，有可能需要進行一些調整，或實現離線安裝需求，若工具客製化程度不高的話，可能會影響到部署的結果。
* **通過 [CNCF Kubernetes Conformance](https://github.com/cncf/k8s-conformance)**: 同 Testing 環境，雖然通過不代表就完全沒問題，但至少能對該工具多一點信心。
* **最佳實踐(Best Practices)**: 能夠盡可能依據當前 Kubernetes 版本的功能做最佳實踐，確保不會在預設下開啟實驗中或已棄用的功能。

那麼從上面這些條件中過濾，有哪些可以選擇呢?我個人大概是使用以下這些:

* [Kubespray](https://github.com/kubernetes-sigs/kubespray)
* [Kops](https://github.com/kubernetes/kops)
* [Rancher](https://github.com/rancher/rancher)
* [Breeze](https://github.com/wise2c-devops/breeze)
* [KubeOne](https://github.com/kubermatic/kubeone)
* [Puppetlabs Kubernetes](https://github.com/puppetlabs/puppetlabs-kubernetes)

> 其實還有很多可以選擇，但這邊比較偏向以自己使用過為主，若有興趣的人可以到 [CNCF Landscape](https://landscape.cncf.io/category=certified-kubernetes-installer&format=card-mode) 查看經過一致性認證的部署工具。

不過這麼多究竟哪個是最好呢?

事實上，沒有所謂最好最合適的部署工具，因為我通常還是依據情況來選擇，例如:

* 若環境都是裸機節點時，就會利用 Kubespray 以 Ansible 機制來自動化安裝多節點。且 Kubespray 可客製化程度很高，學習門檻也相對比一個用程式碼撰寫的工具來得低。

> 在裸機部署時，往往需要連同作業系統要一起安裝，但若叢集的節點過多的話，就會增加維運負擔，因此我還會結合 PXE/iPXE 以 Network boot 方式安裝作業系統，因為裸機自建環境不像公有雲提供的服務一樣，點一點就能部署起來。

* 若環境是在公有雲上自建 Kubernetes 時，就會偏向使用 Kops 或 KubeOne 工具來達成，因為這些工具整合了公有雲服務，可以方便幫你快速進行公有雲最佳實踐。
* 若希望有管理 UI 的話，那選擇 Rancher 或 Breeze 會是一個不錯方案，尤其是 Rancher 提供了非常完善的 Web-based 管理與監控介面，且能對多個 Kubernetes 叢集進行管理，並整合許多原生 Kubernetes 沒有的功能。

## 生產環境(Production)
當預備環境上的服務經過 QA 與軟體測試人員嚴厲的測試後，就會進入部署生產環境的階段，這時在生產環境上的工具該如何選擇呢?

在上一小節有提到預備環境可以看作是`生產環境的副本`，因此所有預備環境的條件都要去考量。但要額外注意的是`升級叢集`時，要確保使用工具一次執行的節點數是否正確，因為要是一次太多節點升級發生故障，且沒有多餘節點支撐服務運作的話，將會帶來嚴重的損失。

在這部分，我較多情況下是使用 Kubespray 與 Rancher 結合使用。部署 Kubernetes 叢集會以 Kubespray 為基礎來自行調整與修改，而 Rancher 則作為管理介面來管理 Kubespray 部署的 Kubernetes 叢集。

## 結語
這幾天都是分享自己選擇部署工具的看法與經驗，但老實說沒有哪個一個工具是最好的，終究還是依據公司跟團隊的需求來找出最適合的。不過若公司真的想要搞自建(或地端)部署的話，尤其是要幫客戶提供部署服務，我蠻建議手動去嘗試 [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) 的過程，至少可以讓自己對於部署 Kubernetes 的過程有更近一步的認知，這樣才不會遇到部署有問題時無從下手。

而若團隊在部署方面有很多經驗的話，那基於 Kubernetes The Hard Way 與 kubeadm 來開發一套公司專用的工具也是一個選擇。像是我們也有這樣做，透過分析 kubeadm 與手動部署的流程來了解每個部署過程，並以 Ansible 或 Puppet 方式來實現自動化。只是若開發都是由某一個人貢獻的話，那麼他離開時，就會面臨沒人維護問題。

## Reference
- https://chrislema.com/staging-environment/
- https://dzone.com/articles/13-reasons-why-staging-environment-is-failing-for-1
- https://www.quora.com/What-is-difference-between-testing-environment-and-staging
- https://kubernetes.io/docs/setup/best-practices/
- https://techbeacon.com/enterprise-it/6-best-practices-highly-available-kubernetes-clusters
- https://blog.sqreen.com/kubernetes-security-best-practices/
- https://www.cio.com/article/3411994/kubernetes-security-best-practices-for-enterprise-deployment.html