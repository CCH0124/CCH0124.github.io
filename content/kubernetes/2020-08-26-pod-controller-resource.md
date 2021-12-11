---
title: kubernetes - day05
date: 2020-08-26
description: "POD 相關資源介紹"
tags: [kubernetes]
draft: false
---


上一篇文章，教大家操作 Kubernetes 群集相關的操作，之前也大致說明 POD 是什麼。今天這篇文章會介紹到與 POD 相關的 POD 控制器和一些資源應用，這為了能夠更加豐富的運用 POD 資源，進而將容器的應用變得更加靈活、完善和操作等。下圖顯示了 Kubernetes POD 如何被 Kubernetes 的更抽象資源去應用。


<!-- ![](../assets/img/k8s/K8s-day05.jpg) -->
{{< figure src="/images/k8s/K8s-day05.jpg" width="auto" height="auto">}}

在前面文章有介紹過 `POD`，它負責運行*容器*，同時解決環境依賴的一些問題，可能會是共享的儲存、配置資訊或網頁伺服器要的 SSL 等。然而 `POD` 的運行過程中可能會遇到一些資源配置不足或是一些突發狀況導致 `POD` 非正常的終止。這問題 `Kubernetes` 由*負載類型*的 `POD` 控制器去負責，適當將故障的 `POD` 重建。

雖然 `POD` 控制器能夠重建資源，但有些應用程序是有順序的概念，因此這邊又可將 `POD` 控制器分類為**無狀態**和**有狀態**類型。圖中 `ReplicaSet` 和 `Deployment` 屬於管理無狀態類型服務，`StatefulSet` 則是有狀態。除了這些， Kubernetes 還提供一些特殊應用的服務，像是 `DaemonSet` 它用於為每個節點上運行單一個 `POD` 的資源，可能是 Log 蒐集或是監控服務等應用，還有 `Job` 控制器，它可以提供短暫批次任務，完成後即可終止該資源。

POD 資源一般來說是群集內可相互存取的，再次重建後也應該要被*發現*。如果 POD 要給外部使用者存取則需要透過暴露方式，並且要有負載均衡。在 Kubernetes 中透過 `Service` 和 `Endpoint` 與 `Ingress` 可以解決發現、服務暴露和附載均衡。

在 Kubernetes 中設計了 `Volume`，可用於幫助 `POD` 的資源儲存，它支援了儲存設備和系統，像是 `GlusterFS`、`CEPH RBD` 和 `NFS` 等。不過 Kubernetes 有抽象了 `CSI（Container Storage Interface）` 統一儲存介面以便擴展更多的儲存應用。容器在運行時將不同環境所需的變數用*環境變數*作為解決方案，但容器啟動後不得更改。在 Kubernetes 中 `configMap` 能夠以*環境變數*或*volume*方式傳入至 POD 的容器中，它同時可被不同應用的 POD 共享，使得靈活性提高，然而對於此方式較隱密的資訊較不適合 `configMap` 需使用 `Secret`。

透過上面的描述，我們將與 `POD` 相關的資源進行一系列的詳細說明。但是 Kubernetes 卻沒有那麼簡單，還有許多應用需要去學習，像是 `HorizontalPodAutoscaler` 自動伸縮工作負載；`LimitRange` 可以對硬體資源請求做限制等。預計下一篇會開始操作 `POD` 資源。


## 參考資源

- Kubernetes in action
- [官方 POD](https://kubernetes.io/docs/concepts/workloads/pods/)