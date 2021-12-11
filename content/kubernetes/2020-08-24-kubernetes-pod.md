---
title: kubernetes - day03
date: 2020-08-24
description: "kubernetes POD 資源基本介紹"
tags: [kubernetes]
draft: false
---

# Kubernetes 核心資源 POD 介紹

在集群的 `Master` 上提供了 `API Server` 元件，同時有著 `RESTful` 的風格。像是 POD 資源包含了所有 POD 對象的集合，其用於描述在哪個節點進行容器的實例、需要配置什麼樣的環境(硬體資源或儲存)和管理上的策略，其策略可能是*重啟*、*升級*等。另外的 Kubernetes 也提供比 POD 更抽象的 POD 控制器，可用來確保 POD 的存在，這些控制器有 *Deployment*、*Service* 等。

## POD 資源對象

POD 是一到多個容器應用的集合，這也包含著儲存、網路等元件。POD 可以想成是一個應用程式運行的實例，當中共享著前面提到的原件，如下圖所示。

<!-- ![](../assets/img/k8s/K8s-POD.jpg) -->
{{< figure src="/images/k8s/K8s-POD.jpg" width="auto" height="auto">}}

因為共享著網路，所以 POD 中的對象都會是同一網段。因此可以透過 IP 直接進行通訊，不論是在哪一個節點上。所以這樣的結構可以想像成是一個 VM 虛擬機，但不同的是 POD 中的對象容器各個行程都相互彼此隔離。整體來看，該 POD 中的網路與儲存是容器的關鍵資源。但是，與 POD 外部的元件通訊需要透過 `service` 資源所創建的 `ClusterIP` 等類型和對應 `Port` 進行通訊。為了讓 POD 中資源能夠共享數據，因此提供了 `volume` 的資源。在更高階的抽象中該 `volume` 還可以讓容器在生命週期中時確保數據不遺失。

之後的章節會分享對於 POD 的擴充機制、訊息的處裡等資源。

圖中我們還看到一個為 `Pause` 的容器，該容器是 POD 中的基礎設施。它的 `image` 很小，都處於暫停狀態，它是解決 POD 網路而產生的，所以 POD 的 Network Namespace 的對應就是該容器。其實現程式碼從此[鏈接](https://github.com/kubernetes/kubernetes/tree/master/build/pause)可以得知如何實現。


## POD 控制器

不只有容器有生命週期，POD 本身也有生命週期，當透過 `Master` 中控制器建立 POD 資源時會被 `Scheduler` 調度到節點，並由 `kubelet` 建立容器，然而容器的執行任務結束後，會被正常程序給終止並刪除。

當中的 POD 資源假設某種原因故障進而終止程序並刪除，這對於維運者來說會是一個挑戰。因此 Kubernetes 提供了 POD 控制器來進行一些高階的操作，像是實現固定數量、POD 的擴縮容機制、滾動式的更新等，這些操作會自動的做調度處裡，以解決節點故障問題等。
