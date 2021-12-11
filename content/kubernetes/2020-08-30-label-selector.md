---
title: kubernetes - day09
date: 2020-08-30
description: "POD 資源管理-part02"
tags: [kubernetes]
draft: false
---

這一章節承接第 8 天的文章，此篇文章要分享的是*標籤*和*標籤選擇器*的應用。

在一個 `Kubernetes` 環境下，當 `POD` 數量越來越多，分類和管理將會變得重要，因為這樣才能提升管理的效率，而在 `K8s` 中有一個 `Label` 的字段，它可用來定義資源而外的訊息像是測試環境、開發環境、前端等，往後在藉由標籤選擇器進行過濾完成想要的目標和任務。

## 標籤
標籤可以依附在每個 K8s 資源對象之上，一個對象可有不止一個標籤，而同一個標籤可被新增到多個資源上。如上述所講，它可以靈活的進行資源對象的分類進行管理，標籤可用版本、環境、分層架構等方式進行貼標籤動作。下圖為 "Kubernetes in action" 書中圖片


![](https://i.imgur.com/9yonwLu.png)


>相較於 `annotations`，`annotations` 無法用來挑選資源對象，僅提供"原數據"，類似註解

## 管理標籤

K8s 中標籤字段定義在 `metadata` 上，以下面為範例，定義兩個標籤分別是 `app` 和 `tier`，值分別是 `myapp` 和 `backend`。

```yaml=
# pod-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-label-demo
  namespace: default
  labels:
    app: myapp
    tier: backend
spec:
  containers:
  - name: myapp
    image: nginx:1.18
```

如果上面的 `yaml` 檔部署完成後，可用 `--show-labels` 的選項，讓列出 `POD` 資源時帶有定義的標籤。

```shell
$ kubectl get pod --show-labels
NAME             READY   STATUS    RESTARTS   AGE     LABELS
pod-label-demo   1/1     Running   0          10s     app=myapp,tier=backend
```

假設標籤給很多時，使用 `-L key1,key2...`，可指定要看的標籤訊息。

```shell
$ kubectl get pod -L tier
NAME             READY   STATUS    RESTARTS   AGE     TIER
envar-demo       1/1     Running   0          3d20h
pod-demo         2/2     Running   93         3d21h   frontend
pod-label-demo   1/1     Running   0          2m48s   backend # this
pod-ports-demo   2/2     Running   92         3d20h   frontend
$ kubectl get pods -L tier --show-labels
NAME             READY   STATUS    RESTARTS   AGE     TIER       LABELS
envar-demo       1/1     Running   0          3d20h              purpose=demonstrate-envars
pod-demo         2/2     Running   93         3d21h   frontend   app=myapp,tier=frontend
pod-label-demo   1/1     Running   0          4m43s   backend    app=myapp,tier=backend
pod-ports-demo   2/2     Running   92         3d20h   frontend   app=myapp,tier=frontend
```

在 K8s 操作中，也提供用 `kubectl label` 方式打標籤

```shell
$ kubectl label pods pod-demo release=canary
pod/pod-demo labeled
$ kubectl get pod -l release --show-labels
NAME       READY   STATUS    RESTARTS   AGE     LABELS
pod-demo   2/2     Running   93         3d21h   app=myapp,release=canary,tier=frontend

# 當要更正已存在標籤則要使用 --overwrite

$ kubectl label pods pod-demo release=stable --overwrite
pod/pod-demo labeled
$ kubectl get pod -l release --show-labels
NAME       READY   STATUS    RESTARTS   AGE     LABELS
pod-demo   2/2     Running   93         3d21h   app=myapp,release=stable,tier=frontend
```

## 標籤選擇器
如果客戶端想針對某個特定標籤下執行特定動作(刪除、查看)，這需要使用標籤選擇器。使用標籤選擇器須注意以下

1. 同時指定的多個選則器之間的邏輯關係為 "AND" 操作
2. 使用空值的標籤選擇器表示每個資源對象都被選擇
3. 空的標籤選擇器無法選擇任何資源

進行標籤選擇時可使用以下方式

- 等值關係
    - `=`
    - `==`
    - `!=`

```shell
# 此範例為選擇 release 標籤且值為 stable
$ kubectl get pods -l release=stable --show-labels
NAME       READY   STATUS    RESTARTS   AGE     LABELS
pod-demo   2/2     Running   93         3d21h   app=myapp,release=stable,tier=frontend
# 此範例為選擇 app 標籤且值為 myapp 和  tier 標籤且值為 backend
$ kubectl get pods -l app=myapp,tier==backend --show-labels
NAME             READY   STATUS    RESTARTS   AGE   LABELS
pod-label-demo   1/1     Running   0          13m   app=myapp,tier=backend
# 此範例為選擇 app 標籤且值為 myapp 和  tier 標籤且值不為 backend
$ kubectl get pods -l app=myapp,tier!=backend -L tier --show-labels
NAME             READY   STATUS    RESTARTS   AGE     TIER       LABELS
pod-demo         2/2     Running   93         3d21h   frontend   app=myapp,release=stable,tier=frontend
pod-ports-demo   2/2     Running   92         3d20h   frontend   app=myapp,tier=frontend
```
- 集合關係
    - key `in`
    - key `notin`
    - key 
    - !key 

```shell
$ kubectl get pods -l "tier in (backend)" -L tier --show-labels
NAME             READY   STATUS    RESTARTS   AGE   TIER      LABELS
pod-label-demo   1/1     Running   0          18m   backend   app=myapp,tier=backend
```

K8s 資源中必須以標籤選擇器方式關連到相關的 `POD` 資源，像是 `Service`、`Deployment` 和 `ReplicaSet` 等資源。這些資源會透過在 `spec` 中定義 `selector` 字段，並使用 `matchLabels` 指定標籤選擇器，或者有些資源可用 `matchExpressions` 帶有表達式方式去做篩選機制。

- matchLabels
    - 給定鍵值來指定標籤選擇器
- matchExpressions 
    - 基於表達式方式指定標籤選擇器
    - [官方介紹](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/)



## POD 選擇節點
指定讓 `POD` 在某個節點上運行，假設只有某節點有 `SSD`、`GPU` 等資源時， 讓`scheduler` 用標籤和標籤選擇器讓 `POD` 選擇匹配的節點。正常的來說 `POD` 部署會藉由 `scheduler` 挑選節點上資源允許的節點部署。

在 `POD` 的 `spec` 中的 `nodeSelector` 可用來選擇想要的節點，前提下節點需設置好標籤。

使用 `kubectl label node` 將節點打標籤。在預設上節點上是有標籤可透過 `kubectl get node --show-labels` 觀察。

```shell
$ kubectl label node gke-cluster-1-default-pool-7dc8b11b-z2gx disk=ssd device=gpu
node/gke-cluster-1-default-pool-7dc8b11b-z2gx labeled
$ kubectl get node -L disk,device 
NAME                                       STATUS   ROLES    AGE    VERSION          DISK   DEVICE
gke-cluster-1-default-pool-7dc8b11b-cxs1   Ready    <none>   7d2h   v1.15.12-gke.2
gke-cluster-1-default-pool-7dc8b11b-r7rg   Ready    <none>   7d2h   v1.15.12-gke.2
gke-cluster-1-default-pool-7dc8b11b-z2gx   Ready    <none>   7d2h   v1.15.12-gke.2   ssd    gpu
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeselector-demo
  namespace: default
  labels:
    app: myapp
    tier: backend
    node: ssd
spec:
  containers:
  - name: myapp
    image: nginx:1.18
  nodeSelector:
    disk: ssd
```
部署完上面 `yaml` 時
```shell
$ kubectl describe pod pod-nodeselector-demo 
....
Events:
  Type    Reason     Age   From                                               Message
  ----    ------     ----  ----                                               -------
  Normal  Scheduled  6s    default-scheduler                                  Successfully assigned default/pod-nodeselector-demo to gke-cluster-1-default-pool-7dc8b11b-z2gx
  Normal  Pulled     5s    kubelet, gke-cluster-1-default-pool-7dc8b11b-z2gx  Container image "nginx:1.18" already present on machine
  Normal  Created    5s    kubelet, gke-cluster-1-default-pool-7dc8b11b-z2gx  Created container myapp
  Normal  Started    5s    kubelet, gke-cluster-1-default-pool-7dc8b11b-z2gx  Started container myapp
```


對於這種綁定還有一種類型叫 `nodeName`，它直接指定目標節點。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodename-demo
  namespace: default
  labels:
    app: myapp
    tier: backend
    node: ssd
spec:
  containers:
  - name: myapp
    image: nginx:1.18
  nodeName: "gke-cluster-1-default-pool-7dc8b11b-r7rg"
```
部署完後用下面進行部署節點位置觀察。

```shell
$ kubectl get pod -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP          NODE                                       NOMINATED NODE   READINESS GATES
...
pod-nodename-demo       1/1     Running   0          43s     10.4.2.9    gke-cluster-1-default-pool-7dc8b11b-r7rg   <none>           <none>
pod-nodeselector-demo   1/1     Running   0          5m8s    10.4.0.11   
...
```

## 資源註釋
與標籤類似，但它的描述內容不受字元大小限制，它可用來添加一些而外訊息。可使用 `kubectl describe pods` 去查看 `POD` 資源的 `Annotations`。


在 `yaml` 中則是在 `metadata` 中使用 `annotations` 去定義，否則也可使用 `kubectl annotate pods` 方式。
