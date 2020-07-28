---
title: "淺談 Kubernetes 的部署工具選擇 Part1"
date: 2019-9-16
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
我們都知道 Kubernetes 有很多特性可以確保系統與應用程式能夠更加穩定、部署更加流暢與管理更加簡單，而且使用起來也相對容易，但在自建(或地端)部署時，卻與想像的不同，有很多需要關注的問題與事情，例如:高可靠性(Highly Available，HA)、如何安全地更新叢集、裸機負載平衡器(Bare-metal Load Balancer)、動態 DNS 更新、硬體設備整合、分散式儲存整合與離線安裝(Offline Installation)等等，除了上述問題外，還要為了客戶能夠對部署的 Kubernetes 更有信心，因此必須進行壓力測試、E2E 測試、一致性測試與叢集安全掃描等等。從上述簡短的描述中，可以了解到自己蓋一套可以用的 Kubernetes 叢集，並且自己維運是多麼麻煩的事，這對公司來說都是成本支出。也正因此大多數人會偏向使用公有雲的 Kubernetes 服務，如 EKS、GKE 與 AKS 等等來減少維運成本的支出。下圖顯示不同部署的方式需要管理的部分。

<!--more-->

> 當然不是所有系統跟應用程式跑在 Kubernetes 就能活得好好的，整個好棒棒。

![](https://i.imgur.com/xMS4XfX.png)
(圖片擷取自：[kubernetes.io](https://kubernetes.io/docs/setup/))

<!--more-->

而在研替剛進去那一年，我因為在學期間有接觸過 Kubernetes 手動安裝，因此在還沒進入狀況下，被公司安排處理所有 Kubernetes 相關案子，這也使我從一開始對 Kubernetes 懵懵懂懂，到現在才覺得對 Kubernetes 有一點點認知(老實說還是一知半解XDD)。由於研替是在一家人員有限的公司，再加上大多數人對於 Kubernetes 不是那麼熟悉，作為一位軟體工程師的我，因此還需要兼顧不同角色，來解決幫助客戶自建的 Kubernetes 問題，因此在過程中學習到了一點點東西，而剛好有幸參加本次鐵人賽，想透過研替最後的 30 天來盡可能把經驗分享給大家。

而本系列文章將分為三大部分來分享經驗，分別為:

* On-Premise(Custom) Container 與 Kubernetes 經驗
* Controller 與 Operator 開發經驗
* Kubernetes Certification 經驗

### On-Premise(Custom) Container 與 Kubernetes 經驗
本部分將說明自己在 On-Premise(Custom) 部署的經驗，就如同前言提到的那些內容，我將說明自己在協助客戶自建 Kubernetes 時，使用到與學習到的東西，並帶大家了解如何實踐。

### Controller 與 Operator 開發經驗
本部分將說明如何開發 Kubernetes Controller/Operator，過程中將實現幾個範例來帶大家了解 Controller/Operator 原理，並教導如何正確使用 Finalizer、Leader election 等等機制與功能。

### Kubernetes Certification 經驗
本部分將說明如何準備與通過 `Certified Kubernetes Administrator`、`Certified Kubernetes Application Developer`、`Kubernetes Certified Service Provider` 與 `Certified Kubernetes Conformance Program`，這四項都是自己有經驗的部分。

> 由於認證題目與要求會隨著時間而改變，這邊只能盡可能提供一些 Tips 來幫助大家。

## 所以，Kubernetes 的部署工具該怎選擇?
第一次接觸 Kubernetes 時，大多數人都會思考該用什麼工具部署比較好，因為現在市面上有太多 Kubernentes 部署方案與開源專案，因此選擇時，往往無從下手，比如說下圖內的工具，相信就夠大家花很多時間去研究與嘗試了。

![Kubernetes Tools](https://i.imgur.com/C9fYkmj.png)  

而這麼多部署工具，究竟該怎麼選擇呢?哪一個才是最好的工具呢?

事實上，沒有一個部署工具與方案是適用於每一間公司的，因此在自建一座 Kubernetes 叢集前，我們必須依據需求，思考以下幾件事情:

1. Kubernetes 支援的版本
2. HA(Highly Available)
3. 能夠安全地更新叢集元件
4. 叢集部署規模
5. 是否會擴展節點(能輕易地新增與刪除節點)
6. 自動化與客製化程度
7. 部署叢集使用的情境與用途
8. 支援的網路插件(CNI Plugin)
9. 該工具是否有持續更新改進
10. 團隊有沒有共識(這也很重要)
11. 是否跨不同 CPU 指令集架構
12. 是否跨不同作業系統
13. 該工具是否通過一致性測試(CNCF Certified Kubernetes)

思考這幾點是因為部署完一座 Kubernetes 叢集後，接下來的維運才是真正的開始。當選擇了錯誤的工具與方案後，後續維運往往會受到影響，因此而增加困難性與複雜性，導致各種風險發生，比如說上述提到的 1 與 9 點，當一個工具因為無法跟上 Kubernetes 版本快速演進時，團隊就必須面臨更換的決策，而一但確定更換時，又必須面臨升級與遷移問題;又或者如 2 - 8 點沒考慮到時，隨便選擇一個工具就進行部署，結果該工具對於這些需求支援程度低，就會增加維運團隊的困難與時間成本，甚至面臨整組打掉重練的地步。

> Kubernetes 採用 [Semantic Versioning](http://semver.org/) 方式來管理版本，其中以次要版本號(Minor version)與補丁版本號(Patch version)為主。通常一個新的次要版本大約`3 個月`釋出，並持續維護約`9 個月`就會進入 EoL(End of Life)，而補丁版本則大約`2 - 3 週`釋出，除非遇到重大錯誤。

而過去 Kubernetes 社區裡一直沒有足夠的重視這些問題，所以一直沒有一套完整的工具來降低 Kubernetes 部署與維運的困難。一直到 2016 年才在社區貢獻者的推動下，發起了由 Kubernetes 社區維護的部署工具 [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)。

> kubeadm 是一位名為 Lucas Käldström 芬蘭人貢獻者發起，該貢獻者在當時僅是個高中生。我想這也是開源社區魅力所在，任何人都可以對社區進行貢獻，儘管貢獻的品質可能不是最好，但跟著成熟的開源社區中頂尖貢獻者持續地改善的話，終將會往好的方向發展。

而為何要提到 kubeadm 呢?

相信用過的人肯定會發現這工具真的很簡單，我們只需要幾個指令就可以部署一座 Kubernetes 叢集。kubeadm 設計非常精簡，又不失最佳實踐(Best Practices)理念，使大家會覺得很有『官方原生』的感覺，事實也是如此。既然官方有維護部署工具，那為什麼還有那麼多部署工具與方案呢?事情是這樣的，kubeadm 在過去版本雖然一直在實踐最佳的 Kubernetes 組態設定，但是卻缺乏了對生產環境重要的 HA 一環支援，因此許多人僅把這個工具當作測試用叢集部署使用。

然而經過 Kubernetes 社區持續的發展，kubeadm 在 2018 年 12 月的 v1.13 版本釋出時，正式進入 GA(Generally available) 階段，並被標示為 Production-Ready 的工具，且隨著後續版本的推進，對於 HA 的支援已經慢慢完善; 另一方面可以觀察到有些部署工具開始從自行實現部署方案，慢慢轉為以 kubeadm 為基礎進行部署與產生組態檔，比如說 kubespray、kops、KRIB、minikube、SUSE skuba 等等。

而為什麼以 kubeadm 為基礎最近部署與產生組態檔案比較好呢?我想這是因為 kubeadm 的架構設計能夠提供各種介面與方法來進行客製化 Kubernetes 從，如 phases 能讓使用者輕易地客製化每個部署階段，也正因為這樣的好處，所以上層工具開發者(商)只需要實現符合 kubeadm 的標準或 APIs 就能夠輕易改變叢集功能，此舉可以讓開發者(商)簡化複雜性，一來又能夠減少跟進新版的功能時間成本，直接藉由 Kubernetes 官方的最佳實踐，來保持與社區工具一致性。當然，這也是 kubeadm 維護團隊 SIG cluster lifecycle 團隊希望達到的，他們希望提供一系列工具鏈與 APIs 給大家開發自家的 Cluster provisioners，因此才會有越來越多工具往這方向發展，我想這樣發展是充滿信心的，畢竟自己在參與 Kubernetes 貢獻時，覺得 Kubernetes 是一個蠻成功的開源社區。

![SIG Cluster Lifecycle Projects](https://i.imgur.com/p9CAHTC.png)
(圖片擷取自：[KubeCon EU 2019](https://static.sched.com/hosted_files/kccnceu19/c4/2019%20KubeConEU%20-%20SCL-Intro.pdf))

![Built to be part of a higher-level solution](https://i.imgur.com/ZwYETAL.png)
(圖片擷取自：[KubeCon EU 2019](https://static.sched.com/hosted_files/kccnceu19/c4/2019%20KubeConEU%20-%20SCL-Intro.pdf))

今天僅是簡單說明選擇一個工具時，必須要思考幾件事情，並提一下目前社區的發展。或許有人看到這邊可能會想說:『現在開始選用 kubeadm 就對了?』，確實 kubeadm 能夠完成大多數事情，但當面臨不同規模的叢集時，還是要搭配不同工具與系統的結合，才能發揮最大效果，因為要考慮到部署與維運的時間成本，以及部署完節點一致性問題。比如說有大量節點需要部署作業系統，那該怎麼辦呢?因為自建叢集不像公有雲提供的服務一樣，能便利地建立所需的資源;又或者說節點太多，要如來確保節點部署的一致性呢?諸如此類問題是自建(地端)的 Kubernetes 叢集經常會遇到的，而下一篇我將依據不同情境來說明怎麼選擇工具，或如何利用 kubeadm 結合其他工具來盡可能解決，當然也會提一下自己實際上作法。

## Reference
- https://kubernetes.io/docs/setup/
- https://kccnceu19.sched.com/event/MPhI/intro-cluster-lifecycle-sig-lucas-kaldstrom-independent-tim-st-clair-vmware
- https://caylent.com/50-useful-kubernetes-tools
- https://github.com/ramitsurana/awesome-kubernetes#installers
- https://docs.google.com/spreadsheets/d/1LxSqBzjOxfGx3cmtZ4EbB_BGCxT_wlxW_xgHVVa23es/edit#gid=0
- https://kubernetes.io/docs/setup/release/version-skew-policy/
- https://github.com/kubernetes/community/blob/master/contributors/design-proposals/release/versioning.md#kubernetes-release-versioning
- https://kubernetes.io/blog/2019/06/24/automated-high-availability-in-kubeadm-v1.15-batteries-included-but-swappable/
- https://kubernetes.io/blog/2018/12/04/production-ready-kubernetes-cluster-creation-with-kubeadm/