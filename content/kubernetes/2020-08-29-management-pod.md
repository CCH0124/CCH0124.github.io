---
title: kubernetes - day08
date: 2020-08-29
description: "POD 資源管理-part01"
tags: [kubernetes]
draft: false
---


`POD` 和容器一樣，應該只運行一個應用，這樣才有輕量化的感覺。舉例來說，前端和後端應該要在各自的 `POD` 上，這樣的優勢有很多，像是被調度到不同節點上運行，提高資源使用上的效率。然而，`Kubernetes` 的伸縮功能，可針對每個獨立的 `POD` 進行，這樣提高了*靈活性*。但是，實際上有些系統設計需要在一個 `POD` 中運行多個容器，而這些的設計想必又會有一套原則去實踐，如下：

- Sidecar pattern
- Ambassador pattern
- Adapter pattern
- 等等

## 管理 POD 容器

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  namespace: default
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: myapp # 容器名稱
    image: nginx:1.18 # image
  - name: busybox
    image: busybox:latest
    command: ["bin/sh", "-c", "sleep 3600"]
```

以上面的 `POD` 資源清單來看，`containers` 是被用來定義容器清單。在 `POD` 中必定要有一個容器，因此該字段必須是要定義的。然而容器的環境設置還有許多參數可設定，可用以下方式去查看，會列出關於 `containers` 的相關字段。

```shell
$ kubectl explain pods.spec.containers
KIND:     Pod
VERSION:  v1

RESOURCE: containers <[]Object>
...
```
## Image 取得策略

上面 `yaml` 在 `containers` 中有定義 `image` 字段，用來獲取定義的 `image`。我們都知道 `POD` 是用來運行容器，想必節點上必須要有 `CRI`，這邊以 `Docker` 為例，其獲取 `image` 過程如下圖。

![](https://static.coderbridge.com/img/techbridge/images/kdchang/docker101/linux-vs-docker-comparison-architecture-docker-components.png) 來自"TechBridge技術共筆部落格"

運作大致是，運行容器時，容器的引擎會於本地尋找所定義的 `image` 檔案，當不存在時會從 `Registry` 下載到本地端。而在 `Kubernetes` 中可讓用戶自定義關於容器 `image` 的取得策略，其字段為 `imagePullPolicy`，其可有以下值做設定，詳細可參考[官網](https://kubernetes.io/docs/concepts/containers/images/#updating-images)。

- Always
    - 總是從倉庫下載，不論本地有無
    - latest 標籤使用此決策
- Never
    - 本地有就使用，否則需手動下載
- IfNotPresent
    - 本地不存在下載
    - 預設設定

可嘗試使用上面的 `yaml` 建立 `POD`，用 `-o yaml` 觀察 `imagePullPolicy` 是否預設為 `IfNotPresent`，當 `image` 帶有 `latest` 標籤時始以 `Always` 作為預設。而 `imagePullPolicy` 設置如下

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  namespace: default
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: myapp
    image: nginx:1.18
    imagePullPolicy: Always
  - name: busybox
    image: busybox:latest
    command: ["/bin/sh", "-c", "sleep 3600"]
```



## Port 映射

在 `Docker` 網路設計中，每個容器的網路需要透過 `NAT` 機制將其暴露到網路中才能被其他節點上的容器訪問。但在 `Kubernetes` 中，每個 `POD` 的 `IP` 位址都在同一個網路上，而這樣的設計讓其它節點的 `POD` 客戶端可直接訪問，因此 `Port` 的定義可為集群的客戶端提供一個快速連接 `POD` 的可訪問 `Port` 途徑。

其 `yaml` 配置如下

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ports-demo
  namespace: default
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: myapp
    image: nginx:1.18
    imagePullPolicy: Always
    ports: # 從這邊開始
    - name: http # 可被 Service 資源調用
      containerPort: 80 # 在 POD 對象的 IP 地址上映射的容器 Port
      protocol: TCP
  - name: busybox
    image: busybox:latest
    command: ["/bin/sh", "-c", "sleep 3600"]
```

上述的設定無法讓集群外部的使用者進行訪問，解決此方式是將節點的 `IP` 和 `Port` 將其映射至集群外部，可使用 `hostPort` 和 `hostIP` 將其資訊應設至主機上，叫好的方式是使用 `Service` 資源。


- hostPort
    - 將接收的請求藉由 `NAT` 轉發至由 `containerPort` 提供的指定 `Port`
- hostIP
    - `hostPort` 要綁定的 `IP`，默認是 `0.0.0.0` 表所有可用 `IP` 地址


## 環境變數

容器是一種隔離技術，對於應用程式其環境配置相當麻煩，因此環境變數這個概念，使得該概念可在容器啟用時傳遞一個可配置的訊息。在 `POD` 中的容器環境變數傳遞方法有以下

- `env`
- `envFrom`
- `ConfigMap`
- `Secret`

下面為 `env` 使用範例

```yaml
# 官方範例
apiVersion: v1
kind: Pod
metadata:
  name: envar-demo
  labels:
    purpose: demonstrate-envars
spec:
  containers:
  - name: envar-demo-container
    image: gcr.io/google-samples/node-hello:1.0
    env:
    - name: DEMO_GREETING
      value: "Hello from the environment"
    - name: DEMO_FAREWELL
      value: "Such a sweet sorrow"
```

當上述 `yaml` 成功啟動時，透過 `exec` 去驗證，是否以環境變數傳遞。

```shell
$ kubectl exec envar-demo -- printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=envar-demo
DEMO_GREETING=Hello from the environment # this
DEMO_FAREWELL=Such a sweet sorrow # this
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.8.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.8.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.8.0.1
KUBERNETES_SERVICE_HOST=10.8.0.1
NPM_CONFIG_LOGLEVEL=info
NODE_VERSION=4.4.2
HOME=/root
```

## 共享節點網路

簡單來說就是把主機的網路掛到 `POD` 中，使用 `hostNetwork` 字段即可實現。如下圖所示

<!-- ![](../assets/img/k8s/K8s-day08.jpg) -->
{{< figure src="/images/k8s/K8s-day08.jpg" width="auto" height="auto">}}


我們定義以下 `yaml` 驗證。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-host-net-demo
  namespace: default
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: myapp
    image: nginx:1.18
    imagePullPolicy: Always
    ports:
    - name: http
      containerPort: 80
      protocol: TCP
  - name: busybox
    image: busybox:latest
    command: ["/bin/sh", "-c", "sleep 3600"]
  hostNetwork: True
```

從 `IP` 可以看出來它變成節點 `IP`。

```shell
$ kubectl get pods -o wide
NAME                READY   STATUS    RESTARTS   AGE     IP                NODE     NOMINATED NODE   READINESS GATES
pod-host-net-demo   2/2     Running   0          116s    192.168.134.135   node02   <none>           <none>
```


## POD 容器安全

在 `POD` 中可以設置容器的權限和訪問控制。在 `POD` 等級使用 `$ kubectl explain pods.spec.securityContext` 查看，至於容器則是 `$ kubectl explain pods.spec.containers.securityContext` 。詳細可參考[官方資源](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)


## 總結

今天描述了 `POD` 資源清單定義容器的一些應用，上面資源清單可使用 `kubectl apply -f` 方式去部署 `POD`，同時在熟練前面幾個章節介紹的指令去觀察 `POD`，像是 `describe`、`get`、`-o wide` 等。