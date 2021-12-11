---
title: kubernetes - day24
date: 2020-09-14
description: "儲存與持久化儲存 part04 - Storage Class"
tags: [kubernetes]
draft: false
---

在 `K8s` 中 `StorageClass` 對象的目的，就是不去創建 `PV` 而是透過 `PVC` 需求去建立 `PV`，這表示不必要再去管理 `PV` 資源，相較於 `PVC` 請求 `PV` 方式帶來更高的靈活性。

在創建 `StorageClass` 對象時，`name` 的定義一樣是重要的，`PVC` 在調用時會使用 `storageClassName` 去對應該 `StorageClass` 對象的 `name`，創建時還需定義以下通用字段

- provisioner
    - 提供儲存的儲存系統，也就是供應商
- parameter
    - 會依照 `provisioner` 的不同而有不同的參數
- reclaimPolicy
    - `PV` 回收策略，預設是 `Delete` 否則可定義 `Retain`
- volumeBindingMode
    - 定義讓 `PVC` 完成提供和綁定資源

## GKE 上 StorageClass 實作

我們以 Nginx 為例，為它建立一個 `StorageClass`。我們會定義以下的 `yaml`。`volumeBindingMode` 字段可參考[官方](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/#%E5%8D%B7%E7%BB%91%E5%AE%9A%E6%A8%A1%E5%BC%8F)。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass #  定義 StorageClass
metadata:
  name: standard-us-centrall-a-b-c
provisioner: kubernetes.io/gce-pd # 提供儲存的地方
parameters:
  type: pd-standard # 使用一般類型硬碟
reclaimPolicy: Delete # 刪除後自動清除相關資源
volumeBindingMode: WaitForFirstConsumer 
allowedTopologies: # 這邊是設定儲存建立的區域
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values: # 建立區域有以下
    - us-central1-a
    - us-central1-b
    - us-central1-c
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim # 建立 PVC
metadata:
  name: nginx-pvc # 名稱
  namespace: default
  labels:
    app: nginx
    role: backend
spec: # 下面字段都說明過
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 8Gi
  storageClassName: standard-us-centrall-a-b-c # 這邊要對應 StorageClass 對象定義的 name
```


定義 Nginx 的 `POD`，控制器使用 `Deployment`。
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - image: nginx:1.18
          name: nginx
          ports:
            - containerPort: 80
              name: "http-server"
          volumeMounts:
            - name: nginx-persistent-storage
              mountPath: /usr/share/nginx/html
      volumes:
        - name: nginx-persistent-storage
          persistentVolumeClaim: # PVC
            claimName: nginx-pvc # PVC 的 name 字段
```

當上面檔案都建立後，使用 `apply` 方式布署。布署完後先查看 `StorageClass` 是否被建立，結果如下有被建立。

```shell
$ kubectl get storageclass
NAME                         PROVISIONER            AGE
standard (default)           kubernetes.io/gce-pd   20d
standard-us-centrall-a-b-c   kubernetes.io/gce-pd   88s
```

接著查看 `PVC` 是否有綁定到 `StorageClass` 所建立的 `PV`，從下面結果看是有的，`STATUS` 為 `Bound`，用 `describe` 觀察 `Events` 也是可以觀察到成功的綁定。

```shell
$ kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                 AGE
nginx-pvc   Bound    pvc-dfb99470-421f-44ca-aa2c-7f47806f0037   8Gi        ROX            standard-us-centrall-a-b-c   109s

$ kubectl describe pvc nginx-pvc
Name:          nginx-pvc
Namespace:     default
StorageClass:  standard-us-centrall-a-b-c
Status:        Bound
Volume:        pvc-dfb99470-421f-44ca-aa2c-7f47806f0037
Labels:        app=nginx
               role=backend
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/gce-pd
               volume.kubernetes.io/selected-node: gke-cluster-1-default-pool-7dc8b11b-r7rg
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      8Gi
Access Modes:  ROX
VolumeMode:    Filesystem
Mounted By:    nginx-deploy-56fc578995-xhf52 # 由我們建立的 POD 綁定
Events:
  Type     Reason                 Age                  From                         Message
  ----     ------                 ----                 ----                         -------
  Warning  ProvisioningFailed     2m15s                persistentvolume-controller  storageclass.storage.k8s.io "standard-us-centrall-a-b-c" not found
  Normal   WaitForFirstConsumer   73s (x5 over 2m13s)  persistentvolume-controller  waiting for first consumer to be created before binding
  Normal   ProvisioningSucceeded  65s                  persistentvolume-controller  Successfully provisioned volume pvc-dfb99470-421f-44ca-aa2c-7f47806f0037 using kubernetes.io/gce-pd
```

這邊是觀察 `POD` 部分，使用 `describe` 觀察 `Volumes` 部分在 `ClaimName` 下確實有綁定到 `PVC`，在觀察 `Events` 同樣有成功綁定訊息。

```shell
$ kubectl describe pods nginx-deploy-56fc578995-xhf52
Name:           nginx-deploy-56fc578995-xhf52
Namespace:      default
...
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:        100m
    Environment:  <none>
    Mounts:
      /usr/share/nginx/html from nginx-persistent-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-nnghk (ro)
...
Volumes:
  nginx-persistent-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  nginx-pvc
    ReadOnly:   false
  default-token-nnghk:
...
Events:
  Type    Reason                  Age    From                                               Message
  ----    ------                  ----   ----                                               -------
  Normal  Scheduled               2m24s  default-scheduler                                  Successfully assigned default/nginx-deploy-56fc578995-xhf52 to gke-cluster-1-default-pool-7dc8b11b-r7rg
  Normal  SuccessfulAttachVolume  2m19s  attachdetach-controller                            AttachVolume.Attach succeeded for volume "pvc-dfb99470-421f-44ca-aa2c-7f47806f0037"
...
```

查看 `StroageClass` 所建立的 `PV`，建立 `CAPACITY` 為設定的 8Gi，表示說這樣的使用方式可以避免空間浪費的可能。

```shell
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS                 REASON   AGE
pvc-dfb99470-421f-44ca-aa2c-7f47806f0037   8Gi        ROX            Delete           Bound    default/nginx-pvc   standard-us-centrall-a-b-c            4m4s
$ kubectl describe pv pvc-dfb99470-421f-44ca-aa2c-7f47806f0037
Name:              pvc-dfb99470-421f-44ca-aa2c-7f47806f0037
Labels:            failure-domain.beta.kubernetes.io/region=us-central1
                   failure-domain.beta.kubernetes.io/zone=us-central1-c
Annotations:       kubernetes.io/createdby: gce-pd-dynamic-provisioner
                   pv.kubernetes.io/bound-by-controller: yes
                   pv.kubernetes.io/provisioned-by: kubernetes.io/gce-pd
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      standard-us-centrall-a-b-c
Status:            Bound
Claim:             default/nginx-pvc
Reclaim Policy:    Delete
Access Modes:      ROX
VolumeMode:        Filesystem
Capacity:          8Gi
Node Affinity:
  Required Terms:
    Term 0:        failure-domain.beta.kubernetes.io/zone in [us-central1-c]
                   failure-domain.beta.kubernetes.io/region in [us-central1]
Message:
Source:
    Type:       GCEPersistentDisk (a Persistent Disk resource in Google Compute Engine)
    PDName:     gke-cluster-1-1d54e52c-pvc-dfb99470-421f-44ca-aa2c-7f47806f0037
    FSType:     ext4
    Partition:  0
    ReadOnly:   false
Events:         <none>
```

下面兩張圖是在 `GKE` 上的呈現。Google 有一個[qwiklab](https://github.com/CCH0124/qwiklab/blob/master/Kubernetes%20Solutions/Using%20Kubernetes%20Engine%20to%20Deploy%20Apps%20with%20Regional%20Persistent%20Disks.md) 也是針對於 `StorageClass` 進行實驗，有興趣的可以參考。

![](https://i.imgur.com/5oFyuov.png)

![](https://i.imgur.com/4SyPfYG.png)




## Other

```shell
$ kubectl get pods
NAME                                READY   STATUS              RESTARTS   AGE
...
nginx-deploy-56fc578995-bsr8b       0/1     ContainerCreating   0          3m54s
nginx-deploy-56fc578995-svwc5       0/1     ContainerCreating   0          3m54s
nginx-deploy-56fc578995-xhf52       1/1     Running             0          11m
...
$ kubectl describe pod nginx-deploy-56fc578995-bsr8b
...
Events:
  Type     Reason              Age                 From                     Message
  ----     ------              ----                ----                     -------
  Normal   Scheduled           117s                default-scheduler        Successfully assigned default/nginx-deploy-56fc578995-bsr8b to gke-cluster-1-default-pool-7dc8b11b-cxs1
  Warning  FailedAttachVolume  13s (x8 over 111s)  attachdetach-controller  AttachVolume.Attach failed for volume "pvc-dfb99470-421f-44ca-aa2c-7f47806f0037" : googleapi: Error 400: RESOURCE_IN_USE_BY_ANOTHER_RESOURCE - The disk resource 'projects/sunny-catwalk-286908/zones/us-central1-c/disks/gke-cluster-1-1d54e52c-pvc-dfb99470-421f-44ca-aa2c-7f47806f0037' is already being used by 'projects/sunny-catwalk-286908/zones/us-central1-c/instances/gke-cluster-1-default-pool-7dc8b11b-r7rg'
```

以 `ReplicaSet` 來看在定義 `POD` 時需要讓它關聯至一個 `PVC`，而有多個副本時都會共享這個 `PVC`，因此都會綁定到相同的 `PV`。從結果來看不能針對每個 `POD` 去建立獨立的 `PV`，也就是說不能用 `ReplicaSet` 來分配每個 `POD` 一個獨立 `PV`，上面的錯誤就是這種情況。但是，透過手動創建 `POD`、一個 `POD` 對應一個 `ReplicaSet` 然後定義每個 `POD` 的 `PVC` 或是將共享的 `PV` 中利用目錄方式去建立每個 `POD` 的存取目錄，不過這些方式有點繁瑣，加上重新調度問題，這些還會再延伸出問題。在 `K8s` 中有一個 `StatefulSet` 的控制器可以將每個應用程式都分配一個專屬資源，這在後面章節在提。

如果上面的 `ReplicaSet` 資源要共享 `PV` 時，在 `POD` 中 `containers` 的 `VolumeMounts` 字段中將 `readOnly` 設置為 true。


>readOnly 設置結果：Mounted read-only if true, read-write otherwise (false or unspecified). Defaults to false.


```shell
$ kubectl edit deploy nginx-deploy # 修改 replicas 個數
$ kubectl get pods
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deploy-6f4c7bf454-58xtd       0/1     ContainerCreating   0          11m
nginx-deploy-6f4c7bf454-6l7vs       1/1     Running             0          23m
nginx-deploy-6f4c7bf454-8blj5       1/1     Running             0          23m
nginx-deploy-6f4c7bf454-9wdzp       1/1     Running             0          23m
nginx-deploy-6f4c7bf454-c24p9       0/1     ContainerCreating   0          11m
nginx-deploy-6f4c7bf454-j8dcb       0/1     ContainerCreating   0          11m
nginx-deploy-6f4c7bf454-ksdxh       1/1     Running             0          11m # 成功
nginx-deploy-6f4c7bf454-ps7ps       0/1     ContainerCreating   0          11m
nginx-deploy-6f4c7bf454-wzd64       0/1     ContainerCreating   0          11m
nginx-deploy-6f4c7bf454-ztgnr       0/1     ContainerCreating   0          11m
```

以下從檔案更新個數在 `apply` 也是相同錯誤結果。

```shell
Events:
  Type     Reason              Age                   From                                               Message
  ----     ------              ----                  ----                                               -------
  Normal   Scheduled           9m55s                 default-scheduler                                  Successfully assigned default/nginx-deploy-6f4c7bf454-c24p9 to gke-cluster-1-default-pool-7dc8b11b-8kvr
  Warning  FailedMount         66s (x4 over 7m52s)   kubelet, gke-cluster-1-default-pool-7dc8b11b-8kvr  Unable to mount volumes for pod "nginx-deploy-6f4c7bf454-c24p9_default(618dce42-0312-481c-a2ad-17edf2c9a9fd)": timeout expired waiting for volumes to attach or mount for pod "default"/"nginx-deploy-6f4c7bf454-c24p9". list of unmounted volumes=[nginx-persistent-storage]. list of unattached volumes=[nginx-persistent-storage default-token-nnghk]
  Warning  FailedAttachVolume  43s (x12 over 9m50s)  attachdetach-controller                            AttachVolume.Attach failed for volume "pvc-451989b4-ecc8-4571-89ba-2de8c76181c5" : googleapi: Error 400: RESOURCE_IN_USE_BY_ANOTHER_RESOURCE - The disk resource 'projects/sunny-catwalk-286908/zones/us-central1-c/disks/gke-cluster-1-1d54e52c-pvc-451989b4-ecc8-4571-89ba-2de8c76181c5' is already being used by 'projects/sunny-catwalk-286908/zones/us-central1-c/instances/gke-cluster-1-default-pool-7dc8b11b-r7rg'
```

這邊的錯誤結果有待在釐清一下。

## PV 與 PVC 生命週期
1. 提供儲存

- 靜態提供
    - 需要定義 `PV`，並由 `PVC` 匹配
- 動態提供
    - `PVC` 去建立 `PV`

2. 儲存綁定

`PVC` 找到符合的 `PV` 後建立關聯。當 `PVC` 無法有符合 `PV` 時，會等待到有符合的 `PV` 為止，只不過狀態並非是綁定。這邊提到說一個正在使用的 `PVC` 被刪除的話，並非是立即的，會直到無任何資源使用時才會刪除。

3. 儲存回收

就是前面說的 `Retained`、`Recycled` 和 `Deleted`。

4. 擴展 PVC

簡單來說是擴充容量大小，這功能並非所有儲存方案都有，詳細可看[官方](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#lifecycle-of-a-volume-and-claim)


## 參考資源

- [官方 storage-classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [GKE persistent-volumes](https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes?hl=zh-tw)
- [PV 和 PVC 生命週期](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#lifecycle-of-a-volume-and-claim)