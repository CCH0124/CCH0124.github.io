---
title: kubernetes - day19
date: 2020-09-09
description: "Service 和 Ingress part03"
tags: [kubernetes]
draft: false
---

## Loadbalance 類型的 Service 
上一章節實作了 `nodePort` 類型的資源，它可讓集群外部存取，但 `nodePort` 資源有著一個缺點，要存取時須知道集群中至少一個節點 `IP`，該結點如果故障得要取得其它節點的 `IP`。這問題可藉由在集群外部部署一個負載均衡器，可以經由它存取外部客户端請求同時調度到集群中相應的 `nodePort`，這類型的資源是 `LoadBalancer`。我們透過 `GKE` 環境實作。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: loadbalance-service
spec:
  type: LoadBalancer
  selector:
    app: hello-kubernetes
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 9376
      nodePort: 32223
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes-loadbalance
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

查看建立的 `LoadBalancer` 資源，此資源會分配一個對外存取的 `IP`(EXTERNAL-IP)。
```shell
$ kubectl get svc -w
NAME                  TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
headless-demo         ClusterIP      None          <none>        80/TCP           12h
kubernetes            ClusterIP      10.8.0.1      <none>        443/TCP          16d
loadbalance-service   LoadBalancer   10.8.13.147   <pending>     8080:30862/TCP   7s
loadbalance-service   LoadBalancer   10.8.13.147   34.70.146.27   8080:30862/TCP   35s
```
用 `describe` 查看建立的資源，其資訊和 `nodePort` 等都大同小異
```shell
$ kubectl describe svc loadbalance-service
Name:                     loadbalance-service
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=hello-kubernetes
Type:                     LoadBalancer
IP:                       10.8.13.147
LoadBalancer Ingress:     34.70.146.27
Port:                     <unset>  9376/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  32223/TCP
Endpoints:                10.4.1.37:8080,10.4.2.47:8080,10.4.3.111:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason                Age                  From                Message
  ----    ------                ----                 ----                -------
  Normal  EnsuredLoadBalancer   5m39s (x2 over 10m)  service-controller  Ensured load balancer
  Normal  EnsuringLoadBalancer  17s (x3 over 11m)    service-controller  Ensuring load balancer
```

實際上 `LoadBalancer` 也繼承了 `nodePort` 透過以下驗證，`NodePort` 沒有指定 `Port` 時系統會自動分配一個。

```shell
$ gcloud compute instances list
NAME                                      ZONE           MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
gke-cluster-1-default-pool-7dc8b11b-8kvr  us-central1-c  e2-medium                  10.128.0.24  35.238.113.123  RUNNING
gke-cluster-1-default-pool-7dc8b11b-cxs1  us-central1-c  e2-medium                  10.128.0.21  34.67.164.169   RUNNING
gke-cluster-1-default-pool-7dc8b11b-r7rg  us-central1-c  e2-medium                  10.128.0.23  34.72.35.109    RUNNING
$ kubectl exec -it client-99cbcdc74-4srn9 bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@client-99cbcdc74-4srn9:/# curl 10.128.0.24:32223
<!DOCTYPE html>
<html>
<head>
    <title>Hello Kubernetes!</title>
...
      <th>pod:</th>
      <td>hello-kubernetes-loadbalance-5b89dbbc4b-qccrq</td>
...
```

接著我們從本機的瀏覽器輸入 `34.70.146.27:9376`，結果如下，可以不斷的做請求會看到均衡負載效果。

<!-- ![](../assets/img/k8s/K8s-loadbalancer.png) -->
{{< figure src="/images/k8s/K8s-loadbalancer.jpg" width="auto" height="auto">}}


>有一個字段 `loadBalancerSourceRanges` 它可限制負載均衡器可存取的客戶端 IP 範圍


## Ingress 資源

在 `K8s` 提供了兩種內建的負載均衡機制，分別針對的是*傳輸層*和*應用層*，應用層相較於傳輸層可以做到 `URL` 的映射或 `SSL` 的卸載等功能。我們介紹的 `nodePort`、`LoadBalancer` 等都是屬於傳輸層的負載均衡，這邊要說的 `Ingress` 是一個使用 `DNS` 或 `URL` 方式將請求轉發至指定的 `Service` 資源上。但，`Ingress` 是*規則*，無法對流量進行處理，為了能達到請求流量等功能，會配合著 `Ingress Controller` 這個資源。`Ingress Controller` 可以由有反向代理功能的應用程式實現，如 `Nginx`、`HAProxy` 等，在 `K8s` 中它也是一個 `POD`，會與被代理的 `POD` 在相同的網路中。如果 `Ingress` 進行流量分發時，`Ingress Controller` 可以將 `Ingress` 定義的規則直接轉發至 `POD` 上，這當中繞過了 `Service` 資源因此減少了 `kube-proxy` 轉發開銷。以下整理了 `Ingress` 主要資源

- Ingress Resource
    - 由 `Kubernetes`，就是一個資源清單
- Ingress Controller
    - `Kubernetes` 本身沒有此資源，需仰賴第三方資源
    - 觀察 `Ingress Resource`，有資源清單有無更新，當有變動時會送給 `Ingress Server`
- Ingress Server
    - `Kubernetes` 本身沒有此資源，需仰賴第三方資源
    - 接收請求並轉發至後端服務
 

### 創建 Ingress 資源並與 Nginx Ingress 控制器共舞

`Ingress` 是基於 `virtual hosting` 或 URL 的轉發規則。以下面的 `Ingress`  資源 `yaml` 為例，會把符合  `test.com/v1/` 的規則送至 `hellok8s` 的 `Service` 資源，以此類推。這邊大致上說明一下字段的含意。從 `spec` 來看最為重要的是 `rule` 和 `backend`。`rule` 就是拿來定義規則，只要規則沒有符合則會將其送至預設定義的 `backend` 屬性，`backend` 字段有 `serviceName` 和 `servicePort` 分別是對應 `Service` 資源名稱和 `Port`。

`Ingress` 的類型還有分以下
- 基於 `URL`
```yaml
rules:
- host: test.com
    http:
      paths:
      - path: /v1/
        backend:
          serviceName: hellok8s
          servicePort: 80
```
- 基於 `virtual hosting`
```yaml
rules:
- host: api.ik8s.io
    http:
      paths:
      - backend:
          serviceName: api
          servicePort: 80

```
- 基於 `TLS`
```yaml
tls:
- hosts:
  - tomcat.lab
  secretName: tomcat-secret
rules:
- host: tomcat.lab
  http:
    paths:
    - path: /
      backend:
        serviceName: tomcat-svc
        servicePort: 80
```

以下[範例](https://github.com/hwchiu/hiskio-course/tree/master/introduction/ingress)是來自 hwchiu 大大的，`README.md` 檔案內容也需要執行，它是 `Nginx Ingress` 控制器。這邊拿 Ingress 的資源講解，其它的都在前面文章都有提到。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-http
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: test.com
    http:
      paths:
      - path: /v1/
        backend:
          serviceName: hellok8s
          servicePort: 80
      - path: /v2/
        backend:
          serviceName: httpd
          servicePort: 80
  - host: hello.com
    http:
      paths:
      - backend:
          serviceName: hellok8s
          servicePort: 80
  - host: httpd.com
    http:
      paths:
      - backend:
          serviceName: httpd
          servicePort: 80
```


我們簡易的把整個架構用下圖表示

<!-- ![](../assets/img/k8s/K8s-ingress-01.jpg) -->
{{< figure src="/images/k8s/K8s-ingress-01.jpg" width="auto" height="auto">}}


同樣的上述的檔案使用 `apply` 進行部署。這邊比較重要的是 `Service` 因此這邊會以 `Service` 資源做觀察。

```shell
$ kubectl get svc # 有部署兩個應用程式，也分別為兩個應用程式建立 service 資源
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
hellok8s     ClusterIP   10.8.2.77     <none>        80/TCP    22s
httpd        ClusterIP   10.8.10.193   <none>        80/TCP    22s
$ kubectl get endpoints
NAME         ENDPOINTS                                       AGE
hellok8s     10.4.1.40:8080,10.4.2.53:8080,10.4.3.116:8080   58s # 對應的 POD 資訊
httpd        10.4.1.41:80,10.4.2.54:80,10.4.3.117:80         58s # 對應的 POD 資訊
```

### NodePort

```shell
$ kubectl -n ingress-nginx get svc
NAME            TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.8.7.41    <none>        80:32739/TCP,443:31198/TCP   117s
```

部署的 `ingress` 資源，其 `ADDRESS` 對應的是 `ingress-nginx` 控制器的 `NodePort`，此兩個是相互關連的，且可以透過 `NodePort` 給外界存取。因此這邊實驗是 `client <--> NodePort <--> Ingress <--> Service`。

```shell
$ kubectl get ingress # 取得 Ingress 資訊
NAME           HOSTS                          ADDRESS     PORTS   AGE
ingress-http   test.com,hello.com,httpd.com   10.8.7.41   80      2m51s
$  kubectl describe ingress ingress-http # 詳細訊息
Name:             ingress-http
Namespace:        default
Address:          10.8.7.41
Default backend:  default-http-backend:80 (10.4.1.4:8080)
Rules:
  Host        Path  Backends
  ----        ----  --------
  test.com
              /v1/   hellok8s:80 (10.4.1.40:8080,10.4.2.53:8080,10.4.3.116:8080)
              /v2/   httpd:80 (10.4.1.41:80,10.4.2.54:80,10.4.3.117:80)
  hello.com
                 hellok8s:80 (10.4.1.40:8080,10.4.2.53:8080,10.4.3.116:8080)
  httpd.com
                 httpd:80 (10.4.1.41:80,10.4.2.54:80,10.4.3.117:80)
Annotations:  nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type     Reason     Age                    From                      Message
  ----     ------     ----                   ----                      -------
  Normal   UPDATE     11m (x4 over 106m)     nginx-ingress-controller  Ingress default/ingress-http
  Warning  Translate  2m21s (x23 over 106m)  loadbalancer-controller   error while evaluating the ingress spec: service "default/hellok8s" is type "ClusterIP", expected "NodePort" or "LoadBalancer"; service "default/httpd" is type "ClusterIP", expected "NodePort" or "LoadBalancer"; service "default/hellok8s" is type "ClusterIP", expected "NodePort" or "LoadBalancer"; service "default/httpd" is type "ClusterIP", expected "NodePort" or "LoadBalancer"
```

這邊從 `Cloud Shell` 遠端至一台節點並設定 `/etc/hosts`，它為一個靜態對找表，預設是往這邊搜尋，我們當下打 `test.com` 它會導到 `10.8.7.41`，當沒結果才往外部 `DNS` 做搜尋。

```shell
@gke-cluster-1-default-pool-7dc8b11b-8kvr$ sudo vi /etc/hosts
10.8.7.41 test.com
10.8.7.41 hello.com
10.8.7.41 httpd.com
```

驗證
```shell
@gke-cluster-1-default-pool-7dc8b11b-8kvr$ curl test.com/v1/
...
    <tr>
      <th>pod:</th>
      <td>hello-kubernetes-5b89dbbc4b-t7bvr</td>
    </tr>
...
@gke-cluster-1-default-pool-7dc8b11b-8kvr$ curl test.com/v2/
<html><body><h1>It works!</h1></body></html>
```

### Loadbalance

`service` 檔案資源清單定義的 `Ingress` 是 `NodePort` 這邊嘗試用 `LoadBalancer`。

```shell
$ kubectl -n ingress-nginx get svc
NAME            TYPE           CLUSTER-IP   EXTERNAL-IP    PORT(S)                      AGE
ingress-nginx   LoadBalancer   10.8.7.41    34.70.146.27   80:32739/TCP,443:31198/TCP   26m
$ kubectl get ingress
NAME           HOSTS                          ADDRESS        PORTS   AGE
ingress-http   test.com,hello.com,httpd.com   34.70.146.27   80      26m
$ kubectl describe ingress ingress-http # 詳細資訊
Name:             ingress-http
Namespace:        default
Address:          34.70.146.27
Default backend:  default-http-backend:80 (10.4.1.4:8080)
Rules:
  Host        Path  Backends
  ----        ----  --------
  test.com
              /v1/   hellok8s:80 (10.4.1.40:8080,10.4.2.53:8080,10.4.3.116:8080)
              /v2/   httpd:80 (10.4.1.41:80,10.4.2.54:80,10.4.3.117:80)
  hello.com
                 hellok8s:80 (10.4.1.40:8080,10.4.2.53:8080,10.4.3.116:8080)
  httpd.com
                 httpd:80 (10.4.1.41:80,10.4.2.54:80,10.4.3.117:80)
Annotations:  nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type     Reason     Age                  From                     Message
  ----     ------     ----                 ----                     -------
  Warning  Translate  5m6s (x19 over 89m)  loadbalancer-controller  error while evaluating the ingress spec: service "default/hellok8s" is type "ClusterIP", expected "NodePort" or "LoadBalancer"; service "default/httpd" is type "ClusterIP", expected "NodePort" or "LoadBalancer"; service "default/hellok8s" is type "ClusterIP", expected "NodePort" or "LoadBalancer"; service "default/httpd" is type "ClusterIP", expected "NodePort" or "LoadBalancer"
```


這邊從 `Cloud Shell` 設定 `/etc/hosts`，它為一個靜態對找表，預設是往這邊搜尋，我們當下打 `test.com` 它會導到 `34.70.146.27`，當沒結果才往外部 `DNS` 做搜尋。

```shell
echo "34.70.146.27 test.com" | sudo tee -a /etc/hosts
echo "34.70.146.27 hello.com" | sudo tee -a /etc/hosts
echo "34.70.146.27 httpd.com" | sudo tee -a /etc/hosts
```

使用 `curl` 驗證
```shell
$ curl test.com/v1/ # 只要 URL 符合 test.com/v1/ 都會打到 test.com/v1/ 規則的 Service
<!DOCTYPE html>
<html>
...
  <table>
    <tr>
      <th>pod:</th>
      <td>hello-kubernetes-5b89dbbc4b-t7bvr</td>
    </tr>
    <tr>
      <th>node:</th>
      <td>Linux (4.19.112+)</td>
...
$ curl test.com/v2/
<html><body><h1>It works!</h1></body></html>
$ curl test.com/v1 # 預設的 backend
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.17.8</center>
</body>
</html>
```

我們將上面請求過程用下面這張圖示意

<!-- ![](../assets/img/k8s/K8s-ingress-example.jpg) -->
{{< figure src="/images/k8s/K8s-ingress-example.jpg" width="auto" height="auto">}}


觀察 `Nginx` 控制器有做了什麼。

```shell
$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-7fcf8df75d-p5s5b   1/1     Running   0          53m
$ kubectl exec -it -n ingress-nginx nginx-ingress-controller-7fcf8df75d-p5s5b bash
bash-5.0$ vi /etc/nginx/nginx.conf # 控制器幫我們設定好 nginx 設定檔了!!
...
 ## start server test.com                                    
        server {                                                    
                server_name test.com ;                              
                                                                    
                listen 80  ;          
                listen 443  ssl http2 ;
                                       
                set $proxy_upstream_name "-";
                                             
                ssl_certificate_by_lua_block {
                        certificate.call()    
                }                             
                                              
                location ~* "^/v2/" {             
                                              
                        set $namespace      "default";
                        set $ingress_name   "ingress-http";
                        set $service_name   "httpd";       
                        set $service_port   "80";          
                        set $location_path  "/v2/";        
                                                           
                        rewrite_by_lua_block {             
                                lua_ingress.rewrite({      
                                        force_ssl_redirect = false,
                                        ssl_redirect = true,       
                                        force_no_ssl_redirect = false,
                                        use_port_in_redirects = false,
                                })                                    
...
```

這邊有 google 的 [qwiklab](https://github.com/CCH0124/qwiklab/blob/master/Kubernetes%20Solutions/02_NGINX%20Ingress%20Controller%20on%20Google%20Kubernetes%20Engine.md) 文章，也是針對 Nginx `Ingress` 去實作。


## 參考資源

- [官方 - 建立外部均衡負載](https://kubernetes.io/zh/docs/tasks/access-application-cluster/create-external-load-balancer/)
- [官方 - Ingress](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/)