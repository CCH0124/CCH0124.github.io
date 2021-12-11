---
title: kubernetes - day07
date: 2020-08-28
description: "namespace 和 POD 基本資源操作"
tags: [kubernetes]
draft: false
---

## 查看 Namespace 與資源對象

預設的 Kubernetes 提供了幾個 namespace 用於不同的目的，下面的結果在 GKE 或 kubeadm 上目前使用是一樣的，至於 namespace 名稱的用途可參考此[鏈接](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

```shell
$ kubectl get namespace 
NAME              STATUS   AGE
default           Active   13h
kube-node-lease   Active   13h
kube-public       Active   13h
kube-system       Active   13h
```

針對某一個特定 `namespace` 進行詳細訊息查看

```shell
$ kubectl describe namespaces default
```

如果使用了 `namespace` 將一些應用進行*隔離*，我們要查看特定 `namespace` 下資源時須使用 `-n` 參數進行切換，預設是在 `default` 上。下面結果是 `GKE` 的環境。

```shell
$ kubectl get pods -n kube-system
NAME                                                        READY   STATUS    RESTARTS   AGE
event-exporter-v0.3.0-5cd6ccb7f7-mp7p4                      2/2     Running   0          13h
fluentd-gcp-scaler-6855f55bcc-mchvv                         1/1     Running   0          13h
fluentd-gcp-v3.1.1-f8wc8                                    2/2     Running   0          13h
fluentd-gcp-v3.1.1-g6mbn                                    2/2     Running   0          13h
fluentd-gcp-v3.1.1-zq4xm                                    2/2     Running   0          13h
heapster-gke-7c7bdf567c-cmqhm                               3/3     Running   0          13h
kube-dns-5c446b66bd-5ltbw                                   4/4     Running   0          13h
kube-dns-5c446b66bd-fqvwk                                   4/4     Running   0          13h
...
```

## Namespace 資源管理

我們拿上一章節的實驗來做說明。

定義 `namespace` 清單。
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test
```

使用 `create` 建立該清單的資源。
```shell
$ kubectl creat -f namespace-example.yaml
$ kubectl get namespace # 這邊會看到所建立的 test namespace 資源
```

直接建立

```shell
$ kubectl create namespace prod
```

上面說了建立，在刪除方面只要刪除了 `namespace` 會刪除與其相關的資源。下面顯示了刪除的方式，`ns` 為 `namespace` 縮寫。

```shell
$ kubectl delete ns test
namespace "test" deleted
```

## POD 資源管理操作

POD 是 Kubernetes 中的一個核心組件，可以將其用 `json` 或 `yaml` 各式定義乘資源清單，最後由聲明式或宣告式進行資源管理。整個流程如下圖，用戶透過 `create` 或 `apply` 將資源清單請求提交至 `API Server` 並將其保存至集群狀態儲存系統 `etcd` 中，之後由調度元件把用戶請求調度至最佳節點，而被選中節點將借助 `kubectl` 和 `CRI` 進行容器建立。這種由客戶端直接由 `API Server` 建立的 `POD` 稱作為*自主式 POD*。

<!-- ![](../assets/img/k8s/K8s-day07.jpg) -->
{{< figure src="/images/k8s/K8s-day07.jpg" width="auto" height="auto">}}

>我們使用 yaml 定義資源清單，但 Api Server 會將其轉 JSON 再執行

## Imperative Object Configuration 

### 建立 POD 資源

POD 是用於容器相關應用，因此在 `spec` 字段中需要定義 `containers`，它為容器的列表，可以建立多個容器。下面為一個 POD 資源清單範例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo # POD 名稱
  namespace: default # 選擇 namespace
  labels: # 標籤
    app: myapp
    tier: frontend
spec:
  containers: # 定義容器
  - name: myapp
    image: nginx:1.18
  - name: busybox
    image: busybox:latest
    command: ["bin/sh", "-c", "sleep 3600"]
```

使用 `create` 建立 `POD`，`-f` 可以支援以路徑方式或 `URL`。

```shell=
$ kubectl create -f pod-demo.yaml
pod/pod-demo created
```

### 查看 POD 狀態

`get` 通常用來顯示資源對象狀態訊息。使用 `-o wide` 時可以知道更多訊息。
```shell
$ kubectl get pods pod-demo
NAME       READY   STATUS    RESTARTS   AGE
pod-demo   2/2     Running   0          10s
$ kubectl get pods pod-demo -o wide
NAME       READY   STATUS    RESTARTS   AGE     IP         NODE                                       NOMINATED NODE   READINESS GATES
pod-demo   2/2     Running   0          9m11s   10.4.2.5   gke-cluster-1-default-pool-7dc8b11b-r7rg   <none>           <none>
```

`describe` 可以詳細的描述出 `POD` 資源的內容，像是 `Volume`、`QoS`等，當中 `event` 相當重要，其訊息可檢視出為何 `POD` 會故障等訊息以用來除錯。

```shell
$ kubectl describe pods pod-demo
Name:         pod-demo
Namespace:    default
Priority:     0
Node:         gke-cluster-1-default-pool-7dc8b11b-r7rg/10.128.0.22
Start Time:   Sun, 30 Aug 2020 10:56:36 +0000
Labels:       app=myapp
              tier=frontend
Annotations:  kubernetes.io/limit-ranger: LimitRanger plugin set: cpu request for container myapp; cpu request for container busybox
...
```

### 更新 POD 資源

正在運行的 `POD` 上帶有非常多的字段，而並非所有字段都能修改。我們使用 `replace` 方式進行實驗，我們將修改容器 `image`。

先在 `yaml` 檔將 `image` 版本從 `1.18` 變成 `1.12`，在執行以下指令。但發現這樣使用出現了錯誤。
```shell
$ kubectl replace pods -f pod-demo.yaml
The Pod "pod-demo" is invalid: spec: Forbidden: pod updates may not change fields other than `spec.containers[*].image`, `spec.initContainers[*].image`, `spec.activeDeadlineSeconds` or `spec.tolerations` (only additions to existing tolerations)
  core.PodSpec{
-       Volumes: nil,
```

使用以下方式就可以避免上面的失敗。對於 `POD` 來說，能夠修改的欄位有限，自定義的 `yaml` 跟系統上面的 `yaml` 不一樣，因此無法覆蓋，會存在著少資源覆蓋多資源問題。因此在修改時應該是要跟系統 `yaml `一模一樣才對，透過 `-o yaml` 可以取得。在進行 `replace` 時，無定義的 `yaml` 都會被檢視出來，反正必須取得當前運行的 `yaml`，修改後呼叫 `replace` 覆蓋。

```shell
$ kubectl get pod pod-demo -o yaml > pod-replace-demo.yaml
$ vi pod-replace-demo.yaml
$ kubectl replace -f pod-replace-demo.yaml
pod/pod-demo replaced
$ kubectl describe pod pod-demo # 容器 image 已經更改
```

當然除了上面方法，還有加 `--force` 的方式，它會將 `POD` 資源先刪除並重新建立。
```shell
$ kubectl replace -f pod-demo.yaml --force
pod "pod-demo" deleted
pod/pod-demo replaced
$ kubectl describe pod pod-demo # 容器 image 已經更改
```

而 `edit` 是另一種方式，像是使用 `vi`，它直接編輯了該 `POD` 完整資訊，因此可以直接利用此方式進行修改。

```shell
$ kubectl edit pods pod-demo # 此方式也能進行修改
```

### 刪除 POD 資源

```shell
$ kubectl delete pod pod-demo
pod "pod-demo" deleted
```

然而，`replace` 在現實中並非很理想，對於更改的配置難以追蹤，取而代之的是 `Declarative object configuration `，可以嘗試將上面的步驟用 `apply` 取代。

```shell
$ kubectl apply -f pod-demo.yaml
```


## 總結

我們介紹了 `Namespace` 操作同時知道 `POD` 資源配置是由 `kind`、`apiVersion`、`metadata`、`spec` 和 `status` 組成。同時也比較 `Imperative` 和 `Declarative`。這邊也學習到使用 `create`、`delete`、`replace`、`edit`、`apply` 等指令應用。