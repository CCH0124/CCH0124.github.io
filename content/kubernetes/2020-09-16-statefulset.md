---
title: kubernetes - day26
date: 2020-09-16
description: "StatefulSet 控制器"
tags: [kubernetes]
draft: false
---

`StatefulSet` 簡單來說就是管理有狀態的應用程式，它使得每個應用程式都有不可替換的資源個體，其每個 `POD` 都有固定的 `hostname` 和專屬 `volume`，即時重新調度也不影響。

## Stateful 和 Stateless
一個應用程式是否要紀錄前一次或多次連線中的內容訊息做為下一次連線的資訊，而該連線的對象可能是設備、用戶端或其它應用程式等。透過是否紀錄來分辨有狀態與無狀態，前者為需要記錄動作，後者則不用。

## StatefulSet 與 ReplicaSet
`StatefulSet` 在文章開頭大致上將它的重點說明了。但我們可以比較它和 `ReplicaSet`。`ReplicaSet` 隨著被資源調度就會被新的資源所取代，因為其網路等資源被改變，`StatefulSet` 則是就算被重新調度，其源個體都會有著相同的資源。同樣的 `StatefulSet` 有 `ReplicaSet` 的 `replicas` 功能，但和 `ReplicaSet` 生成的 `POD` 副本卻是不一樣的，`StatefulSet` 中的 `POD` 會有獨立的 `PV` 也就是儲存空間，另一個是 `POD` 名稱，它會使用編號方式去命名，而 `StatefulSet` 也支持滾動更新，基於這些功能 `StatefulSet` 都會講求順序，在後面範例就會明白。

## StatefulSet 特性
### 網路標識
通常一個 `StatefulSet` 會在建立一個 `headless` Service 資源，用來記錄每個 `POD` 的網路標識，就像先前文章講的，每一個 `POD` 都會有一個獨立的 `DNS` 紀錄，使得客戶端能夠透過 `hostname` 去找到服務。

### 資源調度
當 `StatefulSet` 下的 `POD` 發生故障時，會像 `ReplicaSet` 一樣將其重新建立，但是不一樣的是 `StatefulSet` 會讓該新 `POD` 擁有之前 `POD` 的 `hostname` 等，因此透過該 `hostname` 進行訪問時還是會存取到一樣的 `POD` 資源。

### 擴展
前面有提到說 `StatefulSet` 的 `POD` 會有順序編號，該順序編號使得擴展能夠被預知。以 `ResplicaSet` 進行縮減的話則是以隨機方式，無法預知哪個 `POD` 會被終止。

### 個人儲存
竟然資源調度時網路標識能夠相同，那儲存想必也要。在定義 `StatefulSet` 的資源清單時，它可以定義多個 `volumeClaimTemplates`，而建立的儲存會在創建 `POD` 之前被建立，最後在綁定至 `POD`。因為 `StatefulSet` 可以像 `ReplicaSet` 一樣的去擴展，如果以縮減 `POD` 來看，並不會刪除儲存的部分，對於 `StatefulSet` 來說它是一個有狀態的應用，如果將其儲存刪除將會是一個災難，因此手動刪除確認是最好的。也因為不會刪除 `PVC` 的資源，在誤刪時在將縮減後的 `POD` 在擴展時 `PV` 會綁定在相同宣告的儲存上。


## 建立 StatefulSet
這邊使用 `GKE` 進行實驗。

定義 `StorageClass` 資源
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-us-centrall-a-b-c
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - us-central1-a
    - us-central1-b
    - us-central1-c
```
定義 `StatefulSet` 和 `headless` 資源，`StatefulSet` 重要字段大致上是 `serviceName` 和 `volumeClaimTemplates`。而 `volumeClaimTemplates` 就像是定義 `POD` 的模板一樣(template)，因此他才能針對每個 `POD` 去做一個獨立的 `PV`。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sts-svc
  labels:
    app: sts-svc
spec:
  clusterIP: None
  selector:
    app: nginx-pod
  ports:
  - port: 80
    name: web
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-sts
  labels:
    app: nginx
spec:
  serviceName: sts-svc
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
        - image: nginx:1.18
          name: nginx
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: nginx-persistent-storage
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: nginx-persistent-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard-us-centrall-a-b-c # 關聯著我們定義的 StorageClass
      resources:
        requests:
          storage: 10Gi
```

使用 `apply` 布署，並使用 `-w` 觀察 `POD`，如下它們都是按照順序去建立 `POD`，並非像 `ReplicaSet` 可以並行，這種現象並非只有創建 `POD` 在刪除或是滾動更新都是呈現這模式。如果要使用並行需在 `spec` 下定義 `podManagementPolicy` 為 `Parallel`，預設效果就是下面結果。

```shell
$ kubectl get pods -w
NAME                                READY   STATUS    RESTARTS   AGE
...
nginx-sts-0                         0/1     Pending   0          0s
nginx-sts-0                         0/1     Pending   0          3s
nginx-sts-0                         0/1     ContainerCreating   0          3s
nginx-sts-0                         1/1     Running             0          14s
nginx-sts-1                         0/1     Pending             0          0s
nginx-sts-1                         0/1     Pending             0          3s
nginx-sts-1                         0/1     ContainerCreating   0          3s
nginx-sts-1                         1/1     Running             0          14s
nginx-sts-2                         0/1     Pending             0          0s
nginx-sts-2                         0/1     Pending             0          3s
nginx-sts-2                         0/1     ContainerCreating   0          3s
nginx-sts-2                         1/1     Running             0          23s
```

查看 `StatefulSet` 資源
```shell
$ kubectl get sts -o wide
NAME        READY   AGE     CONTAINERS   IMAGES
nginx-sts   3/3     7m57s   nginx        nginx:1.18
```

詳細查看某個 `StatefulSet` 資源訊息
```shell
$ kubectl describe sts nginx-sts
```

查看建立的 `PVC` 與 `PV`，結果都是 `Bound` 狀態。
```shell
$ kubectl get pvc
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                 AGE
nginx-persistent-storage-nginx-sts-0   Bound    pvc-4c81312a-38d3-4e0e-bd6b-9dcc9ff93ccb   10Gi       RWO            standard-us-centrall-a-b-c   12m
nginx-persistent-storage-nginx-sts-1   Bound    pvc-a9d9b751-dc74-4d00-ae87-1a878aec9c09   10Gi       RWO            standard-us-centrall-a-b-c   11m
nginx-persistent-storage-nginx-sts-2   Bound    pvc-fa583a17-5ca5-4634-87c4-52f85d211fb4   10Gi       RWO            standard-us-centrall-a-b-c   11m
```
```shell
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                          STORAGECLASS                 REASON   AGE
pvc-4c81312a-38d3-4e0e-bd6b-9dcc9ff93ccb   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-0   standard-us-centrall-a-b-c            12m
pvc-a9d9b751-dc74-4d00-ae87-1a878aec9c09   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-1   standard-us-centrall-a-b-c            12m
pvc-fa583a17-5ca5-4634-87c4-52f85d211fb4   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-2   standard-us-centrall-a-b-c            11m
```

這邊觀察其 `POD` 中 `hostname` 與外部 `POD` 名稱相同，想像一下在 `ReplicaSet` 情況下它並非是以順序編號，而是隨機亂數。
```shell
$ for i in 0 1 2; do kubectl exec nginx-sts-$i --  sh -c 'hostname'; done
nginx-sts-0
nginx-sts-1
nginx-sts-2
```

### 驗證有狀態的 POD 是否會關聯同一個 PV

從以下結果來看，沒有改變其原先對應的儲存。

```shell
$ kubectl delete pod nginx-sts-1 nginx-sts-0 # 刪除 POD 
$ kubectl describe pod nginx-sts-0 # 觀察 ClaimName，並和 get pv 中的 RECLAIM 進行比對
...
Volumes:
  nginx-persistent-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  nginx-persistent-storage-nginx-sts-0
...
```

更清楚的演示，寫入檔案至掛載目錄下，並使用另一個 `POD` 或任一節點使用 `curl` 驗證。這邊就不再演示了。
```shell
$ for i in 0 1 2; do kubectl exec nginx-sts-$i -- sh -c 'echo $(date), Hostname: $(hostname) > /usr/share/nginx/html/index.html'; done
```

### 擴展與縮減
```shell
$ kubectl edit sts nginx-sts # 修改 replicas 成 5
$ kubectl get pv # 新增兩個
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                          STORAGECLASS                 REASON   AGE
pvc-1d3ecb25-891d-4d6d-9708-2498148bcd92   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-3   standard-us-centrall-a-b-c            96s
pvc-4ae8113a-08f6-4d51-a01e-601a349b87e3   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-4   standard-us-centrall-a-b-c            73s
pvc-4c81312a-38d3-4e0e-bd6b-9dcc9ff93ccb   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-0   standard-us-centrall-a-b-c            43m
pvc-a9d9b751-dc74-4d00-ae87-1a878aec9c09   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-1   standard-us-centrall-a-b-c            43m
pvc-fa583a17-5ca5-4634-87c4-52f85d211fb4   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-2   standard-us-centrall-a-b-c            42m

$ kubectl edit sts nginx-sts # 修改 replicas 成 2
$ kubectl get pv # 依舊保留
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                          STORAGECLASS                 REASON   AGE
pvc-1d3ecb25-891d-4d6d-9708-2498148bcd92   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-3   standard-us-centrall-a-b-c            3m36s
pvc-4ae8113a-08f6-4d51-a01e-601a349b87e3   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-4   standard-us-centrall-a-b-c            3m13s
pvc-4c81312a-38d3-4e0e-bd6b-9dcc9ff93ccb   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-0   standard-us-centrall-a-b-c            45m
pvc-a9d9b751-dc74-4d00-ae87-1a878aec9c09   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-1   standard-us-centrall-a-b-c            45m
pvc-fa583a17-5ca5-4634-87c4-52f85d211fb4   10Gi       RWO            Delete           Bound    default/nginx-persistent-storage-nginx-sts-2   standard-us-centrall-a-b-c            44m
cchong0124@cloudshell:~/sc (sunny-catwalk-286908)$
```

### 滾動更新

編輯 `yaml` 檔並將其 nginx 版本變為 1.15，接著從新 `apply`

```shell
$ kubectl rollout status sts nginx-sts
Waiting for 1 pods to be ready...
partitioned roll out complete: 3 new pods have been updated...
$ kubectl get pods -l app=nginx-pod -o custom-columns=NAME:metadata.name,IMAGE:spec.containers[0].image
NAME          IMAGE
nginx-sts-0   nginx:1.15
nginx-sts-1   nginx:1.15
nginx-sts-2   nginx:1.15
$ kubectl rollout history sts nginx-sts
statefulset.apps/nginx-sts
REVISION
1
2
```

回滾，以下實驗可以透過 `get pod -w` 方式觀察 `POD` 的變動是否是按順序。

```shell
$ kubectl rollout undo  sts nginx-sts
statefulset.apps/nginx-sts rolled back
$ kubectl rollout status sts nginx-sts
Waiting for 1 pods to be ready...
Waiting for partitioned roll out to finish: 1 out of 3 new pods have been updated...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
Waiting for partitioned roll out to finish: 2 out of 3 new pods have been updated...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
partitioned roll out complete: 3 new pods have been updated...
$ kubectl get pods -l app=nginx-pod -o custom-columns=NAME:metadata.name,IMAGE:spec.containers[0].image
NAME          IMAGE
nginx-sts-0   nginx:1.18
nginx-sts-1   nginx:1.18
nginx-sts-2   nginx:1.18
```

### DNS

這邊大致演示 DNS 部可搭配，前面擴縮來驗證是否會綁定。下面 `ANSWER SECTION` 顯示三個指向 headless service SRV 的資源紀錄，`ADDITIONAL SECTION` 部分則顯示每個 `POD` 的資源紀錄，沒有排序是正常的，因為它有著相同的優先權。

```shell
$ kubectl run -it srvlookup --image=tutum/dnsutils --rm --restart=Never -- dig SRV sts-svc.default.svc.cluster.local

; <<>> DiG 9.9.5-3ubuntu0.2-Ubuntu <<>> SRV sts-svc.default.svc.cluster.local
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 10672
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 3

;; QUESTION SECTION:
;sts-svc.default.svc.cluster.local. IN  SRV

;; ANSWER SECTION:
sts-svc.default.svc.cluster.local. 30 IN SRV    10 33 0 nginx-sts-1.sts-svc.default.svc.cluster.local.
sts-svc.default.svc.cluster.local. 30 IN SRV    10 33 0 nginx-sts-0.sts-svc.default.svc.cluster.local.
sts-svc.default.svc.cluster.local. 30 IN SRV    10 33 0 nginx-sts-2.sts-svc.default.svc.cluster.local.

;; ADDITIONAL SECTION:
nginx-sts-1.sts-svc.default.svc.cluster.local. 30 IN A 10.4.1.45 
nginx-sts-0.sts-svc.default.svc.cluster.local. 30 IN A 10.4.2.84
nginx-sts-2.sts-svc.default.svc.cluster.local. 30 IN A 10.4.3.124

;; Query time: 71 msec
;; SERVER: 10.8.0.10#53(10.8.0.10)
;; WHEN: Tue Sep 22 09:46:06 UTC 2020
;; MSG SIZE  rcvd: 294

pod "srvlookup" deleted
```

## 結論

了解了 `statefulSet` 與 `ReplicaSet` 的差異像是回滾、PV 等。也明白 `statefulSet` 透過 `DNS` 方式來保持客戶端可見性。

## 參考資源

- [官方 statefulset](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)