---
title: K8s-day01
date: 2020-08-19
description: "Kubernetes 概觀"
tags: [kubernetes]
draft: false
---

## 什麼是 Kubernetes

Kubernetes 是一個集群管理系統，它擁有可移植、擴展和高可用等特性。它用於管理容器(container)，然而容器的出現改變了佈署方式。容器類似於 VM 與主機共享內核(kernel)，容器有自己的檔案系統(filesystem)、CPU、記憶體(memory)、進程空間(process space)等。因為它們與基礎架構分離，因此可以跨雲(公、私有或混合雲中)和作業系統進行移植。然而容器佈署的數量越多管理難度明顯的會提升，使用者必須對容器進行分組，並對其分配網路、監控或安全等服務。藉由 kubernetes 提供的編排和管理功能，以便完成大規模佈署，且 kubernetes 提供了資源調度、容器擴展等管理功能，這些的作用讓 kubernetes 可以管理容器應用程式的生命週期。

## Kubernetes 架構

Kubernetes 匯集多個物理機、虛擬機將其變成一個集群。如下圖

<!-- ![](../assets/img/k8s/K8s-day01.jpg) -->
{{< figure src="/images/k8s/K8s-day01.jpg" width="auto" height="auto">}}

- master
    - 是用戶端與集群之間的核心聯絡重要位置
    - 追蹤其它服務器的健康狀態
    - 以最佳方式調度工作的負載
    - 編排其它組件之間的任務
- node
    - 集群中的工作節點，負責接收來自 master 的工作調度
        - 創建或銷毀 `POD`
        - 調整網路好讓路由與轉發流量
- POD
    - `Kubernetes` 中嘴小運算單元
    - 封裝多個容器
    - 一個 `POD` 中的容器共享 `namespace` 和儲存資源，但彼此又在 `mount`、`user`、`PID` 等名稱空間保持隔離

在 kubernetes 中有很多的組件會來支撐整個集群的運作邏輯，像是編排、暴露、恢復等，之後文章會慢慢的提到。

## master 組件
### API Server

負責輸出 `RESTful` 的 Kubernetes API，負責接收和驗證相關的請求，其狀態資源會被存入 `etcd` 中。

### etcd

該 `etcd` 會確保生產環境的可用性。它會提供監聽的機制，用於推送和監聽變更。只要有發生變化便會與 `API Server` 溝通，並做出適當的動作向客戶端輸出。

### Controller Manager

控制器完成的功能包括*生命週期*和 `API` 一些邏輯。

- 生命週期
    - `namespace` 創建
    - `Event` 垃圾回收
    - `POD` 終止相關垃圾回收
    - 等等
- API
    - `ReplicaSet` 
    - `DaemonSet`
    - 等等

### Scheduler

當 `API Server` 確認 `POD` 對象的建立請求後，需要由 `Scheduler` 根據集群內各節點的可用資源狀態，以及要運行的容器資源需求做出調度決策。總而言之就是幫助 API Server 尋找合適節點部署。

使用以下流程進行容器佈署
- Filtering
    - 淘汰不適合的節點
    - 可能從以下資源進行過濾
        - CPU
        - Memory
        - GPU
- Scoring
    - 進行加權，將過濾的節點進行，分數比較。可能是不要跟某節點一起等
- Deploying
    - 從 scoring 選擇最高分數，並回傳 `API Server` 由它處裡


## Node 組件

### kubelete

運行於工作節點之上的一個進程，從 `API Server` 接收關於 `POD` 對象的*配置訊息*並確保處於期望的狀態。同時也會向 `master` 報告節點資源狀態。`kubelet` 依賴於 `CRI` 來創建容器。

### kube-proxy

工作節點的進程，按需為 `Service` 資源對象生成 `iptables` 或 `ipvs` 規則，進而擷取存取當前 `Service` 的 `ClusterIP` 的流量並將其轉發至正確的 `POD` 上。總而言之網路部分由它負責。
 

### Container-runtime

- CRI
    - 抽象的，因為能運行的容器不只有 `Docker`，而每個容器的設計都不一樣。
- 負責建立 container

## 其他

- CoreDNS
    - 用於服務發現和註冊的動態名稱解析服務
- Prometheus
    - 資源監控
- Ingress Controller
    - 負載均衡器，實現在第七層

## 參考資料

- [kubernetes 官網 what-is-kubernetes](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)
- [redhat what-is-kubernetes](https://www.redhat.com/en/topics/containers/what-is-kubernetes)
- [微軟 what-is-kubernetes](https://azure.microsoft.com/zh-tw/topic/what-is-kubernetes/#overview)
- 書籍：Kubernetes进阶实战