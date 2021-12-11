---
title: kubernetes - day10
date: 2020-08-31
description: "POD 資源管理-part03"
tags: [kubernetes]
draft: false
---

此篇分享是要說關於 POD 對象的生命週期，POD 對象從建立到終止退出為它們的一個生命週期。而在這之間可以透過 POD 的定義執行一些創建容器、初始化容器等操作。在 `openshift` 中提供該生命週期流程圖，如下所示

![](https://i.imgur.com/Y1zhYv5.png)

這邊使用 `openshift` 文章內容進行翻譯解釋其流程

1. 啟動其它容器之前，將啟動 `infra` 容器，以建立其它容器加入的名稱空間，依照此理解應該是指 `pause` 容器
2. 用戶定義的第一個容器啟動是 `init` 容器，可將其用於 POD 範圍內的初始化
3. `main` 容器和 `post-start hook` 同時啟動，在範例中為4秒鐘
4. 在秒數為 7 時，再次按每個容器啟動 `liveness` 和 `readiness probes`
5. 在秒數為 11 時，當 POD 被終止時，執行 `pre-stop hook`，最後在寬限期後終止 `main` 容器

## POD 定相

不管 POD 是被手動建立、或是透過一些 POD 控制器建立，POD 應當處於以下幾個定相

- Pending
    - API Server 創建了 POD 資源並存入 etcd 中，但它尚未被調度完成，或者仍處於從倉庫下載 image 的過程中
- Running
    - POD 已經被調度到某節點，且所有容器已由 `kubelet` 創建
- Failed
    - 所有容器都已經終止，但至少有一個容器終止失敗，即容器返回了非 0 值得退出狀態或已被系統終止
- Succeeded
    - POD 中所有容器都已經成功終止並且不會被重啟
- Unknown
    - API Server 無法取得 POD 狀態，通常是與 `kubelet` 通訊時出錯



- [官方 pod-phase](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase)
- [openshift kubernetes-pods-life](https://www.openshift.com/blog/kubernetes-pods-life)

## POD 建立過程

POD 是 K8s 中的一個核心元件，理解其建立過程對於系統上的運行有一定的幫助。這邊將利用下圖說明建立過程

![](https://i.imgur.com/70aDBqb.png) from `https://blog.heptio.com/`


1. 客戶端透過 `kubectl` 或其它 API，客户端提交 `POD` `Spec` 給 `API Server`
2. `API Server` 嘗試將 POD 對象相關訊息存入 `etcd` 中，待寫入操作執行完成，`API Server` 即回傳確認訊息至客户端
3. `API Server` 開始反映 `etcd` 中的狀態變化
4. 所有 `Kubernetes` 元件都使用 `watch` 機制来追蹤檢查 `API Server` 上相關的變化
5. `scheduler` 透過其 `watcher` 觀察到 `API Server` 創建了新的 POD 對象，但尚未绑定至任何節點
6. `scheduler` 幫 POD 對象選擇節點並將結果訊息更新至 `API Server`
7. 調度結果訊息由 `API Server` 更新至 `etcd`，而 `API Server` 也會反映此 `POD` 對象的調度結果
8. POD 被調度到的節點上的 `kubelet` 嘗試在節點上調用 `Docker`，並把容器結果狀態回傳至 `API Server`
9. `API Server` 將 `POD` 狀態訊息存入 `etcd` 中
10. 在 `etcd` 確認寫入操作成功完成後，`API Server` 將確認訊息發送至相關的`kubelet`，事件將透過它被接受

## POD 生命周期中的行為
### Init Container

`main` 容器啟動之前要運行的容器，常用於為 `main` 容器執行一些預先操作。`init` 容器需要運行完成到結束，若 `init` 容器運行失敗，那 `K8s` 需要重啟它直到成功完成，若 `restartPolicy` 字段是 `Never`，則該 `init` 容器運行失敗時，不會被重啟。這在像是等待其它關聯的元件、從一些倉庫獲取配置等。此字段在 `spec` 中以  `initContainers` 列表方式定義。


### Lifecycle Hook

容器生命週期鉤子(lifecycle hook)使它能夠感知其自身生命周期管理中的事件，並在相應的時刻到来時運行由客戶端指定的處裡程序。`K8s` 提供兩種方式
- postStart
    - 於容器創建完成後立即運行的 `hook` 操作，不過 `K8s` 無法確保它一定會於容器中的`ENTRYPOINT` 之前運行
- preStop
    - 於容器終止操作之前立即運行的 `hook` 操作，它以同步的方式調用，因此在其完成之前會阻塞刪除容器的操作的調用

上面兩種方式定義於 `spec` 中 `lifecycle` 字段。

### Container Probe

容器探測(container probe)是 POD 生命周期中的一項任務，它是 `kubelet` 對容器周期性執行的健康狀態診斷，該操作由容器的處理器(handler)定義。`K8s` 定義了三種方式

- ExecAction
- TCPSocketAction
- HTTPGetAction

這三種方式會回應以下三種狀態

- Success
    - 表示成功
- Failure
- Unknown


`kubelet` 可在容器上執行下面兩種類型的檢測

- livenessProbe
    - 存活性探測
    - failureThreshold
        - 失敗閾值
    - periodSeconds
        - 探測時間週期
        - 規定 `kubelet` 要每隔幾秒執行一次 `liveness probe`
    - initialDelaySeconds
        - 告訴 `kubelet` 在第一次執行 `probe` 之前要的等待幾秒鐘
    - 並且容器將根據其 `restart Policy` 決定未來
- readinessProbe
    - 就緒性探測

##### livenessProbe Example

此範例使用 `exec`

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness-exec
  name: liveness-exec
spec:
  containers:
  - name: liveness-demo
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 60; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - test
        - -e
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

在這資源清單上，`POD` 中只有一個容器。`periodSeconds` 字段指定了 `kubelet` 應該每 5 秒執行一次存活探測(指的是 `livenessProbe` 字段)。 `initialDelaySeconds` 字段向 `kubelet` 在執行第一次探測前應該等待 5 秒。`kubelet` 在容器内執行 `test -e /tmp/healthy` 進行探测。如果執行成功且返回值為 `0`，`kubelet` 會認為此容器是健康存活的。如果返回非 `0` 值，`kubelet` 會終止此容器並重新啟動。

```shell
$ kubectl describe pod/liveness-exec
....
Events:
  Type    Reason     Age   From                                               Message
  ----    ------     ----  ----                                               -------
  Normal  Scheduled  21s   default-scheduler                                  Successfully assigned default/liveness-exec to gke-cluster-1-default-pool-7dc8b11b-cxs1
  Normal  Pulling    19s   kubelet, gke-cluster-1-default-pool-7dc8b11b-cxs1  Pulling image "busybox"
  Normal  Pulled     19s   kubelet, gke-cluster-1-default-pool-7dc8b11b-cxs1  Successfully pulled image "busybox"
  Normal  Created    19s   kubelet, gke-cluster-1-default-pool-7dc8b11b-cxs1  Created container liveness-demo
  Normal  Started    19s   kubelet, gke-cluster-1-default-pool-7dc8b11b-cxs1  Started container liveness-demo
```

30 秒後

```shell
$ kubectl describe pod/liveness-exec
....
Events:
  Type     Reason     Age              From                                               Message
  ----     ------     ----             ----                                               -------
  Normal   Scheduled  73s              default-scheduler                                  Successfully assigned default/liveness-exec to gke-cluster-1-default-pool-7dc8b11b-cxs1
  Normal   Pulling    71s              kubelet, gke-cluster-1-default-pool-7dc8b11b-cxs1  Pulling image "busybox"
  Normal   Pulled     71s              kubelet, gke-cluster-1-default-pool-7dc8b11b-cxs1  Successfully pulled image "busybox"
  Normal   Created    71s              kubelet, gke-cluster-1-default-pool-7dc8b11b-cxs1  Created container liveness-demo
  Normal   Started    71s              kubelet, gke-cluster-1-default-pool-7dc8b11b-cxs1  Started container liveness-demo
  Warning  Unhealthy  3s (x2 over 8s)  kubelet, gke-cluster-1-default-pool-7dc8b11b-cxs1  Liveness probe failed:
```

再 30 秒後

```shell
$ kubectl describe pod/liveness-exec
...
Restart Count:  1 # 被重啟一次
...
```


詳細可參考以下官方文章

- [liveness and readiness](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [configure-liveness-readiness-probes](https://k8smeetup.github.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)


## 容器重啟策略

此字段定義在 `spec` 中 `restartPolicy`，`POD` 可能會發生故障，而對這故障的 `POD` 要做重啟或是不做任何策略都是藉由 `restartPolicy` 定義。

- Always
    - POD 對象終止就將其重啟，默認設定
- OnFailure
    - 僅在 POD 對象出現錯誤時方才將其重啟
- Never
    - 永不重啟


## POD 終止過程

我們以下圖做說明

![](https://i.imgur.com/bgrGO6k.png)


1. 客戶端向 `API Server` 發送刪除 POD 的請求
2. `API Server` 中的 `POD` 會隨著時間的推移而更新，在寬限期 30秒内（預設），POD 被視為 `dead`
3. 將 `POD` 標記為 `Terminating` 狀態
4. 與上一步同時作用，`kubelet` 在監控到 `POD` 轉為 `Terminating` 狀態同時啟動 `POD` 關閉過程
5. 與第三步一同運行，端點控制器(Endpoint controller)監控到 `POD` 的關閉行為時將期從所有匹配到此端點的 `Service` 資源的端點列表中移除
6. 如果當前 `POD` 資源定義了 `preStop`，則在其標記為 `terminating` 後即會以同步的方式啟動運行，如果寬限期結束後，`preStop` 仍未執行完成，則第 2 步將被重新執行並額外取得一個 2 秒的寬限期
7. `POD` 中的容器行程收到 `TERM` 訊號
8. 寬限期結束後，若存在任何一個仍在運行的行程，那麼 `POD` 會收到 `SIGKILL` 訊號
9. `Kubelet` 請求 `API Server` 將此 `POD` 資源的寬限期設置為 0 從而完成刪除作業

>prtStop 只會在容器終止前被調用

## 參考資源
- [pod-lifecycle](https://jimmysong.io/kubernetes-handbook/concepts/pod-lifecycle.html)
- 馬哥 Kubernetes