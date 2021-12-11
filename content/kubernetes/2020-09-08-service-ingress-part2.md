---
title: kubernetes - day18
date: 2020-09-08
description: "Service 和 Ingress part02"
tags: [kubernetes]
draft: false
---

## 預設的 Service

這預設是讓節點能夠跟 `API Server` 等資源做通訊。

```shell
$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.8.0.1     <none>        443/TCP   15d
$ kubectl describe svc kubernetes
Name:              kubernetes
Namespace:         default
Labels:            component=apiserver
                   provider=kubernetes
Annotations:       <none>
Selector:          <none>
Type:              ClusterIP
IP:                10.8.0.1
Port:              https  443/TCP
TargetPort:        443/TCP
Endpoints:         34.66.217.13:443
Session Affinity:  None
Events:            <none>
```
## clusterIP 類型的 Service 
同樣的使用 `nginx` 做為範例。下面是實驗的 `yaml`。在前面文章有提到過，使用 `expose` 也可以達到創建 `Service` 資源。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 5
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
        role: backend
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.12
        ports:
        - containerPort: 80
--- # 區分資源，透過此方式可在一個檔案中定義不同多個資源
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
spec:
  selector:
    run: my-nginx # 關聯 POD 標籤
  type : ClusterIP
  ports:
  - port: 8080 # 暴露端口
    targetPort: 80 # 目標 POD 上的 Port
```

使用 `apply` 部署

```shell
$ kubectl apply -f nginx-service-demo.yaml
deployment.apps/my-nginx created
service/nginx created
```

使用 `get` 查看我們部署的 `service` 資源。`TYPE` 選擇是 `ClusterIP` 此值也是預設值；`CLUSTER-IP` 為自動分配，也可以透過 `spec` 中 `clusterIP` 屬性定義；`PORT` 是由 `port` 定義，協定預設為 `TCP`，可以在 `ports` 屬性下用 `protocol` 定義。`describe` 顯示詳細訊息，`Endpoints` 自動幫我們關聯，並維護相關的 `POD` 資源，可以對應下面 `get pods` 的結果。我們定義的 `Service` 類型只可以透過此 `IP` 接受從集群内的客户端 `POD` 中的請求。

```shell
$ kubectl get svc nginx # svc 為 service 縮寫
NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
nginx   ClusterIP   10.8.5.220   <none>        8080/TCP   27s
$ kubectl describe svc nginx
Name:              nginx
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          run=my-nginx
Type:              ClusterIP
IP:                10.8.5.220
Port:              <unset>  8080/TCP
TargetPort:        80/TCP
Endpoints:         10.4.1.28:80,10.4.2.33:80,10.4.2.34:80 + 2 more...
Session Affinity:  None
Events:            <none> 
$ kubectl get endpoints # 取得 endpoints
NAME         ENDPOINTS                                            AGE
kubernetes   34.66.217.13:443                                     15d
nginx        10.4.1.28:80,10.4.2.33:80,10.4.2.34:80 + 2 more...   35m
$ kubectl get pods -l run=my-nginx -o custom-columns=NAME:.metadata.name,hostIP:.status.hostIP,podIP:.status.podIP
NAME                        hostIP        podIP
my-nginx-74585d7fc6-jv2tz   10.128.0.21   10.4.1.28
my-nginx-74585d7fc6-mjnbs   10.128.0.23   10.4.2.34
my-nginx-74585d7fc6-rhhtb   10.128.0.24   10.4.3.95
my-nginx-74585d7fc6-vsbxt   10.128.0.23   10.4.2.35
my-nginx-74585d7fc6-zq6nm   10.128.0.23   10.4.2.33
```

整個請求會是 `Pod <---> Endpoint(tcp:80) <---> Service(tcp:8080, with VIP)`。而 `Service` 在選擇 `POD` 時是透過 `Iptables` 的設定隨機選擇。

## 向 Service 請求服務

### POD 存取
使用以下 `yaml` 模擬一個客戶端請求
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: test
    image: hwchiu/netutils
```

建立好客戶端的 `POD` 後，我們使用 `curl` 去請求。
```shell
$ kubectl exec -it myapp-pod -- bash
root@myapp-pod:/# curl 10.8.5.220:8080 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### 節點存取

在 `GKE` 上節點是由 `VM` 組成，因此我們使用 `ssh` 遠端到節點。
```shell
$ kubectl get node
NAME                                       STATUS   ROLES    AGE     VERSION
gke-cluster-1-default-pool-7dc8b11b-8kvr   Ready    <none>   30h     v1.15.12-gke.2
gke-cluster-1-default-pool-7dc8b11b-cxs1   Ready    <none>   15d     v1.15.12-gke.2
gke-cluster-1-default-pool-7dc8b11b-r7rg   Ready    <none>   3d17h   v1.15.12-gke.2
$ gcloud compute ssh --project  sunny-catwalk-286908  --zone us-central1-c  gke-cluster-1-default-pool-7dc8b11b-cxs1
```

請求
```shell
...@gke-cluster-1-default-pool-7dc8b11b-cxs1 ~ $ curl 10.8.5.220:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```


最後透過下面這張圖我們來理解上述的實驗

<!-- ![](../assets/img/k8s/K8s-service-cluster.jpg) -->
{{< figure src="/images/k8s/K8s-service-cluster.jpg" width="auto" height="auto">}}

當中紅色虛線表示無法存取，原因有兩種
1. `CLUSTER-IP` 無法知道如何送到集群中，從以下的路由表來看並沒有記錄這 `CLUSTER-IP` 的去向
```shell
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
```
2. 假設設定路由表，可以送到 `CLUSTER-IP`，當中會被 `kube-proxy` 阻擋


這邊補充，`Service` 中 `Session affinity`，可以將來自同一客戶端的請求轉發至同一個後端的 POD，但這會影響負載均衡的調度。透過 `explain` 查詢其內容如下，`ClientIP` 會基於客户端 `IP` 識別客户端，把同一個來源 `IP` 請求都都調度到同一個 `POD`，也可配合 `clientIP` 下的 `timeoutSeconds` 屬性設定效果的時間。

```shell
$ kubectl explain svc.spec.sessionAffinityConfig
KIND:     Service
VERSION:  v1

RESOURCE: sessionAffinityConfig <Object>

DESCRIPTION:
     sessionAffinityConfig contains the configurations of session affinity.

     SessionAffinityConfig represents the configurations of session affinity.

FIELDS:
   clientIP     <Object>
     clientIP contains the configurations of Client IP based session affinity.

$ kubectl explain svc.spec.sessionAffinity
KIND:     Service
VERSION:  v1

FIELD:    sessionAffinity <string>

DESCRIPTION:
     Supports "ClientIP" and "None". Used to maintain session affinity. Enable
     client IP based session affinity. Must be ClientIP or None. Defaults to
     None. More info:
     https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies
```


## NodePort 類型的 Service

簡單來說就是外部可藉由每個節點的 IP 來存取內部應用程式。我們定義以下實驗的 `yaml`。`type` 類型需要指定，`nodePort` 屬性可設定，不設定預設會是從 30000 至 32767 隨機選取。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-demo
spec:
  type: NodePort 
  ports:
  - port: 80
    targetPort: 8080 # Service Port 80 -> POD: 8080
    nodePort: 30080 # Node Port 30080  -> POD: 8080
  selector:
    app: hello-kubernetes
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-kubernetes
  template:
    metadata:
      labels:
        app: hello-kubernetes
    spec:
      containers:
      - name: hello-kubernetes
        image: paulbouwer/hello-kubernetes:1.7
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-client
  labels:
    app: client
spec:
  containers:
  - name: client
    image: hwchiu/netutils
```

使用 `apply` 部屬，因為上面的 `yaml` 檔放置同一個目錄下，這時可使用 `-R` 進行遞規。

```shell
$ kubectl apply -R -f nodePort/
deployment.apps/hello-kubernetes created
deployment.apps/client created
service/nodeport-demo created
```


使用 `get` 查看 `nodePort` 資源
```shell
$ kubectl get svc nodeport-demo -o wide
NAME            TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE   SELECTOR
nodeport-demo   NodePort   10.8.0.16    <none>        80:30080/TCP   12s   app=hello-kubernetes```

```shell
$ kubectl describe svc nodeport-demo
Name:                     nodeport-demo
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=hello-kubernetes
Type:                     NodePort
IP:                       10.8.0.16
Port:                     <unset>  80/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30080/TCP
Endpoints:                10.4.1.31:8080,10.4.2.40:8080,10.4.3.99:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

### POD 存取
因為 `NodePort` 繼承 `ClusterIP` 的功能，多實現了節點上的存取。

```shell
$ kubectl get pods -l app=client -o wide # 查看 POD
NAME                     READY   STATUS    RESTARTS   AGE     IP           NODE                                       NOMINATED NODE   READINESS GATES
client-99cbcdc74-qz99k   1/1     Running   0          19m     10.4.3.97    gke-cluster-1-default-pool-7dc8b11b-8kvr   <none>           <none>
pod-client               1/1     Running   0          2m25s   10.4.3.100   gke-cluster-1-default-pool-7dc8b11b-8kvr   <none>           <none>
```

驗證
```shell
$ kubectl exec -it pod-client curl nodeport-demo # 利用 DNS 解析
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
<!DOCTYPE html>
<html>
<head>
    <title>Hello Kubernetes!</title>
    <link rel="stylesheet" type="text/css" href="/css/main.css">
    <link rel="stylesheet" href="https://fonts.googleapis.com/css?family=Ubuntu:300" >
</head>
<body>

  <div class="main">
    <img src="/images/kubernetes.png"/>
    <div class="content">
      <div id="message">
  Hello world!
</div>
<div id="info">
  <table>
    <tr>
      <th>pod:</th>
      <td>hello-kubernetes-5b89dbbc4b-9hbgv</td>
    </tr>
    <tr>
      <th>node:</th>
      <td>Linux (4.19.112+)</td>
    </tr>
  </table>

</div>
    </div>
  </div>

</body>
```

```shell
$ kubectl exec -it pod-client curl 10.8.0.16 # 透過 ClusterIP
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
<!DOCTYPE html>
<html>
<head>
    <title>Hello Kubernetes!</title>
    <link rel="stylesheet" type="text/css" href="/css/main.css">
    <link rel="stylesheet" href="https://fonts.googleapis.com/css?family=Ubuntu:300" >
</head>
<body>

  <div class="main">
    <img src="/images/kubernetes.png"/>
    <div class="content">
      <div id="message">
  Hello world!
</div>
<div id="info">
  <table>
    <tr>
      <th>pod:</th>
      <td>hello-kubernetes-5b89dbbc4b-9hbgv</td>
    </tr>
    <tr>
      <th>node:</th>
      <td>Linux (4.19.112+)</td>
    </tr>
  </table>

</div>
    </div>
  </div>

</body>
```
### 節點存取

從 `gcloud` 獲取對外 `IP`，當然也可以用 `get node` 方式。
```shell
$ gcloud compute instances list
NAME                                      ZONE           MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
gke-cluster-1-default-pool-7dc8b11b-8kvr  us-central1-c  e2-medium                  10.128.0.24  35.238.113.123  RUNNING
gke-cluster-1-default-pool-7dc8b11b-cxs1  us-central1-c  e2-medium                  10.128.0.21  34.67.164.169   RUNNING
gke-cluster-1-default-pool-7dc8b11b-r7rg  us-central1-c  e2-medium                  10.128.0.23  34.72.35.109    RUNNING
$ kubectl get node -o custom-columns='NAME:.metadata.name,EXTERNAL-IP:.status.addresses[?(@.type=="ExternalIP")].address,INTERNAL-IP:.status.addresses[?(@.type=="InternalIP")].address,POD-CIDR:.spec.podCIDR'
NAME                                       EXTERNAL-IP      INTERNAL-IP   POD-CIDR
gke-cluster-1-default-pool-7dc8b11b-8kvr   35.238.113.123   10.128.0.24   10.4.3.0/24
gke-cluster-1-default-pool-7dc8b11b-cxs1   34.67.164.169    10.128.0.21   10.4.1.0/24
gke-cluster-1-default-pool-7dc8b11b-r7rg   34.72.35.109     10.128.0.23   10.4.2.0/24
```

先遠端至一個 `VM` 節點上，並使用 `curl` 驗證
```shell
$ curl 10.128.0.24:30080
<!DOCTYPE html>
<html>
<head>
    <title>Hello Kubernetes!</title>
    <link rel="stylesheet" type="text/css" href="/css/main.css">
    <link rel="stylesheet" href="https://fonts.googleapis.com/css?family=Ubuntu:300" >
</head>
<body>

  <div class="main">
    <img src="/images/kubernetes.png"/>
    <div class="content">
      <div id="message">
  Hello world!
</div>
<div id="info">
  <table>
    <tr>
      <th>pod:</th>
      <td>hello-kubernetes-5b89dbbc4b-9hbgv</td>
    </tr>
    <tr>
      <th>node:</th>
      <td>Linux (4.19.112+)</td>
    </tr>
  </table>

</div>
    </div>
  </div>

</body>
```


簡單來說上面實驗可以說是 `client <---> NodeIP:NodePort <---> ClusterIP:ServicePort <---> PodIP:containePort`。


這邊補充在 `GKE` 下每個 `VM` 都會有 `EXTERNAL_IP`，但是要讓它可以作用至 `NodePort` 需要設置以下防火牆，設置完之後透過瀏覽器輸入 `EXTERNAL_IP:NodePort` 即可存取相關 `POD`。

```shell
$ gcloud compute firewall-rules create nodeport-svc-rule --allow=tcp:{NodePort}
```

## Headless 類型的 Service
在這架構下，`loadbalancing` 由自己決定，但它會取得所有 `endopints` 資訊也就是直接調度 `POD` 資訊。設定方面是將 `clusterIP` 設為 `None`，並非 `type`。而 `kube-proxy` 不會處裡此 `service` 相關的資訊像是負載均衡。只要在前端有*服務發現*功能時，可以省去 `clusterIP` 需求，同樣的此類型也是透過*標籤選擇器*，去決定 `endopints`，最重要的是集群中 `DNS` 服務 `A` 紀錄會解析到後端 `POD` 的 `IP`。假設無設置標籤選擇器時，則不會配置 `endopints`，`DNS` 服務會查找和配置。


```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-demo
spec:
  clusterIP: None # headless 須為 None
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: hello-kubernetes
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes-headless
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-kubernetes
  template:
    metadata:
      labels:
        app: hello-kubernetes
    spec:
      containers:
      - name: hello-kubernetes
        image: paulbouwer/hello-kubernetes:1.7
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-client-headless
  labels:
    app: client
spec:
  containers:
  - name: client
    image: hwchiu/netutils
```

部署
```shell
$ kubectl apply -R -f .
```

查看 `service` 資源，`CLUSTER-IP` 為 `None`，同時用 `describe` 和 `get pods` 觀察標籤選擇器的結果並對應 `Endpoints`。
```shell
$ kubectl get svc headless-demo
NAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
headless-demo   ClusterIP   None         <none>        80/TCP    3m
```

```shell
$ kubectl describe svc headless-demo
Name:              headless-demo
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=hello-kubernetes
Type:              ClusterIP
IP:                None
Port:              <unset>  80/TCP
TargetPort:        8080/TCP
Endpoints:         10.4.1.32:8080,10.4.2.41:8080,10.4.3.101:8080
Session Affinity:  None
Events:            <none>
$ kubectl get pods -l app=hello-kubernetes -o wide -L app # 可對照 Endpoints
NAME                                         READY   STATUS    RESTARTS   AGE     IP           NODE                                       NOMINATED NODE   READINESS GATES   APP
hello-kubernetes-headless-5b89dbbc4b-2gbs2   1/1     Running   0          6m37s   10.4.1.32    gke-cluster-1-default-pool-7dc8b11b-cxs1   <none>           <none>            hello-kubernetes
hello-kubernetes-headless-5b89dbbc4b-7hqnz   1/1     Running   0          6m37s   10.4.2.41    gke-cluster-1-default-pool-7dc8b11b-r7rg   <none>           <none>            hello-kubernetes
hello-kubernetes-headless-5b89dbbc4b-vb2zq   1/1     Running   0          6m37s   10.4.3.101   gke-cluster-1-default-pool-7dc8b11b-8kvr   <none>           <none>            hello-kubernetes
```

### POD 資源發現

前面提到過說可用 `DNS` 去解析相關 `POD` 的 `IP` 位置，因此這建立的 `service` 資源是直接發現 `POD`。藉由 `pod-client-headless` 我們嘗試使用 `nslookup` 解析，結果正是標籤選擇器(selector) 所選擇的 `POD` 的 `IP`，因此它是直接對 `POD`，不必有轉發動作。

```shell
$ kubectl get pods -l app=client
NAME                  READY   STATUS    RESTARTS   AGE
pod-client-headless   1/1     Running   0          8m46s
$ kubectl exec -it pod-client-headless -- bash
root@pod-client-headless:/# nslookup headless-demo
Server:         10.8.0.10
Address:        10.8.0.10#53

Name:   headless-demo.default.svc.cluster.local
Address: 10.4.1.32
Name:   headless-demo.default.svc.cluster.local
Address: 10.4.2.41
Name:   headless-demo.default.svc.cluster.local
Address: 10.4.3.101   
```


這邊為了顯示出此 `service` 透過域名方式可綁定 `POD`，就算該 `POD` 被重新調度。這邊將 `Deployment` 的部署改用 `StatefulSet` 方式呈現較為直觀，可以將它想像成是一個有序的服務，`StatefulSet` 在後面文章後會提到，這邊不知道不影響對 `headless` 理解。

觀察部署的 `POD`
```shell
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
client-99cbcdc74-h2x7x   1/1     Running   0          10s
hello-kubernetes-0       1/1     Running   0          10s
hello-kubernetes-1       1/1     Running   0          8s
hello-kubernetes-2       1/1     Running   0          7s
```

我們用 `nslookup` 去解析 `hello-kubernetes-0` 服務，得到 `10.4.2.44` 的 `POD` 地址。 
```shell
$ kubectl exec -it client-99cbcdc74-h2x7x -- bash
root@client-99cbcdc74-h2x7x:/# nslookup hello-kubernetes-0.headless-demo
Server:         10.8.0.10
Address:        10.8.0.10#53

Name:   hello-kubernetes-0.headless-demo.default.svc.cluster.local
Address: 10.4.2.44

root@client-99cbcdc74-h2x7x:/#
```

嘗試將 `hello-kubernetes-0` 刪除，並用 `nslookup` 觀察，此結果可以說 `POD` 被重新調度後，透過 `DNS` 發現方式還是可以將請求丟到同一個 `POD` 上。

```shell
$ kubectl delete pods hello-kubernetes-0
pod "hello-kubernetes-0" deleted
# 在解析一次
root@client-99cbcdc74-h2x7x:/# nslookup hello-kubernetes-0.headless-demo
Server:         10.8.0.10
Address:        10.8.0.10#53

Name:   hello-kubernetes-0.headless-demo.default.svc.cluster.local
Address: 10.4.2.45
```

## ExternalName 類型的 Service
這也算是 `headless` 的一種，在資源定義上，`type` 是 `ExternalName`，`externalName` 要指定集群外部的服務，需要是一個 `CNAME`，這樣 `POD` 才能去存取。此資源不必要定義標籤選擇器(selector)和 `clusterIP`，因此很像 `headless`。這邊較為簡易就不演示了。


## 參考資源

- [官方 service](https://kubernetes.io/zh/docs/concepts/services-networking/service/)
- Kubernetes in action