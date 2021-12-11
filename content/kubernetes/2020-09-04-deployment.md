---
title: kubernetes - day14
date: 2020-09-04
description: "Deployment 控制器"
tags: [kubernetes]
draft: false
---

`Deployment` 縮寫為 `deploy`，它被建構在 `ReplicaSet` 控制器之上的控制器。為 `POD` 和 `ReplicaSet` 提供聲明式更新，而 `Deployment` 基本上和 `ReplicaSet` 很相似，只不過多了這些特性

- 可以觀察 `Deployment` 升級的過程
- 可以用回滾方式回到歷史版本中的某一個版本
- 對 `Deployment` 操作紀錄作保存，以便可回滾
- 在每一次升級過程中都可暫停或啟動
- 自動更新機制有兩種
    - Recreate
        - 一次性刪除所有 `POD`，之後再用新版本佈署
    - RollingUpdate
        - 滾動式更新，小階段的更新

## Deployment 的建立

其定義的字段與 `Replica` 很像，如 `replicas`、`selector`、`template` 等。下面是本章節的實驗範例。

```yaml
# deploy-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy
  namespace: default
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp
      release: canary
  template:
    metadata:
     labels:
       app: myapp
       release: canary
    spec:
      containers:
      - name: myapp
        image: nginx:1.10
        ports:
        - name: http
          containerPort: 80
```


佈署該 `yaml`

```shell
$ kubectl apply -f deploy-demo.yaml
```

列出 Deployment 的相關訊息

```shell
$ kubectl get deploy myapp-deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deploy   5/5     5            5           14s
```

- UP-TO-DATE
    - 以達到想要的 `POD` 個數
- AVAILABLE
    - 當前處於可用狀態的數量

前面有提到說 `Deployment` 和 `ReplicaSet` 之間關係，從下面的查看可以驗證，`Deployment` 在 `ReplicaSet` 之上，在 `NAME` 方面 `myapp-deploy` 控制了 `ReplicaSet` 資源，因此會有此前綴。而在標籤選擇器上，`ReplicaSet` 使用的是 `Deployment` 資源清單所設定的標籤選擇器。

```shell
$ kubectl get rs -l app=myapp -o wide
NAME                      DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES       SELECTOR
myapp-deploy-5bc49f9bb9   5         5         5       4m15s   myapp        nginx:1.10   app=myapp,pod-template-hash=5bc49f9bb9,release=canary
```

```shell
$ kubectl describe rs myapp-deploy-5bc49f9bb9
Name:           myapp-deploy-5bc49f9bb9
Namespace:      default
Selector:       app=myapp,pod-template-hash=5bc49f9bb9,release=canary
Labels:         app=myapp
                pod-template-hash=5bc49f9bb9
                release=canary
Annotations:    deployment.kubernetes.io/desired-replicas: 5
                deployment.kubernetes.io/max-replicas: 7
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/myapp-deploy # 這邊清楚的知道 Deployment 控制 ReplicaSet
Replicas:       5 current / 5 desired
Pods Status:    5 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=myapp
           pod-template-hash=5bc49f9bb9
           release=canary
  Containers:
   myapp:
    Image:        nginx:1.10
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  28m   replicaset-controller  Created pod: myapp-deploy-5bc49f9bb9-hcfbq
  Normal  SuccessfulCreate  28m   replicaset-controller  Created pod: myapp-deploy-5bc49f9bb9-bpcjp
  Normal  SuccessfulCreate  28m   replicaset-controller  Created pod: myapp-deploy-5bc49f9bb9-5hqgb
  Normal  SuccessfulCreate  28m   replicaset-controller  Created pod: myapp-deploy-5bc49f9bb9-75hxv
  Normal  SuccessfulCreate  28m   replicaset-controller  Created pod: myapp-deploy-5bc49f9bb9-lhnsz
```


接下來我們查看 `POD` 資源，我們觀察 `NAME` 的命名格式，它以上面的 `ReplicaSet` 的名稱為前綴，表示由 `ReplicaSet` 控制。這樣的標記方式讓管理者更直覺的知道這是什麼。

```shell
$ kubectl get pods -l app=myapp,release=canary
NAME                            READY   STATUS    RESTARTS   AGE
myapp-5t5qf                     1/1     Running   0          14h
myapp-deploy-5bc49f9bb9-5hqgb   1/1     Running   0          8m30s
myapp-deploy-5bc49f9bb9-75hxv   1/1     Running   0          8m30s
myapp-deploy-5bc49f9bb9-bpcjp   1/1     Running   0          8m30s
myapp-deploy-5bc49f9bb9-hcfbq   1/1     Running   0          8m30s
myapp-deploy-5bc49f9bb9-lhnsz   1/1     Running   0          8m30s
myapp-dnbf6                     1/1     Running   0          14h
myapp-flv45                     1/1     Running   0          22h
myapp-gjpp6                     1/1     Running   0          14h
myapp-t82p4                     1/1     Running   0          14h
```

```shell
$ kubectl describe pods myapp-deploy-5bc49f9bb9-75hxv
Name:           myapp-deploy-5bc49f9bb9-75hxv
Namespace:      default
Priority:       0
Node:           gke-cluster-1-default-pool-7dc8b11b-z2gx/10.128.0.20
Start Time:     Thu, 10 Sep 2020 06:53:22 +0000
Labels:         app=myapp
                pod-template-hash=5bc49f9bb9
                release=canary
Annotations:    kubernetes.io/limit-ranger: LimitRanger plugin set: cpu request for container myapp
Status:         Running
IP:             10.4.0.20
IPs:            <none>
Controlled By:  ReplicaSet/myapp-deploy-5bc49f9bb9 # 驗證說明此 POD 由 ReplicaSet 控制
```
藉由上面的範例，可以更清楚知道 `Deployment` 和 `ReplicaSet` 之間關係，其操作的技巧都非常相似，但 `Deployment` 卻有著更高階的操作技巧。總體來說它門之間關係如下圖所示

<!-- ![](../assets/img/k8s/K8s-deployment.jpg) -->
{{< figure src="/images/k8s/K8s-deployment.jpg" width="auto" height="auto">}}

## 更新策略
在 `ReplicaSet` 文章中，我們發現它對於更新操作需要人為介入，有人為操作就有操作錯誤的風險。而 `Deployment` 就移除了人為操作更新的介入，不論修改容器中鏡像版本或是更改 `Template`，`Deployment` 會幫助我們完成。這邊使用 `describe` 觀察建立的 `Deployment` 預設更新策略，發現是 `RollingUpdate`。

```shell
$ kubectl describe deploy myapp-deploy
Name:                   myapp-deploy
Namespace:              default
CreationTimestamp:      Thu, 10 Sep 2020 06:53:22 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=myapp,release=canary
Replicas:               5 desired | 5 updated | 5 total | 5 available | 0 unavailable
StrategyType:           RollingUpdate # 更新策略
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=myapp
           release=canary
  Containers:
   myapp:
    Image:        nginx:1.10
    Port:         80/TCP
...
```

這邊在藉由 `explain` 來看更新策略是否是文章開頭的兩種，從下面來看確實是 `RollingUpdate` 和 `Recreate` 兩種，而 `RollingUpdate` 是預設。前面有稍微提到 `Recreate` 的用法，一次終止所有 `POD` 在進行佈署，這依然存在有短暫時間無法存取的問題，感覺上適用於舊版與新版的兼容性很差時。`RollingUpdate` 它以小批次方式進行更新，刪除一些 `POD` 在創建新版的 `POD`。相較於 `Recreate`，會有要存取新版還是舊版 `POD` 的問題，但是不會有無法存取的問題。

```shell
$ kubectl explain deployment.spec.strategy
KIND:     Deployment
VERSION:  extensions/v1beta1

RESOURCE: strategy <Object>

DESCRIPTION:
     The deployment strategy to use to replace existing pods with new ones.

     DeploymentStrategy describes how to replace existing pods with new ones.

FIELDS:
   rollingUpdate        <Object>
     Rolling update config params. Present only if DeploymentStrategyType =
     RollingUpdate.

   type <string>
     Type of deployment. Can be "Recreate" or "RollingUpdate". Default is
     RollingUpdate.
```

在 `Deployment` 中 `RollingUpdate` 的操作非是同一個 `ReplicaSet` 來執行，而是會新建一個 `ReplicaSet`，這之間會將先前 `ReplicaSet` 控管的 `POD` 做刪除，同時間新的 `ReplicaSet` 會增加新版的 `POD`，停止條件簡單來說就是舊 `ReplicaSet` 要沒有 `POD`，新 `ReplicaSet` 要符合正確的 `POD` 數量，這過程用下圖表示，圖中可以看到舊版的 `ReplicaSet` 未被刪除，是因為它可用作*回滾*這個動作，`revisionHistoryLimit` 這字段用於控制可保存 `ReplicaSet` 的數量，預設是 10。

<!-- ![](../assets/img/k8s/K8s-deploy-rollingupdate.jpg) -->
{{< figure src="/images/k8s/K8s-deploy-rollingupdate.jpg" width="auto" height="auto">}}


然而 `Deployment` 中有調整滾動更新細粒度的設置選項，分別是 `maxSurge` 和 `maxUnavailable`。透過 `kubectl explain deployment.spec.strategy.rollingUpdate` 可以查看其作用。這邊可以使用 `describe` 觀察 `deployment` 的設定其預設是 `25% max unavailable, 25% max surge`。這邊簡單的說明作用

- maxSurge
    - 更新期間存在的 `POD` 總數量最多可超出的數量
- maxUnavailable
    - 更新期間能不可存取 `POD` 的數量
    - 不論是舊版還是新版


下兩張圖為 *Kubernetes in action* 的圖，可以嘗試理解其作用。

![](https://i.imgur.com/v7X7Iet.png)

![](https://i.imgur.com/rrFZ2iR.png)


## Deployment 更新

這邊將要實驗 `Deployment` 的滾動更新，而我們要修改的是 `image`，這部分可以透過 `apply`、`edit`、`patch` 或 `set image` 等來實現。下面使用 `set image` 更換鏡像，再使用 `rollout status` 查看其更新的狀態，接著觀察 `replicaSet`，其保留舊版，而換新版的 `replicaSet` 控制 `POD`，而 `POD` 的 `NAME`，將由新版 `replicaSet` 作為前綴。

```shell
$ kubectl set image deploy myapp-deploy myapp=nginx:1.15 
deployment.extensions/myapp-deploy image updated
$ kubectl rollout status deploy myapp-deploy # 滾動更新的狀態
deployment "myapp-deploy" successfully rolled out
$ kubectl get replicasets -l app=myapp
NAME                      DESIRED   CURRENT   READY   AGE
myapp-deploy-5bc49f9bb9   0         0         0       110m # 舊版的保留
myapp-deploy-8664dd475d   5         5         5       2m15s
```

## 回滾操作

可能因為佈署新版本時遇到鏡像無法拉取或是應用程式的 `Bug` 因此要將此佈署往之前的一個版本做佈署。我們可以使用 `undo` 關鍵字進行回滾

```shell
$ kubectl get pods -l app=myapp,release=canary -o custom-columns=Name:metadata.name,Image:spec.containers[0].image # 先確認鏡像是否更新至 1.15 
Name                            Image
...
myapp-deploy-8664dd475d-gn9lb   nginx:1.15
myapp-deploy-8664dd475d-hqwfq   nginx:1.15
myapp-deploy-8664dd475d-mvpvr   nginx:1.15
myapp-deploy-8664dd475d-plq8h   nginx:1.15
myapp-deploy-8664dd475d-tbtpq   nginx:1.15
...
```
這邊要將 `nginx:1.15` 回滾為 `nginx:1.10`

```shell
$ kubectl rollout undo deploy myapp-deploy
deployment.extensions/myapp-deploy rolled back
$ kubectl get pods -l app=myapp,release=canary -o custom-columns=Name:metadata.name,Image:spec.containers[0].image
Name                            Image
...
myapp-deploy-5bc49f9bb9-2csm6   nginx:1.10 # 版本回滾，RS 資源回到上一個
myapp-deploy-5bc49f9bb9-9lvlp   nginx:1.10
myapp-deploy-5bc49f9bb9-bqxv8   nginx:1.10
myapp-deploy-5bc49f9bb9-ghp6h   nginx:1.10
myapp-deploy-5bc49f9bb9-h45cm   nginx:1.10
...
```

這邊我們藉由，`rollout history` 查看回滾紀錄。觸發回滾時其 `REVISION` 的紀錄訊息會變更，回滾操作會被視為一次滾動更新因此會被新增自歷史紀錄，我們上前面的回滾紀錄被記做是 3，被回滾的紀錄會被刪除。`--to-revision` 這關鍵字可用來要回滾的版本，這邊再新增一個 `nginx:1.18` 的更新紀錄，這樣就有 2、3 和 4 可選，預設會是以第一列回滾。這邊我們嘗試指定 3 來回滾(版本會回到 1.10)。

```shell
$ kubectl rollout history deploy myapp-deploy # 查看 REVISION 紀錄
deployment.extensions/myapp-deploy
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```


```shell
$ kubectl set image deploy myapp-deploy myapp=nginx:1.18
$ kubectl rollout history deploy myapp-deploy
deployment.extensions/myapp-deploy
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
4         <none> # 剛才的更新配紀錄在這邊
$ kubectl get rs -l app=myapp
NAME                      DESIRED   CURRENT   READY   AGE
myapp-deploy-5bc49f9bb9   5         5         5       139m
myapp-deploy-8664dd475d   0         0         0       32m
myapp-deploy-cd5f457bf    0         0         0       3m22s
$ kubectl rollout undo deploy myapp-deploy --to-revision=3
$ kubectl get pods -l app=myapp,release=canary -o custom-columns=Name:metadata.name,Image:spec.containers[0].image
Name                            Image
...
myapp-deploy-5bc49f9bb9-42w6g   nginx:1.10
myapp-deploy-5bc49f9bb9-dcnpp   nginx:1.10
myapp-deploy-5bc49f9bb9-hlk8r   nginx:1.10
myapp-deploy-5bc49f9bb9-k42bd   nginx:1.10
myapp-deploy-5bc49f9bb9-tnt9f   nginx:1.10
...
$ kubectl rollout history deployments myapp-deploy
deployment.extensions/myapp-deploy
REVISION  CHANGE-CAUSE
2         <none>
4         <none>
5         <none> # 剛回滾被記錄在這
```
### pause 與 resume

這兩個元素，用於佈署時可以針對一小部分進行更新，並觀察期更新的 `POD` 運行結果，此時舊版 `POD` 依舊存在，當新的 `POD` 確認穩定後，再將剩下舊版的 `POD` 資源進行更新。

`pause`，只更新一個並讓資源進入暫停狀態。

```shell
$ kubectl set image deploy myapp-deploy myapp=nginx:1.14 && kubectl rollout pause deploy myapp-deploy
deployment.extensions/myapp-deploy image updated
deployment.extensions/myapp-deploy paused
$ kubectl rollout status deploy myapp-deploy # 監聽 rollout 狀態
Waiting for deployment "myapp-deploy" rollout to finish: 5 out of 10 new replicas have been updated...
```

`resume`，讓暫停的資源重新啟用並執行相對應動作

```shell
$ kubectl rollout resume deploy myapp-deploy
deployment.extensions/myapp-deploy resumed
$ kubectl rollout status deploy myapp-deploy
Waiting for deployment "myapp-deploy" rollout to finish: 5 out of 10 new replicas have been updated...
Waiting for deployment spec update to be observed...
Waiting for deployment spec update to be observed...
Waiting for deployment "myapp-deploy" rollout to finish: 5 out of 10 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 5 out of 10 new replicas have been updated...
Waiting for deployment "myapp-deploy" rollout to finish: 3 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 3 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 3 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "myapp-deploy" rollout to finish: 8 of 10 updated replicas are available...
Waiting for deployment "myapp-deploy" rollout to finish: 9 of 10 updated replicas are available...
deployment "myapp-deploy" successfully rolled out```
```

## 伸縮 POD 數量

使用 `patch` 方式跟新 `replicas` 的副本數量，這邊將其擴充到 10 個，這邊不限制是 `patch` 可以是 `edit` 或 `apply` 等。然而因為 `replicas` 非 `Template` 級別因此不會記錄在 `rollout` 中。

```shell
$ kubectl patch deploy myapp-deploy -p '{"spec":{"replicas": 10}}'
deployment.extensions/myapp-deploy patched
$ kubectl get deploy myapp-deploy # 驗證
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deploy   10/10   10           10          146m
```


## 結論

`Deployment` 靈活性高於 `ReplicaSet`，尤其是在滾動更新方面，也介紹像是 `rollingUpdate` 和 `minReadySeconds` 等字段應用。同時也嘗試使用不同更新佈署資源的方式像是 `edit`、`patch` 等。之後會有一篇更好的 `Deployment` 應用，來說明一些模式的操作。上面提到的一些字段使用以下 `yaml` 作為範例。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy
  namespace: default
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  minReadySeconds: 3
  selector:
    matchLabels:
      app: myapp
      release: canary
  template:
    metadata:
     labels:
       app: myapp
       release: canary
    spec:
      containers:
      - name: myapp
        image: nginx:1.10
        ports:
        - name: http
          containerPort: 80
```


## 參考資料

- [官方 deployment](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/)