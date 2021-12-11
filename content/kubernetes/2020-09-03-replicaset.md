---
title: kubernetes - day13
date: 2020-09-03
description: "ReplicaSet 控制器"
tags: [kubernetes]
draft: false
---

`ReplicaSet` 縮寫為 `RS`，是一種 `POD` 控制器。用來確保所管控的 `POD` 副本數在任一時間都能保證用戶所期望的需求。下圖演示了 `RS` 操作，`RS` 觸發後會去找匹配 `selector` 的 `POD`，當前啟動的數量與期望的數量不符合時，會進行一些操作，像是多的數量則刪除，少的話透過 `POD` 模板建立。當期數量符合用戶定義時則不斷的循環。

![](https://i.imgur.com/NUCNZ4d.png) from Kubernetes in action

`RS` 的建立通常以三個字段定義，`selector`、`replicas` 和 `template`，如下圖所示。當中 `selector`、`replicas` 或 `template` 隨時可按照需求做更改。`replicas` 的定義數量為其直接影響；`selector` 可能讓當前的標籤不再匹配，讓 `RS` 不再對當前的 `POD` 進行控制；`template` 的修改是對後續創建新 `POD` 才產生影響。

![](https://i.imgur.com/oPB7ffr.png) from Kubernetes in action


對於一般手動建立 `POD`，`RS` 帶來的好處有以下

- 確保 `POD` 數量符合定義數量
- 當節點發生故障時，自動請求自其它節點創建缺失的 `POD`
- `POD` 的水平縮放


>必要時可透 HPA(HroizontalPodAutoscaler) 控制器針對資源來自動縮放

## 建立 RS

再前面描述有說明 `RS` 重要的資源創建字段有哪些。這邊再說明一個字段 `minReadySecond` 預設值為 0 秒，在建立新 `POD` 時，啟動後有多長時間無出現異常就可被視為*READY* 狀態。以下為建立一個 `RS` 的範例。

```yaml
# rs-demo.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 2 # 期望數量
  selector: # 標籤選擇器
    matchLabels:
      app: myapp
      release: canary
  template: # POD 模板定義
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        release: canary
        env: qa
    spec:
      containers:
      - name: myapp-container
        image: nginx:1.10
        ports:
        - name: http
          containerPort: 80
```

使用 `apply` 部署 `yaml` 定義的資源
```shell
$ kubectl apply -f rs-demo.yaml
```

查看 `POD` 清單，有趣的是在 `NAME` 欄位，只要 `POD` 被 `myapp` 這個 `RS` 控制，該 `POD` 名稱前綴會有該 `RS` 名稱。
```shell
$ kubectl get pods -l app=myapp,release=canary -o wide
NAME          READY   STATUS    RESTARTS   AGE   IP          NODE                                       NOMINATED NODE   READINESS GATES
myapp-9xkht   1/1     Running   0          98s   10.4.0.13   gke-cluster-1-default-pool-7dc8b11b-z2gx   <none>           <none>
myapp-mvkjk   1/1     Running   0          98s   10.4.2.12   gke-cluster-1-default-pool-7dc8b11b-r7rg   <none>           <none>
```

查看 `RS` 清單，當前按照想要的副本數建立了 2 個 `POD`。
```shell
$ kubectl get rs -o wide
NAME    DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES       SELECTOR
myapp   2         2         2       63s   myapp-container   nginx:1.10   app=myapp,release=canary
```

- DESIRED
  - 期望要有幾個 `Ready` 的 `POD`
- CURRENT
  - 當前運行幾個 `POD`
- READY
  - `READY` 的 `POD` 有幾個

## RS 控管 POD 
這邊會用幾個場景來說明 `RS` 的能力。

### 缺少 POD 
可能某些原因導致 `RS` 控管的 `POD` 少了幾個，然而 `RS` 自動的將其補齊。

這邊實作的話要用 `tmux` 或多開終端機來觀察。

```shell
$ kubectl delete pod myapp-9xkht # 嘗試刪除 RS 中一個 POD
pod "myapp-9xkht" deleted
$ kubectl get rs
NAME    DESIRED   CURRENT   READY   AGE
myapp   2         2         2       12m
$ kubectl get pods -l app=myapp,release=canary
NAME          READY   STATUS    RESTARTS   AGE
myapp-flv45   1/1     Running   0          2m53s # 自動新增了
myapp-mvkjk   1/1     Running   0          14m 
```

利用 `-w` 持續監控，這邊是監控 `RS` 所控制的 `POD` 
```shell
$ kubectl get pods -l app=myapp,release=canary -o wide -w
NAME          READY   STATUS    RESTARTS   AGE   IP          NODE                                       NOMINATED NODE   READINESS GATES
myapp-9xkht   1/1     Running   0          11m   10.4.0.13   gke-cluster-1-default-pool-7dc8b11b-z2gx   <none>           <none>
myapp-mvkjk   1/1     Running   0          11m   10.4.2.12   gke-cluster-1-default-pool-7dc8b11b-r7rg   <none>           <none>
myapp-9xkht   1/1     Terminating   0          12m   10.4.0.13   gke-cluster-1-default-pool-7dc8b11b-z2gx   <none>           <none>
myapp-flv45   0/1     Pending       0          0s    <none>      <none>                                     <none>           <none>
myapp-flv45   0/1     Pending       0          0s    <none>      gke-cluster-1-default-pool-7dc8b11b-z2gx   <none>           <none>
myapp-flv45   0/1     ContainerCreating   0          0s    <none>      gke-cluster-1-default-pool-7dc8b11b-z2gx   <none>           <none>
myapp-9xkht   0/1     Terminating         0          12m   10.4.0.13   gke-cluster-1-default-pool-7dc8b11b-z2gx   <none>           <none>
myapp-9xkht   0/1     Terminating         0          12m   10.4.0.13   gke-cluster-1-default-pool-7dc8b11b-z2gx   <none>           <none>
myapp-flv45   1/1     Running             0          2s    10.4.0.14   gke-cluster-1-default-pool-7dc8b11b-z2gx   <none>           <none>
myapp-9xkht   0/1     Terminating         0          12m   10.4.0.13   gke-cluster-1-default-pool-7dc8b11b-z2gx   <none>           <none>
myapp-9xkht   0/1     Terminating         0          12m   10.4.0.13   gke-cluster-1-default-pool-7dc8b11b-z2gx   <none>           <none>
```

<!-- ![](../assets/img/k8s/K8s-RS-delete-pod.jpg) -->
{{< figure src="/images/k8s/K8s-RS-delete-pod.jpg" width="auto" height="auto">}}

假設我們嘗試將 `RS` 下的 `POD` 強制移除標籤。

```shell
$ kubectl label pods myapp-flv45 app= --overwrite # --overwrite 表示當前要修改的標籤存在，需用此選項選擇覆蓋
pod/myapp-flv45 labeled
$ kubectl get pods -l app=myapp,release=canary
NAME          READY   STATUS    RESTARTS   AGE
myapp-mvkjk   1/1     Running   0          20m
myapp-xt9wr   1/1     Running   0          14s
$ kubectl get pods # myapp-flv45 雖然不被 RS 控管，但它還是回存在
NAME                READY   STATUS    RESTARTS   AGE
myapp-flv45         1/1     Running   0          12m
myapp-mvkjk         1/1     Running   0          24m
myapp-xt9wr         1/1     Running   0          3m45s
pod-nodename-demo   1/1     Running   0          4d4h
```

從結果來看，`myapp-flv45` 這個 `POD` 的 `app` 標籤被移除，這不符合 `RS` 所定義要控管的 `POD` 標籤，因此該 `POD` 不再被 `RS` 控管，同時間少了一個 `POD`，所以 `RS` 又建立了 `myapp-xt9wr`。從這些結果來看，對於刪除或節點故障都會導致永久性消失。

<!-- ![](../assets/img/k8s/K8s-RS-remove-label.jpg) -->
{{< figure src="/images/k8s/K8s-RS-remove-label.jpg" width="auto" height="auto">}}
### 多出 Pod

我們嘗試將 `myapp-flv45` 的標籤打回去，這樣 `RS` 就會控管 3 份 `POD`。

```shell
$ kubectl label pods myapp-flv45 app=myapp --overwrite
$ kubectl get pods -l app=myapp,release=canary
NAME          READY   STATUS    RESTARTS   AGE
myapp-flv45   1/1     Running   0          18m
myapp-mvkjk   1/1     Running   0          30m
```

```shell
$ kubectl get pods -l app=myapp,release=canary -w
NAME          READY   STATUS    RESTARTS   AGE
myapp-mvkjk   1/1     Running   0          28m
myapp-xt9wr   1/1     Running   0          7m40s
myapp-flv45   1/1     Running   0          17m
myapp-flv45   1/1     Running   0          17m
myapp-xt9wr   1/1     Terminating   0          9m22s
myapp-xt9wr   0/1     Terminating   0          9m22s
myapp-xt9wr   0/1     Terminating   0          9m23s
myapp-xt9wr   0/1     Terminating   0          9m23s
```

上面結果來說，3 份 `POD` 對於 `myapp` 來說多了一份，因此這邊看到它終止了 `myapp-xt9wr`。這說明了自主式的 `POD` 或屬於其它控制器的 `POD`，當標籤資源被做了變動一不小心可能就終止了其它 POD 資源。

### 查看 RS 資源事件

這邊使用 `describe`，去查看 `RS`。其內容如下面結果所式，有名稱、所屬 `namespace`、`selector` 定義等。在 `Events` 中可以看出過往新增或刪除了什麼 `POD`。`RS` 之所以能做出這些的反應，是因為它有向 `API Server` 註冊 `watch`，這樣就可以有相關資源變動訊息，`API Server` 在變動觸發時會通知給相關有註冊 `watch` 的客戶端。

```shell
$ kubectl describe rs myapp
Name:         myapp
Namespace:    default
Selector:     app=myapp,release=canary
Labels:       <none>
Annotations:  <none>
Replicas:     2 current / 2 desired
Pods Status:  2 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=myapp
           env=qa
           release=canary
  Containers:
   myapp-container:
    Image:        nginx:1.10
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  37m    replicaset-controller  Created pod: myapp-9xkht
  Normal  SuccessfulCreate  37m    replicaset-controller  Created pod: myapp-mvkjk
  Normal  SuccessfulCreate  25m    replicaset-controller  Created pod: myapp-flv45
  Normal  SuccessfulCreate  16m    replicaset-controller  Created pod: myapp-xt9wr
  Normal  SuccessfulDelete  7m22s  replicaset-controller  Deleted pod: myapp-xt9wr
```

## 更新 RS

一開始 `RS` 控管的 `POD` 使用 `nginx:1.10`
```shell
$ kubectl get rs -o wide
NAME    DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES       SELECTOR
myapp   2         2         2       63s   myapp-container   nginx:1.10   app=myapp,release=canary
```

### 更新 Template


##### edit
使用 `edit` 進行修改，將 `nginx:1.10` 改 `nginx:1.18`

```shell
$ kubectl edit rs myapp
$ kubectl get rs -o wide
NAME    DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES       SELECTOR
myapp   2         2         2       54m   myapp-container   nginx:1.18   app=myapp,release=canary
```
```shell
$ kubectl get pods -l app=myapp,release=canary -o \
> custom-columns=Name:metadata.name,Image:spec.containers[0].image
Name          Image
myapp-flv45   nginx:1.10
myapp-mvkjk   nginx:1.10
```

##### apply
```shell
$ kubectl apply -f rs-demo.yaml # 將鏡像改 1.15 版
replicaset.apps/myapp configured
$ kubectl get pods -l app=myapp,release=canary -o custom-columns=Name:metadata.name,Image:spec.containers[0].image
Name          Image
myapp-flv45   nginx:1.10
myapp-mvkjk   nginx:1.10
$ kubectl get rs -o wide
NAME    DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES       SELECTOR
myapp   2         2         2       59m   myapp-container   nginx:1.15   app=myapp,release=canary
```



最後發現還是 `1.10` 版，但 `RS` 資源卻顯示我們要改的版本。這需要將當前的 `POD` 進行刪除或修改標籤選擇器，才會觸發新模板的建立，過程如下圖



<!-- ![](../assets/img/k8s/K8s-rs-template.jpg) -->
{{< figure src="/images/k8s/K8s-rs-template.jpg" width="auto" height="auto">}}


如果對於新舊的交替也就是更新會有以下操作可能

- 一次刪除所有 `POD` 或是更新標籤選擇器
    - 會有一段時間導致客戶端無法存取
- 分批刪除 `POD` 或是更新標籤選擇器
    - 更新時會有新舊版本供存

雖然這樣可以達到以手動方式更新，但 `K8s` 提供了一個 `Deployment` 控制器(下一篇會說明)，其能夠更完善的實現滾動更新和回滾，同時也可讓客戶端定義更新策略。


### POD 伸縮

只要更改 `replicas` 的數量即可達到 `POD` 的水平擴充。可用 `scale` 做更改，也可使用 `edit` 等方式。

使用 `scale` 擴展成 5 個 `POD`
```shell
$ kubectl scale rs myapp --replicas=5
$ kubectl get rs myapp
NAME    DESIRED   CURRENT   READY   AGE
myapp   5         5         5       7h34m
$ kubectl get pods -l app=myapp,release=canary
NAME          READY   STATUS    RESTARTS   AGE
myapp-fh6cp   1/1     Running   0          31s
myapp-flv45   1/1     Running   0          7h22m
myapp-mvkjk   1/1     Running   0          7h34m
myapp-ts7zd   1/1     Running   0          31s
myapp-vbtnn   1/1     Running   0          31s
```

使用 `edit` 減少至 3 個 `POD`

```shell
$ kubectl edit rs myapp # 找到 rcplicas 字段，進行數量修改
$ kubectl get rs myapp
NAME    DESIRED   CURRENT   READY   AGE
myapp   3         3         3       7h37m
$ kubectl get pods -l app=myapp,release=canary
NAME          READY   STATUS    RESTARTS   AGE
myapp-flv45   1/1     Running   0          7h25m
myapp-mvkjk   1/1     Running   0          7h37m
myapp-ts7zd   1/1     Running   0          3m56s
```


## 刪除 RS 資源

可以使用 `delete` 方式進行 `RS` 資源刪除。一般使用 `delete` 時 `RS` 會將其控管的 `POD` 也一併刪除，當中有 `--cascade=false` 可選，可將 `RS` 控管的 `POD`，變成自主式的 `POD`，同時也少了 `RS` 控管的優點。

```shell
$ kubectl delete -f rs-demo.yaml # 刪除 RS 資源
$ kubectl delete rs myapp # 刪除 RS 資源
```

```shell
$ kubectl delete rs myapp --cascade=false
replicaset.extensions "myapp" deleted
$ kubectl get rs # RS 資源被刪除
No resources found in default namespace.
$ kubectl get pods # 設置了 --cascade=false，讓 POD 變成自主式
NAME                READY   STATUS    RESTARTS   AGE
myapp-flv45         1/1     Running   0          7h32m
myapp-mvkjk         1/1     Running   0          7h44m
myapp-ts7zd         1/1     Running   0          10m
```

## Other
因為實驗環境使用 `GKE` 這邊實驗將一個節點關閉，觀察 `RS` 的處裡，這邊先將 `replicas` 更改為 5 個並重新部署。

```shell
$ kubectl get pods -l app=myapp,release=canary -o wide
NAME          READY   STATUS    RESTARTS   AGE     IP          NODE                                       NOMINATED NODE   READINESS GATES
myapp-5t5qf   1/1     Running   0          3m56s   10.4.0.17   gke-cluster-1-default-pool-7dc8b11b-z2gx   <none>           <none>
myapp-dnbf6   1/1     Running   0          3m56s   10.4.1.11   gke-cluster-1-default-pool-7dc8b11b-cxs1   <none>           <none>
myapp-flv45   1/1     Running   0          7h38m   10.4.0.14   gke-cluster-1-default-pool-7dc8b11b-z2gx   <none>           <none>
myapp-mvkjk   1/1     Running   0          7h50m   10.4.2.12   gke-cluster-1-default-pool-7dc8b11b-r7rg   <none>           <none>
myapp-tmft2   1/1     Running   0          3m56s   10.4.2.14   gke-cluster-1-default-pool-7dc8b11b-r7rg   <none>           <none>
```

這邊將 `gke-cluster-1-default-pool-7dc8b11b-r7rg` 節點進行網卡關閉，最後結果也是一樣，兩個狀態變為 `Unknown` 而 `RS` 又新增兩個，該節點網路如果又恢復，想必又終止 2 個 `POD`，這過程完全不用人為干預。

```shell
$ gcloud compute ssh gke-cluster-1-default-pool-7dc8b11b-r7rg --zone=us-central1-c
$ sudo ifconfig eth0 down
$ kubectl get node
NAME                                       STATUS     ROLES    AGE   VERSION
gke-cluster-1-default-pool-7dc8b11b-cxs1   Ready      <none>   11d   v1.15.12-gke.2
gke-cluster-1-default-pool-7dc8b11b-r7rg   NotReady   <none>   11d   v1.15.12-gke.2
gke-cluster-1-default-pool-7dc8b11b-z2gx   Ready      <none>   11d   v1.15.12-gke.2
$ kubectl get pods -l app=myapp,release=canary
NAME          READY   STATUS    RESTARTS   AGE
myapp-5t5qf   1/1     Running   0          14m
myapp-dnbf6   1/1     Running   0          14m
myapp-flv45   1/1     Running   0          7h49m
myapp-gjpp6   1/1     Running   0          3m8s # 新增
myapp-mvkjk   1/1     Unknown   0          8h # this
myapp-t82p4   1/1     Running   0          3m8s # 新增
myapp-tmft2   1/1     Unknown   0          14m # this
```

將網路恢復
```shell
$ gcloud compute instances reset gke-cluster-1-default-pool-7dc8b11b-r7rg --zone=us-central1-c
Updated [https://www.googleapis.com/compute/v1/projects/sunny-catwalk-286908/zones/us-central1-c/instances/gke-cluster-1-default-pool-7dc8b11b-r7rg].
$ kubectl get node
NAME                                       STATUS   ROLES    AGE   VERSION
gke-cluster-1-default-pool-7dc8b11b-cxs1   Ready    <none>   11d   v1.15.12-gke.2
gke-cluster-1-default-pool-7dc8b11b-r7rg   Ready    <none>   8h    v1.15.12-gke.2
gke-cluster-1-default-pool-7dc8b11b-z2gx   Ready    <none>   11d   v1.15.12-gke.2
```
當節點網路恢復時，其狀態回到 `Ready`，並且狀態為 `Unknown` 的 `POD` 將被刪除。

## 結論

透過本篇文章，詳細的知道 `ReplicaSet` 的原理以及操作。也比較說與自主式 `POD` 的差異。而 `Replica` 能夠實現出對於相同的 `POD` 的負載均衡和保持 `POD` 想要維持的數量。但要注意的是 `ReplicaSet` 只是負責 `POD` 數量有無符合期望值，並不會知道 `POD` 的容器是掛掉的還是怎樣。