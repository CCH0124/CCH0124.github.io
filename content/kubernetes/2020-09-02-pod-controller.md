---
title: kubernetes - day12
date: 2020-09-02
description: "POD 控制器"
tags: [kubernetes]
draft: false
---

`POD` 的調度過程，前面文章都有提到過。`kubelete` 對於容器的非主行程錯誤無法去感覺，需要依賴於 `liveness probe` 機制。再想想，如果 `POD` 被惡意刪除或是節點出現問題又該如何解決 ? 我們都知道 `kubelet` 是運行在每個節點上的必要代理器，因此節點故障，對於 `POD` 的各種資源都無法有保證性。對於這些問題我們必須使用節點外的 `POD` 控制器實現該保證性。

`POD` 控制器由主節點上的 `kube-controller-manager` 實現，常見的控制器有 `ReplicaSet`、`Deployment`、`DaemonSet`、`StatefulSet`、`Job` 等，它們實現不同的想法來管理 `POD` 資源。當然 `API Server` 不能少，它將負責儲存使用者的清單資源，再由控制器去實現使用者想要的狀態，在這之間控制器會透過 `API Server` 提供的接口進行不斷的監聽資源狀態，因此發生故障、更新等變動系統狀態的原因會不斷的向用戶想要的狀態不斷接近，而 `status` 的狀態就是紀錄當前狀態。




## 控制器與 POD

正常來說，一個 `POD` 控制器資源應該至少有三個字段
- selector
    - 關聯匹配的 `POD`，並管控其 `POD`
- replica
    - 期望的 `POD` 數量
- POD Template
    - 指定控制器創建 `POD` 資源的配置訊息
    - 相似於定義 `POD` 資源


## 結論

此篇文章用於了解，`POD` 資源被控制器控管的好處，以及如何定義控制器資源。再下面的章節將會介紹一些控制器。


## 參考資源

- [feisky - controller-manager](https://feisky.gitbooks.io/kubernetes/content/components/controller-manager.html)