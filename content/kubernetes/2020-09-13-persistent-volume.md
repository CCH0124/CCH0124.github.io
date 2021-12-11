---
title: kubernetes - day23
date: 2020-09-13
description: "儲存與持久化儲存 part03 - PV 與 PVC"
tags: [kubernetes]
draft: false
---


PersistentVolume(PV)是由管理者提供並配置在某一個儲存方案的一個空間，它將儲存抽象成一個可讓使用者去申請的資源。以前面的 `NFS` 為例，我們都在 `POD` 中直接定義關於儲存的細節像是 `IP` 等，在這的做法缺少靈活性，當儲存要變換 `IP` 或是儲存的路徑，會相對的麻煩。因此可藉由 `PV` 變成是 `POD` 和儲存方案中間的抽象層，這使得 `POD` 不需要知道儲存的細節，這全部由 `PV` 去定義連接與使用，而 `PV` 使用需要透過 PersistentVolumeClaim(PVC) 描述的資源來完成綁定，也就是向 `PV` 申請儲存空間大小或是存取權限。其整體示意圖如下

<!-- ![](../assets/img/k8s/K8s-PV-PVC.jpg) -->
{{< figure src="/images/k8s/K8s-PV-PVC.jpg" width="auto" height="auto">}}

講完了 `PV` 和 `PVC` 概念後這邊來實作，環境是基於上一章。

## 建立 PV 資源

`PersistentVolume` 的定義有以下常見字段
- Capacity
    - `PV` 儲存空間大小定義
- AccessMode
    - 存取模式有以下值，詳細看[官方](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)當中會說明有儲存方案支援哪些模式
        - ReadWriteOnce
        - ReadOnlyMany
        - ReadWriteMany
- persistentVolumeReclaimPolicy
    - `PVC` 移除時對應的 `PV` 動作
    - Retain
        - 保存資料
        - 預設值
    - Delete
        - 對應的 `PV` 資源和資料一同刪除
    - Recycle
        - 保留 `PV`，移除資料
- volumeMode
    - 指定為 `Filesystem` 或是其它，預設為 `Filesystem`
- storageClassName
    - `PV` 所屬的 `StorageClass` 的名稱，默認為空值表示不屬於任何

其字段還有很多可搭配，因為太多東西因此學起來會需要時間...。我們開始實驗吧!!

先在 NFS Server 上創建多個目錄，用於實驗 `PV` 與 `PVC`。

```shell
$ mkdir pv{1,2,3,4,5} 
$ sudo vi /etc/exports
/home/cch/k8sData/pv1 *(rw,sync,no_root_squash)
/home/cch/k8sData/pv2 *(rw,sync,no_root_squash)                                                                          /home/cch/k8sData/pv3 *(rw,sync,no_root_squash)                                                                          /home/cch/k8sData/pv4 *(rw,sync,no_root_squash)                                                                          /home/cch/k8sData/pv5 *(rw,sync,no_root_squash)
$ sudo exportfs -avr
$ showmount -e
Export list for mars:
/home/cch/k8sData/pv5 *
/home/cch/k8sData/pv4 *
/home/cch/k8sData/pv3 *
/home/cch/k8sData/pv2 *
/home/cch/k8sData/pv1 *
```

建立 `PV` 的 `yaml`，`persistentVolumeReclaimPolicy` 則使用預設，下面不設定

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001
  labels:
    name: pv001
spec:
  nfs:
    path: /home/cch/k8sData/pv1
    server: 192.168.134.130
  accessModes: ["ReadWriteMany", "ReadWriteOnce"]
  #persistentVolumeReclaimPolicy: Recycle # persistentVolumeReclaimPolicy 屬性設定方式
  capacity:
    storage: 2Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv002
  labels:
    name: pv002
spec:
  nfs:
    path: /home/cch/k8sData/pv2
    server: 192.168.134.130
  accessModes: ["ReadWriteOnce"]
  capacity:
    storage: 5Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv003
  labels:
    name: pv003
spec:
  nfs:
    path: /home/cch/k8sData/pv3
    server: 192.168.134.130
  accessModes: ["ReadWriteMany", "ReadWriteOnce"]
  capacity:
    storage: 20Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv004
  labels:
    name: pv004
spec:
  nfs:
    path: /home/cch/k8sData/pv4
    server: 192.168.134.130
  accessModes: ["ReadWriteMany", "ReadWriteOnce"]
  capacity:
    storage: 10Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv005
  labels:
    name: pv005
spec:
  nfs:
    path: /home/cch/k8sData/pv5
    server: 192.168.134.130
  accessModes: ["ReadWriteMany", "ReadWriteOnce"]
  capacity:
    storage: 10Gi
```


查看建立的 `PV` 狀態，`RECLAIM POLICY` 就是 `persistentVolumeReclaimPolicy` 的設定。`STATUS` 有以下的可能

- Available
    - 表示 `PV` 為可用狀態，但尚未被 `PVC` 綁定
- Bound
    - 表示已綁定到 `PVC`
- Released
    - `PVC` 已被刪除，但是資源尚未回收
- Failed
    - 回收失敗

```shell
cch@master:~$ kubectl get pv -o wide
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE   VOLUMEMODE
pv001   2Gi        RWO,RWX        Retain           Available                                   63s   Filesystem
pv002   5Gi        RWO            Retain           Available                                   63s   Filesystem
pv003   20Gi       RWO,RWX        Retain           Available                                   63s   Filesystem
pv004   10Gi       RWO,RWX        Retain           Available                                   63s   Filesystem
pv005   10Gi       RWO,RWX        Retain           Available                                   63s   Filesystem
```

`PV` 建立好之後接著要談 `PVC`，上面實驗結果先保留之後 `PVC` 要接續請求 `PV`。

## 建立 PVC 資源

簡單來說就是用來請求一個 `PV` 所建立的，是一個一對一的模式。對 `PV` 做請求時只需要定義要的空間、存取模式、標籤選擇器等訊息即可，系統會自動匹配一個，就像 `POD` 被布署在節點上一樣。下面是一個 `PVC` 請求 `PV` 和 `POD` 請求 `PVC` 的 `yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
  namespace: default
spec:
  accessModes: ["ReadWriteMany"] # 用來匹配 PV
  resources: # 用來匹配 PV
    requests:
      storage: 6Gi # 需要大於等於
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-pvc
  namespace: default
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html/
  volumes: # 定義儲存卷
  - name: html
    persistentVolumeClaim: # 使用 PVC
      claimName: mypvc # 使用的 PVC 名稱
```

布署上面 `yaml`，並使用 `get pv` 查看是否有 `PV` 被綁定。下面結果是 `pv004` 的 `PV`。使用 `get pvc` 也可以查看 `pvc` 被綁定到哪個 `PV`，並有一些相關訊息。但是 `PVC` 請求到的 `PV` 可能會是有浪費空間的情況，因為 `PVC` 是請求一個符合定義條件的 `PV`，但有可能 `PV` 定義的資源並非那麼的剛好，`PVC` 要 10G 的空間，但 `PV` 只有一個 15G 的可用符合條件，選擇了此 `PV` 後將會浪費 5G 的空間。在 `PV` 的資源定義中可在 `spec` 下定義 `claimRef` 指定該 `PV` 要被哪個 `PVC` 使用。

下面 `NAME` 為 pv004 的 `CLAIM` 表示說被 default 的 namespace 中 mypvc 的 `PVC` 綁定。

```shell
$ kubectl apply -f pod-pvc-demo.yaml
persistentvolumeclaim/mypvc unchanged
pod/pod-vol-pvc created
$ kubectl get pv -o wide
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM           STORAGECLASS   REASON   AGE   VOLUMEMODE
pv001   2Gi        RWO,RWX        Retain           Available                                           12m   Filesystem
pv002   5Gi        RWO            Retain           Available                                           12m   Filesystem
pv003   20Gi       RWO,RWX        Retain           Available                                           12m   Filesystem
pv004   10Gi       RWO,RWX        Retain           Bound       default/mypvc                           12m   Filesystem
pv005   10Gi       RWO,RWX        Retain           Available                                           12m   Filesystem
$ kubectl get pvc -o wide
NAME    STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE     VOLUMEMODE
mypvc   Bound    pv004    10Gi       RWO,RWX                       4m27s   Filesystem
```

上面的實驗過程布署從 `PV` 到 `PVC` 到 `POD`，假設是從 `POD` 開始布署則會出現 `Pending` 是因為 `PVC` 沒有找到 `PV`，當然只要 `PV` 沒布署 `PVC` 也會是 `Pending` 狀態，這邊感覺是儲存的需求要先被定義，才能布署 `POD`，這有點矛盾而 `Storage class` 將會改善這些綁定的缺點。

>PVC 具有 namespace，PV 綁定 PVC 可不受限制，但 PVC 和 POD 要在同一個 namespace 下


## 結論

相較於先前用 `volumes` 方式，使用 `PV` 和 `PVC` 讓整個資源變得更複雜一點，多定義了 `PV` 和 `PVC` 資源，這同時也提升除錯難易度。下一個章節會講到 `Storage class`，它為會儲存帶來更好的管理方式。