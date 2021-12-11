---
title: kubernetes - day17
date: 2020-09-07
description: "Service 和 Ingress"
tags: [kubernetes]
draft: false
---

`POD` 並非有持久性，可能出於各種原因重新調度 `POD`，像是失敗的 `liveness` 或 `readiness` 檢查，如果此時與 `POD` 通訊會怎樣？當 `POD` 重啟時，可能具有不同的 IP 地址。這就是為什麼有 `Service` 的資源，`Service` 用於為 `POD` 提供一個固定、統一的存取功能和負載均衡，同時在集群內部使用 `DNS` 實現服務發現，解決客戶端發現容器的問題。`Service` 和 `POD` 的 IP 只在 `Kubernetes` 集群内相互存取，無法直接干預外部流量的請求。而 `Kubernetes` 提供了一些方法像是 `hostPort`、`hostNetwork`、`NodePort` 或 `LoadBalancer` 等，而另一種 `Ingress` 則是資源第七層的均衡負載。`Service` 藉由規則去定義策略，最重要的是還是會藉由*標籤*去實現。


## Service 資源
上述描述的問題，是編排系統可能會遇到的問題。當 `POD` 在做伸縮的應用時或是重新調度，都有可能影響 `IP`，這會導致客戶端存取資源時會錯誤。為了解決此問題 `kubernetes` 提供了 `Service` 資源。前面也提到說 `Service` 還是會藉由*標籤*去實作，如下圖所示，同時 `Service` 也隱藏了真實處裡用戶請求的 `POD`。下圖的標籤選擇器有多個符合的後端，這會讓 `Service` 可以用負載均衡方式進行調度的處裡。

<!-- ![](../assets/img/k8s/K8s-service-label.jpg) -->
{{< figure src="/images/k8s/K8s-service-label.jpg" width="auto" height="auto">}}


`Service` 會提供給 POD 的存取等級，這取決於服務的類型，當前有三種類型：
- ClusterIP
    - 屬於集群內部，只有集群內部的資源可相互存取，默認選項
- NodePort
    - 為節點提供一個可訪問的 IP 與 Port
- LoadBalancer
    - 從雲提供商添加負載均衡器，將流量從服務轉發到服務中的節點

詳細可參考[官網](https://kubernetes.io/zh/docs/concepts/services-networking/service/#publishing-services-service-types)。實際上 `Service` 並非直接與 `POD` 通訊，其之間還有叫 `Endpoints` 的資源，它是由 `IP` 地址和 `Port` 组成的，`Service` 會擁有這個資源是透過標籤選擇器匹配的 `POD` 取得，這部分 `K8s` 幫我們自動實現了。


## 虛擬 IP 和 Service 代理
一個 `Service` 資源其實就是每個節點上的 `Iptables` 或 `ipvs` 規則，將到達 `Service` 的流量轉發至相對應的 `Endpoints`。而這如何將規則創建至 `Iptables` 或 `ipvs` 上呢？這就是 `kube-proxy` 的作用了，它會不斷監聽 `API Server` 中紀錄 `Service` 對應 `POD` 的資訊。

`kube-proxy` 將請求代理到相對應的 `Endpoints` 有三種方式，描述是官方內容
- userspace
    - 多了用戶空間和內核空間之間切換
- iptables
    - 使用 `iptables` 處理流量具有較低的系統開銷，因為流量由 Linux `netfilter` 處理，而無需在用戶空間和內核空間之間切換。這種方法也可能更可靠
- ipvs
    - 與 `iptables` 模式下的 `kube-proxy` 相比，`IPVS` 模式下的 `kube-proxy` 重定向通訊的延遲要短，並且在同步代理規則時具有更好的性能。與其他代理模式相比，`IPVS` 模式還支持更高的網路流量吞吐量。

詳細的內容可參考[官方](https://kubernetes.io/zh/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies)解釋。

