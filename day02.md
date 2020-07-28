---
title: "淺談 Kubernetes 的部署工具選擇 Part2"
date: 2019-9-17
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
上一篇 [淺談 Kubernetes 的部署工具選擇 Part1](https://k2r2bai.com/2019/09/16/ironman2020/day01/) 提到選擇部署工具時，需要思考的幾件事情，並簡單提了一下目前 Kubernetes SIG cluster lifecycle 的發展。而今天我將依據不同環境與情境來說明自己怎麼選用 Kubernetes 部署工具。

> 原本這篇是要放在第一天一起講，但是發現一天超過 6000 字似乎有點壓力... 感覺三天後我就放推了，但因為是團隊報名，所以不能放棄!!!只好拆成三篇來分享。

<!--more-->

接下來的內容，我將分成以下幾個用途來分享自己想法與如何選擇:

1. 開發環境(Development)
2. 測試環境(Testing)
3. 預備環境(Staging)
4. 生產環境(Production)

> 因為都只是自己想法，沒什麼參考價值，大家看看就好。

## 開發環境(Development)
在公司時，我比較常負責開發各種 Kubernetes 相關整合元件(比如說 Controller/Operator、CNI/CSI/Device Plugin、Authentication webhook 等等)，以及提供客戶想要容器化的應用程式，並放到 Kubernetes 中。而其中開發 Kubernetes 整合元件部分又以 Controller 居多，因為客戶希望能夠透過 Kubernetes 機制來實現一些新功能或是管理硬體資源(如透過 Kubernetes resource 來管理防火牆設備等等)，或者希望能夠跟 Kubernetes 既有的 API resources 直接整合，如當建立 Service 時，自動設定對應防火牆的規則。而諸如此類的工作事情，往往需要一座實際的 Kubernetes 叢集來測試與驗證開發的功能，這時該怎麼選擇呢?在開始選擇前，我們先來看看被用於開發與測試的 Kubernetes 部署工具有哪些，下面簡單列幾個專案與方案:

* [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
* [minikube](https://minikube.sigs.k8s.io)
* [kind](https://github.com/kubernetes-sigs/kind)
* [microk8s](https://github.com/ubuntu/microk8s)
* [Kubernetes-powered Docker CE](https://blog.docker.com/2018/01/docker-mac-kubernetes/)
* [kube-spawn](https://github.com/kinvolk/kube-spawn)
* [virtuakube](https://github.com/danderson/virtuakube)
* [Simplekube](https://github.com/valentin2105/Simplekube)

> 除了這些，大家也可以看看 [awesome-kubernetes](https://github.com/ramitsurana/awesome-kubernetes) 中的列表來查看其他工具。

大家會發現有這麼工具多可以使用，那是不是只要自己用的爽，或是反正團隊想用什麼就用什麼，這樣就好了?

答案是『Yes』，但我還是會進一步去考慮一些條件，比如說:

1. **由Kubernetes 社區或某公司維護**: 選擇社區與公司維護的原因在於，往往一個專案若沒有組織維護時，可能會因為本身專案不夠大(或者自身工作太忙碌)，而願意投入的開發者相對少，甚至只有自己時，就會無法持續跟進與改進，這樣時間久了自然就不會在維護。而這點尤其選擇`Kubernetes 社區`維護為佳。
2. **可客製化程度**: 能夠客製化部署的叢集好處在於能夠對一些特別的功能進行驗證，因為 Kubernetes 有些功能在不同版本不是都預設開啟，而在這種情況下，若使用的工具本身難以調整時，就會帶來一些麻煩。另外不同 Kubernetes 版本的支援，是希望確保開發的程式，能夠在同一機器驗證不同版本的執行結果，以降低不必要的時間浪費。
3. **是否支援多節點(Multi-node)環境**: 雖然開發環境大多情況下用單節點 Kubernetes 就能夠完成，但如果遇到是開發 Custom Scheduler 或者特殊應用時，能夠模擬多節點的 Kubernetes 叢集就極為重要。
4. **是否會污染開發機器的環境**: 大多情況下，我不太喜歡因為開發某個程式，就安裝一堆東西來污染整個環境，因為有些工具會安裝過多相依元件，這時可能會造成開發環境的元件難以清理。老實說這是奇摩子問題。
5. **是否跨平台或作業系統**: 最主要是確保團隊成員不管是使用哪個平台(或作業系統)進行開發，都能夠用同一套工具來驗證開發的結果，以保證團隊驗證環境的一致性。
6. **能 3 - 5 行搞定**: 使用工具本來就是為了降低繁雜的部署過程，因此能夠用最少的指令來完成那對於開發人員是最好不過，因為開發人員理論上不應該太專注於 Kubernetes 部署本身，而是要開發的功能。

考慮了這些條件後，我會快速地先過濾掉一些非社區或公司維護的工具，比如說上面列出來的 virtuakube、Simplekube 專案。相信大家點進去看過這些專案後，也會發現大多數個人維護的工具，都會在經過一段時間後，就慢慢沒有那麼頻繁維護，最後 EoL 並 Archived 整個程式碼專案。而當過濾掉這些後，就會進一步依據剩下條件來找出最佳的工具。

比如說用於測試開發程式或上手 Kubernetes 時，個人就特別推薦 minikube，因為它以下幾個好處:

* Kubernetes 社區維護，目前核心人員積極在改進功能。
* 能夠透過執行參數來改變 kube-apiserver、kube-controller-manager 與 kube-scheduler 等等元件的設定。
* 支援不同的平台(需要自己編譯)與作業系統。
* 提供不同 cluster bootstrapper。

> minikube 的 cluster bootstrapper 目前雖然只有 kubeadm，但是 Kubernetes 社區有在規劃 Free-VM(kind)的部署。

* 支援不同虛擬機驅動(VM driver)。也支援 None VM 部署，但僅限於 Linux。

但 minikube 最可惜之處，在於無法模擬多節點部署的 Kubernetes 叢集，因此沒辦法驗證一些開發的東西。

> 事實上也不是不能，畢竟是開源專案，只要稍微改程式就可以了，大家可以參考 [利用 Minikube 快速建立測試用多節點 Kubernetes 叢集](https://k2r2bai.com/2019/01/22/kubernetes/deploy/minikube-multi-node/) 這篇文章。另一方面 minikube 團隊也把多節點部署視為 2019 年希望支援的功能，也看到會議上有持續在討論這個內容，大家可以參考 [Add multi-node support ](https://github.com/kubernetes/minikube/issues/94)。

如果遇到多節點支援問題時，我就會選擇 kind 或 microk8s 作為替代方案，尤其是 kind 是由 Kubernetes 社區維護，其背後也是基於 kubeadm 的函式庫實現而成，因此能夠確保叢集是當前版本的最佳實踐。

這時有人會問說，為啥不直接用 kubeadm?上面很多工具都是基於它開發而成的啊。事情是這樣的，kubeadm 並不像其他工具能幫你建立執行的虛擬環境(如 VM 或 Container)，因此執行 kubeadm 的環境依然需要自己處理，而這過程會增加開發人員不必要的時間浪費，並且對開發這件事失焦。

> 當然喜歡用 [Vagrant](https://www.vagrantup.com/) 的人，也是可以自己寫個 Vagrantfile + Shell/BASH 腳本來部署。

那 Docker 內建的 Kubernetes 呢?Docker 能夠簡單又快速地建立開發與測試用的 Kubernetes 環境，也是知名公司維護阿。確實上是如此沒錯，但我沒有特別推薦用於開發是因為 Kubernetes 是 Docker 工具附加的功能，所有版本跟設定幾乎都是綁死於特定的 Docker 版本，因此我們很難依據個人需求，去調整部署的 Kubernetes 叢集參數與版本。

## 測試環境(Testing)
而當開發完功能後，往往會再額外撰寫單元測試(Unit tests)、煙霧測試(Smoke tests)與 E2E 測試(或整合測試)的程式，並使用持續性整合(Continuous Integration)平台自動化地在程式碼推送至 Git 時，執行團隊要求的測試步驟。比如說當單元測試完成後，會檢查測試覆蓋率與錯誤率是否符合要求，若符合的話，則接著進行 E2E 測試(或整合測試)，這時也需要實際部署一座用於測試的 Kubernetes 叢集來驗證功能。那我會怎麼做呢?

> 要不要把單元測試跟 E2E 測試(或整合測試)放一個階段執行要看團隊，後者往往需要部署一座 Kubernetes 叢集提供測試用，所以為了避免不必要的資源浪費，我比較偏向先跑完單元測試後，在判斷是否執行 E2E 測試(或整合測試)。

這部分我會依據以下幾件事來選擇使用的部署工具:

1. **確認使用的 CI 平台(工具)**: 由於開發的專案放置在不同 Git 儲存庫時，我會依據情況使用不同的 CI 來執行測試，比如說 GitHub 上就會用 Travis CI 來完成，但 Travis CI 無法讓我們透過 minikube 以虛擬機方式，建立一座測試用 Kubernetes 叢集，且使用 None driver 也可能影響到 Travis CI 提供的環境。這時我就會選用 `kind` 這種 Free-VM，但又有隔離性的工具來完成。當然若是自建的 CI 就另當別論，像是我比較喜歡在專案塞個 Vagrantfile 來完成。
2. **儘可能與開發環境使用的工具一致**: 確保開發環境若有對 Kubernetes 叢集做什麼更動時，測試環境也能透過同樣指令與設定來建立相同叢集。
3. **能夠保留/刪除舊叢集，以及快速建立新叢集**: 有時候我會利用上一次測試過的環境來執行新的 Commit，當測試完成後再刪除，並建立新叢集進行第二次測試。這是因為預備環境(Staging)或生產環境(Production)通常不會頻繁更動 Kubernetes 叢集，尤其是生產環境。
4. **通過 [CNCF Kubernetes Conformance](https://github.com/cncf/k8s-conformance)**: CNCF 的一致性測試會基於 [Sonobuoy](https://github.com/heptio/sonobuoy) 工具對 Kubernetes 叢集進行各種測試。若使用的 Kubernetes 部署工具通過測試的話，理論上能在被認證的其他方案與專案上轉移。

> 當然這前提是不考慮太多`網路`與`儲存`狀況。

事實上，在測試環境中，我使用的工具大多是 minikube、kind 與 kubeadm with vagrant。但測試功能需要一座小規模節點數的裸機叢集時，我就會透過結合 Ansible 這種工具來搭配 kubeadm。

## 結語
今天先分享自己在開發環境跟測試環境上的選擇，明天再進一步分享剩下的預備環境(Staging)與生產環境(Production)。老實說也沒有真的哪個方式是最好，只要能達成目標用什麼確實都可以。當然這是不考慮後續維護的前提下，若團隊長期會持續在 Kubernetes 上進行開發的話，我還是建議以 Kubernetes 社區維護的工具為主，至少能確保支援當前的 Kubernetes 的版本與最佳實踐。當然公司團隊有餘力的話，以 Kubernetes 社區的工具為基礎打造一套公司專用的 Kubernetes 部署工具也是不錯選擇。

> 其實原本想要今天也把預備環境(Staging)與生產環境(Production)寫完，但是發現好像只利用晚上時間有點困難...

## Reference
- http://k2r2bai.com
- https://www.hwchiu.com/travisci-k8s.html
- https://github.com/ramitsurana/awesome-kubernetes#installers
- https://blog.docker.com/2018/01/docker-mac-kubernetes/