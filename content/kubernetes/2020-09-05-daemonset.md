---
title: kubernetes - day15
date: 2020-09-05
description: "DaemonSet 控制器"
tags: [kubernetes]
draft: false
---

`DaemonSet` 簡稱 `ds` 也是一個 `POD` 控制器，用於實現在集群中每個節點運行一份 `POD`，即使是後續加入的節點，而移除節點則會進行 `POD` 的回收。假設只需針對某一些節點上進行部署，則可以使用節點的選擇器或是以標籤方式做限制。`DaemonSet` 使用場景可能有以下

- Log 搜集器
- 集群類型的儲存
- 資源監控相關

下圖為 `Kubernetes in action` 一書，比較 `ReplicaSet` 和 `DaemonSet` 差異
![](https://i.imgur.com/UTcPpOl.png) 

實驗環境是使用 `GKE` 我們可以查看預設下有哪些原件是使用 `DaemonSet`
```shell
$ kubectl get ds -n kube-system -l name!=filebeat
NAME                       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                                                              AGE
fluentd-gcp-v3.1.1         4         4         4       4            4           beta.kubernetes.io/fluentd-ds-ready=true,beta.kubernetes.io/os=linux       14d
metadata-proxy-v0.1        0         0         0       0            0           beta.kubernetes.io/metadata-proxy-ready=true,beta.kubernetes.io/os=linux   14d
nvidia-gpu-device-plugin   0         0         0       0            0           <none>                                                                     14d
prometheus-to-sd           4         4         4       4            4           beta.kubernetes.io/os=linux                                                14d
```
## 建立 DaemonSet

我們這邊以 `filebeat` 軟體作為本實驗，其作用是蒐集 `Log`，這邊會將其部署在每個節點上蒐集節點上的 `POD` 日誌訊息。

```yaml
# ds-filebeat-demo.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat-ds
  namespace: kube-system
  labels:
    app: filebeat-logging
spec:
  selector:
    matchLabels:
      name: filebeat
  template:
    metadata:
      labels:
        name: filebeat
    spec:
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.9.1
        resources:
          limits:
            memory: 300Mi
          requests:
            cpu: 500m
            memory: 300Mi
```

使用 apply 進行部署

```shell
$ kubectl apply -f ds-filebeat-demo.yaml
```

使用 `describe` 查看詳細資訊，`Node-Selector` 為空沒有指定特別的節點標籤，因此代表需要在每一個節點上運行。我們在 `GKE` 上有三個節點因此 `Desired Number of Nodes Scheduled` 為 3，從 `Number of Nodes Scheduled with Available Pods` 中也已經部署 3 個可用的 `POD`

```shell
$ kubectl describe ds filebeat-ds -n kube-system
Name:           filebeat-ds
Selector:       name=filebeat
Node-Selector:  <none>
Labels:         app=filebeat-logging
Annotations:    deprecated.daemonset.template.generation: 1
Desired Number of Nodes Scheduled: 3
Current Number of Nodes Scheduled: 3
Number of Nodes Scheduled with Up-to-date Pods: 3
Number of Nodes Scheduled with Available Pods: 3
Number of Nodes Misscheduled: 0
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  name=filebeat
  Containers:
   filebeat:
    Image:      docker.elastic.co/beats/filebeat:7.9.1
    Port:       <none>
    Host Port:  <none>
    Limits:
      memory:  300Mi
    Requests:
      cpu:        500m
      memory:     300Mi
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                  Message
  ----    ------            ----  ----                  -------
  Normal  SuccessfulCreate  62s   daemonset-controller  Created pod: filebeat-ds-jcfzv
  Normal  SuccessfulCreate  62s   daemonset-controller  Created pod: filebeat-ds-4lh6c
  Normal  SuccessfulCreate  62s   daemonset-controller  Created pod: filebeat-ds-gmq69
```
驗證其部署在每個節點上

```shell
$ gcloud compute instances list # GKE 上部署的節點
NAME                                      ZONE           MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
gke-cluster-1-default-pool-7dc8b11b-cxs1  us-central1-c  e2-medium                  10.128.0.21  34.67.164.169  RUNNING
gke-cluster-1-default-pool-7dc8b11b-r7rg  us-central1-c  e2-medium                  10.128.0.23  34.72.35.109   RUNNING
gke-cluster-1-default-pool-7dc8b11b-z2gx  us-central1-c  e2-medium                  10.128.0.20  35.202.103.6   RUNNING
$ kubectl get pods -n kube-system -l name=filebeat -o custom-columns=NAME:metadata.name,NODE:spec.nodeName
NAME                NODE
filebeat-ds-4lh6c   gke-cluster-1-default-pool-7dc8b11b-r7rg
filebeat-ds-gmq69   gke-cluster-1-default-pool-7dc8b11b-z2gx
filebeat-ds-jcfzv   gke-cluster-1-default-pool-7dc8b11b-cxs1
```

如果對於 `DaemonSet` 部署在節點上有硬體資源需求的話，可在 `spec` 中定義 `nodeSelector` 讓其部署能夠針對選擇的節點部署。

## 新增節點

`DaemonSet` 在沒有設定 `nodeSelector` 下，每新增一個節點該節點就會部署 `DaemonSet` 的資源。

以下是在 `GCP` 的操作

```shell
$ gcloud container node-pools list --cluster cluster-1 --zone=us-central1-c
NAME          MACHINE_TYPE  DISK_SIZE_GB  NODE_VERSION
default-pool  e2-medium     100           1.15.12-gke.2
$ gcloud container clusters resize cluster-1 --node-pool default-pool --num-nodes 4 --zone=us-central1-c
Pool [default-pool] for [cluster-1] will be resized to 4.
```

驗證

```shell
$ gcloud compute instances list
NAME                                      ZONE           MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
gke-cluster-1-default-pool-7dc8b11b-8kvr  us-central1-c  e2-medium                  10.128.0.24  35.238.113.123  RUNNING
gke-cluster-1-default-pool-7dc8b11b-cxs1  us-central1-c  e2-medium                  10.128.0.21  34.67.164.169   RUNNING
gke-cluster-1-default-pool-7dc8b11b-r7rg  us-central1-c  e2-medium                  10.128.0.23  34.72.35.109    RUNNING
gke-cluster-1-default-pool-7dc8b11b-z2gx  us-central1-c  e2-medium                  10.128.0.20  35.202.103.6    RUNNING
$ kubectl get pods -n kube-system -l name=filebeat -o custom-columns=NAME:metadata.name,NODE:spec.nodeName
NAME                NODE
filebeat-ds-4lh6c   gke-cluster-1-default-pool-7dc8b11b-r7rg
filebeat-ds-gmq69   gke-cluster-1-default-pool-7dc8b11b-z2gx
filebeat-ds-jcfzv   gke-cluster-1-default-pool-7dc8b11b-cxs1
filebeat-ds-v8tl7   gke-cluster-1-default-pool-7dc8b11b-8kvr
```

## 更新 DaemonSet

`DaemonSet` 同樣也有更新的機制，透過 `explain` 查看，不同於 `Deployment` 的是 `OnDelete` 用於刪除更新也就是刪除相應節點上的 `POD` 後以新版本重建該 `POD`，同樣的 `RollingUpdate` 是預設值運作與 `Deployment` 類似。

```shell
$ kubectl explain daemonSet.spec.updateStrategy
KIND:     DaemonSet
VERSION:  extensions/v1beta1

RESOURCE: updateStrategy <Object>

DESCRIPTION:
     An update strategy to replace existing DaemonSet pods with new pods.

FIELDS:
   rollingUpdate        <Object>
     Rolling update config params. Present only if type = "RollingUpdate".

   type <string>
     Type of daemon set update. Can be "RollingUpdate" or "OnDelete". Default is
     OnDelete.
```

這邊同樣的使用 `set image` 方式去更新

```shell
$ kubectl -n kube-system set image ds filebeat-ds filebeat=docker.elastic.co/beats/filebeat:6.8.12
$ kubectl get pods -n kube-system -l name=filebeat -w # 查看更新過程，先刪除一個節點上 POD，再創建新版 POD，不斷重複
NAME                READY   STATUS             RESTARTS   AGE
filebeat-ds-4lh6c   1/1     Running            0          79m
filebeat-ds-gmq69   1/1     Running            0          79m
filebeat-ds-kfw6b   0/1     ImagePullBackOff   0          74s
filebeat-ds-v8tl7   1/1     Running            0          48m
filebeat-ds-kfw6b   0/1     Terminating        0          104s
filebeat-ds-kfw6b   0/1     Terminating        0          104s
filebeat-ds-kfw6b   0/1     Terminating        0          105s
filebeat-ds-kfw6b   0/1     Terminating        0          105s
filebeat-ds-tzmm9   0/1     Pending            0          0s
filebeat-ds-tzmm9   0/1     Pending            0          0s
filebeat-ds-tzmm9   0/1     ContainerCreating   0          0s
...
$ kubectl describe -n kube-system daemonsets filebeat-ds # 透過 describe 觀察
...
Events:
  Type    Reason            Age    From                  Message
  ----    ------            ----   ----                  -------
  Normal  SuccessfulCreate  51m    daemonset-controller  Created pod: filebeat-ds-v8tl7
  Normal  SuccessfulDelete  4m8s   daemonset-controller  Deleted pod: filebeat-ds-jcfzv
  Normal  SuccessfulCreate  4m1s   daemonset-controller  Created pod: filebeat-ds-kfw6b
  Normal  SuccessfulDelete  2m17s  daemonset-controller  Deleted pod: filebeat-ds-kfw6b
  Normal  SuccessfulCreate  2m16s  daemonset-controller  Created pod: filebeat-ds-tzmm9
  Normal  SuccessfulDelete  2m10s  daemonset-controller  Deleted pod: filebeat-ds-gmq69
  Normal  SuccessfulCreate  2m8s   daemonset-controller  Created pod: filebeat-ds-hqwln
  Normal  SuccessfulDelete  2m3s   daemonset-controller  Deleted pod: filebeat-ds-4lh6c
  Normal  SuccessfulCreate  114s   daemonset-controller  Created pod: filebeat-ds-v5l28
  Normal  SuccessfulDelete  108s   daemonset-controller  Deleted pod: filebeat-ds-v8tl7
  Normal  SuccessfulCreate  103s   daemonset-controller  Created pod: filebeat-ds-ddkk9
```

而 `DaemonSet` 也可以設定 `minReadySeconds`、`pause` 和 `resume` 等操作，同樣的也有*回滾機制*。

## 回滾

用法與 `Deployment` 是相同的。

```shell
$ kubectl -n kube-system rollout history ds filebeat-ds
daemonset.extensions/filebeat-ds
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
$ kubectl -n kube-system rollout undo ds filebeat-ds --to-revision=1 # 回滾
daemonset.extensions/filebeat-ds rolled back
```

驗證，從 `6.8.12` 回滾至 `7.9.1`
```shell
$ kubectl get pods -n kube-system -l name=filebeat -o custom-columns=Name:metadata.name,Image:spec.containers[0].image
Name                Image
filebeat-ds-77sfr   docker.elastic.co/beats/filebeat:7.9.1
filebeat-ds-kfc2f   docker.elastic.co/beats/filebeat:7.9.1
filebeat-ds-kj5cx   docker.elastic.co/beats/filebeat:7.9.1 
filebeat-ds-phktj   docker.elastic.co/beats/filebeat:7.9.1
```

> CHANGE-CAUSE 為 <none> 是因為沒設定 --recode 原因

## 節點選擇

我們有四個節點，我們將兩個打上 `ssd` 標籤，假設我們因為需求需要 `ssd` 支援。
```shell
$ kubectl get node
NAME                                       STATUS   ROLES    AGE     VERSION
gke-cluster-1-default-pool-7dc8b11b-8kvr   Ready    <none>   68m     v1.15.12-gke.2
gke-cluster-1-default-pool-7dc8b11b-cxs1   Ready    <none>   14d     v1.15.12-gke.2
gke-cluster-1-default-pool-7dc8b11b-r7rg   Ready    <none>   2d12h   v1.15.12-gke.2
gke-cluster-1-default-pool-7dc8b11b-z2gx   Ready    <none>   14d     v1.15.12-gke.2
$ kubectl label node gke-cluster-1-default-pool-7dc8b11b-cxs1 disk=ssd
node/gke-cluster-1-default-pool-7dc8b11b-cxs1 labeled
$ kubectl label node gke-cluster-1-default-pool-7dc8b11b-z2gx disk=ssd
$ kubectl get ds -n kube-system filebeat-ds -o wide # 獲取 DaemonSet 部署資訊
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE    CONTAINERS   IMAGES                                   SELECTOR
filebeat-ds   4         4         4       4            4           <none>          116m   filebeat     docker.elastic.co/beats/filebeat:7.9.1   name=filebeat
```

使用 `apply` 方式更新 `DaemonSet` 資源，編輯原 `yaml` 檔案並增加 `nodeSelector`，如下
```yaml
...
    spec:
      nodeSelector:
        disk: ssd
      containers:
...
```
```shell
$ kubectl apply -f ds-filebeat-demo.yaml
$ kubectl get pods -n kube-system -l name=filebeat -o custom-columns=NAME:metadata.name,NODE:spec.nodeName -w  # 變成兩個
NAME                NODE
filebeat-ds-5mfn7   gke-cluster-1-default-pool-7dc8b11b-cxs1
filebeat-ds-hlrbr   gke-cluster-1-default-pool-7dc8b11b-z2gx

$ kubectl get node -l disk=ssd -L disk  # 驗證了部署到指定標籤的節點上
NAME                                       STATUS   ROLES    AGE   VERSION          DISK
gke-cluster-1-default-pool-7dc8b11b-cxs1   Ready    <none>   14d   v1.15.12-gke.2   ssd
gke-cluster-1-default-pool-7dc8b11b-z2gx   Ready    <none>   14d   v1.15.12-gke.2   ssd
$ kubectl get ds filebeat-ds -n kube-system # 從 NODE SELECTOR 也明確知道選擇的節點標籤
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
filebeat-ds   2         2         2       2            2           disk=ssd        125m
```

最後用下面的圖表示

<!-- ![](../assets/img/k8s/K8s-ds.jpg) -->
{{< figure src="/images/k8s/K8s-ds.jpg" width="auto" height="auto">}}

## 參考資源

- [guide-to-kubernetes-daemonsets](https://medium.com/kubernetes-tutorials/a-guide-to-kubernetes-daemonsets-6db7920ad140)
- Kubernetes in action
- [官方 DaemonSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/daemonset/)
- [官方 rollback-daemon-set](https://kubernetes.io/docs/tasks/manage-daemon/rollback-daemon-set/)