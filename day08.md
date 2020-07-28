---
title: "分析 Kubeadm 叢集更新流程"
date: 2019-9-23
catalog: true
toc: true
categories:
- Kubernetes
- IT Ironman
tags:
- Kubernetes
---
## 前言
昨天分享了如何使用 Kubeadm 工具來升級既有叢集，過程中可以發現 Kubeadm 升級時，將許多複雜的工作做了許多簡化，因此讓不熟悉 Kubernetes 的人也能輕易達成叢集元件更新事情。雖然工具協助我們更簡單的達成某些功能是很好的事，但不是所有狀況都能透過 Kubeadm，假設在升級時，發生問題的話，要該怎麼辦呢?這時可能會因為不熟悉整個升級過程，而不知如何下手來解決問題。基於上述原因，今天就是要來分析 Kubeadm 的叢集更新是如何實作的，以讓我們能夠在發生問題時，更快的解決。

<!--more-->

## Kubeadm upgrade 流程
從昨天操作的流程中，我們了解到透過 Kubeadm 更新叢集的指令流程:

![](https://i.imgur.com/wzaRJuA.png)

圖中，我們會發現 kubeadm 最少用一個指令就完成更新的過程，但是事實上這一個指令背後做了許多事情，接下來各小節將針對圖中的指令進行解析。

### kubeadm upgrade plan
該指令主要檢查是否可以升級目前叢集到指定的 Kubernetes 版本，這過程會進行驗證以下幾件事情:

1. 檢查當前叢集中的 Kubeadm 組態檔。
2. 檢查當前叢集是否處於健康狀態。
3. 取得指定版本是否可以使用。若位指定版本的話，則以當前釋出的最新穩定版本。
4. 確認是否符合 [Version Skew Policies](https://kubernetes.io/docs/setup/release/version-skew-policy/)。
5. 若上面執行都沒問題的話，會顯示當前叢集元件版本，以及目標叢集元件版本。

這邊做的事情很簡單，主要都是檢查當前與目標版本狀態，並顯示資訊讓使用者知道狀況。

### kubeadm upgrade apply
一但檢查都沒問題後，我們就可以執行`kubeadm upgrade apply`來進行主節點的更新，這時 kubeadm 會開始進行以下流程:

1. 檢查組態參數是否有效，並載入預設參數與使用者輸入的參數。另外也會檢查當前是否為 root。
2. kubeadm 會透過 admin user 來與 API server 溝通，並檢查當前叢集是否處於健康狀態，以及所有節點是否處於 Ready 狀態。當檢查都完成後，會讀取叢集中的 kubeadm InitConfiguration 組態檔案，並將內容調整成新版的資訊後，更新至當前叢集中，以在後續執行更新時使用。另外該階段也會再次檢查是否符合 [Version Skew Policies](https://kubernetes.io/docs/setup/release/version-skew-policy/)。
3. 當上述確認後，kubeadm 會與 API server 溝通，以建立一個 DaemonSet 來取得指定版本的控制平面元件容器映像檔。這邊原理是利用 Kubernetes 機制來下載映像檔，當 Pod 被建立時，容器 Runtime 就會先下載映像檔才啟動，因此可以確保指定容器映像檔被載入到節點上。這也是為何前面需要確保節點處於 Ready 狀態原因。另外 Kubeadm 不實作直接從容器 Runtime 拉取映像檔，是因為現在有太多容器 Runtime 被使用，因此這麼做的話，會增加程式與維護的困難與複雜性。
4. 一但映像檔載入完成後，kubeadm 就會執行元件升級。首先會備份 etcd 的 mainifest 檔案，並更新成新版本內容，接著等待 etcd 啟動。接著會開始對控制元件進行更新，過程中跟 etcd 類似，會先寫入新版本 YAML 到 /tmp 底下，接著將新版本檔案放到 mainifest 目錄，然後將舊的 mainifest 檔案進行備份，最後等待監聽 kubelet 重新啟動 Static Pod 的變動。另外過程中，如果 kubeadm 檢查發現相關憑證將在 180 天過期的話，kubeadm 會自動更新相關 TLS 憑證，以確保叢集不會因為過期而出問題。
5. 若控制平面元件都升級完成的話，會開始進行以下步驟:
    * 將新版本的 kubeadm ClusterStatus 與 kubelet 組態檔更新至叢集中，並建立相對應的 RBAC 權限，以利其他節點使用。
    * 從叢集取得最新版本的 kubelet 內容，並覆寫到`/var/lib/kubelet/config.yaml`中。
    * 更新當前節點的 `/var/lib/kubelet/kubeadm-flags.env` 檔案，以確保使用新版本的參數。
    * 設定主節點的 Annotations(CRISocket)。
    * 更新 CoreDNS 與 kube-proxy 的 DaemonSet 內榮，以透過 Rolling upgrade 方式更新至新版本。

### kubeadm upgrade node
在更新叢集中，kubeadm 會分為`第一主節點`、`其他主節點`與`工作節點`來進行，而想要更新到新版本，必須先在任一台主節點上執行`kubeadm upgrade apply`指令，來確保新版本的組態檔被新增到 Kubernetes 叢集中。一但有了新版本組態資訊後，就能在其他節點執行`kubeadm upgrade node`進行更新，而執行該指令時，又會依據節點的角色分成`主節點`與`工作節點`進行，這兩者流程如下所示:

1. 取得叢集中的 kubeadm ClusterConfiguration 資訊，並識別該節點為什麼角色。若是主節點的話，則會進行控制平面元件的更新(流程同 kubeadm upgrade apply)。
2. 從叢集取得最新版本的 kubelet 內容，並覆寫到`/var/lib/kubelet/config.yaml`中。
3. 更新當前節點的 `/var/lib/kubelet/kubeadm-flags.env` 檔案，以確保使用新版本的參數。

## 結語
今天簡單分析 kubeadm 更新流程，以學習 kubeadm 叢集更新的實踐方式，透過瞭解其流程也能確保發生問題時，能更快知道問題點。且實際了解後，才能知道如何針對調整過參數的叢集進行更新，因為執行預設值時，kubeadm 會將過去的設定覆蓋掉，並且以升級版本的最佳實踐組態來設定，因此可能會造成原本舊版本叢集有開這功能，但新版本則沒有問題。

> 最近時間被一些事情影響到，沒辦法在短時間內，寫出更多詳細與完整內容，這邊深感抱歉... 我這之後會再利用時間繼續將這挑戰文章優化與調整，以讓大家可以看到更多東西。

## Reference
- https://static.sched.com/hosted_files/kccna18/cf/KubeCon_2018_NA-kubeadm-deep-dive.pdf
- https://v1-15.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade-1-15/
- https://github.com/kubernetes/kubernetes/tree/master/cmd/kubeadm