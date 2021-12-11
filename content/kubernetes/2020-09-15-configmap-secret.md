---
title: kubernetes - day25
date: 2020-09-15
description: "ConfigMap 和 Secret"
tags: [kubernetes]
draft: false
---


這邊在大致說明 `ConfigMap` 和 `Secret`，它們也屬於 `volume` 的一種，前者通常是以配置的數據等為主；後者則是以密鑰等為主。為什麼說是 `volume` 的一種呢？透過掛載方式那些像是 TLS、SSL、CA 證書或一些配置可以與容器的 `image` 做到解偶。在傳統的 Docker 來說，一個容器中的應用程式不大可能使用預設配置，因此我們會透過*環境變數*或*卷(volume)* 等方式進行配置。在 K8s 上也利用 `ConfigMap` 和 `Secret` 完成一些容器所需的配置，透過這種方式也可避免被重新調度後導致配置內容遺失的問題。


## 容器參數配置

透過 `explain` 方式，可以在容器等級字段中找到 `args`、`command` 的屬性，它們是用來定義容器要運行的指令(command)和傳遞參數(args)。[官方](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#running-a-command-in-a-shell)也有說明，`command` 對應於 `Dockerfile` 中的 `ENTRYPOINT`；`args` 則對應於 `CMD`。

>The command field corresponds to entrypoint in some container runtimes. Refer to the Notes below.
>The environment variable appears in parentheses, "$(VAR)". This is required for the variable to be expanded in the command or args field.

## 容器環境變數配置

配置容器中檔案另一種方式就是利用環境變數傳值方式。在容器等級字段中有 `env` 的屬性，可以以 `List` 方式去定義很多的環境變數。透過 `explain` 查看後它由以下組成
- name
    - 環境變數的名稱
- value
    - 環境變數的值
    - 可以使用 `$(NAME)` 引用定義過的環境變數名稱值
    - [官方範例](https://kubernetes.io/docs/tasks/inject-data-application/define-interdependent-environment-variables/)
- valueFrom
    - 環境變數的值從哪邊引用，如 `PDO` 名稱或是從 [ConfigMap](https://kubernetes.io/docs/tasks/inject-data-application/podpreset/#pod-spec-with-configmap-example) 和 [Secret](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/#define-container-environment-variables-using-secret-data) 引用
    - 有 `value` 時 `valueFrom` 不能被定義，反過來也是

上面所提到容器如何寫入環境變數，但是容器這些不是能夠定義了嗎？為什麼還需要 `ConfigMap` 這些呢 ? 上面提到的方式無法為運行的容器即時更新環境變數，而 `ConfigMap` 可以解決此問題。



## ConfigMap

`ConfigMap` 增強了對 `POD` 環境設置的靈活性，其設置方式有透過 `volume` 或是 `valueFrom` 方式。`ConfigMap` 是屬於 `Map` 型態也就是以 Key-Value 進行定義，所以只要透過 `ConfigMap` 方式定義環境設置針對於相同的 `POD` 不同環境會有很大的靈活性。

### 建立 ConfigMap

透過 `creat` 建立

```shell
$ kubectl create configmap test --from-literal=key1=value1 --from-literal=key2=value2 # test 是 ConfigMap 的名稱
configmap/test created
```

查看 test 資源，當中 `data` 字段表示為定義的環境變數

```shell
$ kubectl get configmaps test -o yaml
apiVersion: v1
data:
  key1: value1
  key2: value2
kind: ConfigMap
metadata:
  creationTimestamp: "2020-09-19T02:59:16Z"
  name: test
  namespace: default
  resourceVersion: "7148417"
  selfLink: /api/v1/namespaces/default/configmaps/test
  uid: caddf30e-195b-45d2-9710-8cad2cab1e6c
```

假設是一個配置檔也可以使用檔案方式引入，可參考[官方](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-files)，該鏈接中許很多使用範例。

下面是以資源清單方式定義，同樣的要使用時用 `apply` 或其它方式布署。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-demo
  namespace: default
data:
  log_level: INFO
  log_file: /var/log/sys.log
```
### ConfigMap 傳遞變數至容器

我們使用上面建立的 test `ConfigMap` 進行實驗。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo
  namespace: default
  labels:
    app: myapp
    tier: test
spec:
  restartPolicy: Never
  containers:
  - name: busybox
    image: busybox:latest
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(KEY1)$(KEY2); sleep 3600;done"]
    env:
    - name: KEY1
      valueFrom:
        configMapKeyRef:
          name: test
          key: key1
    - name: KEY2
      valueFrom:
        configMapKeyRef:
          name: test
          key: key2
```

透過以下方式驗證是否有成為環境變數。

```shell
$ kubectl exec configmap-demo ps aux
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh -c while true; do echo value1value2; sleep 3600;done # value1value2 是我們定義的
    6 root      0:00 sleep 3600
    7 root      0:00 ps aux
$ kubectl exec configmap-demo printenv
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=configmap-demo
KEY1=value1 # this
KEY2=value2 # this
```

這邊使用 `volumes` 方式，這邊以檔案方式建立 `ConfigMap`

新增一個 nginx.www 檔案，內容如下
```shell
server {
        server_name cch.lab.com;
        listen 8080;
        root /data/web/html/;
}
```
透過 nginx.www 檔案產生一個 nginx-www 名稱的 `ConfigMap`
```shell
$ kubectl create configmap nginx-www --from-file=./configmap/nginx.www
configmap/nginx-www created
cchong0124@cloudshell:~ (sunny-catwalk-286908)$ kubectl get cm
NAME           DATA   AGE
nginx-config   2      6m
nginx-www      1      8s
$ kubectl get configmap nginx-www -o yaml # 取得 ConfigMap 資源訊息
apiVersion: v1
data:
  nginx.www: "server {\n\tserver_name cch.lab.com;\n\tlisten 8080; \n\troot /data/web/html/;\n}\n"
kind: ConfigMap
metadata:
  creationTimestamp: "2020-08-28T00:02:42Z"
  name: nginx-www
  namespace: default
  resourceVersion: "2803771"
  selfLink: /api/v1/namespaces/default/configmaps/nginx-www
  uid: 848af4cf-732f-4294-ae73-7d5fa75db3f9
```

以下是使用 `volumes` 的 `yaml` 檔
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm
  namespace: default
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: myapp
    image: nginx:1.18
    ports:
    - name: http
      containerPort: 80
    volumeMounts:
    - name: nginxconfig
      mountPath: /etc/nginx/config.d/
      readOnly: true
  volumes:
  - name: nginxconfig
    configMap:
      name: nginx-config
```
```shell
$ kubectl exec -it pod-cm -- /bin/sh
# cd /etc/nginx/config.d
# ls
nginx_host_name  nginx_port
# cat nginx_host_name
cch.lab.com#
# cat nginx_port
8080
```

用 `volume` 方式可以解決上個使用 `env` 的缺點，也就是可以對 `ConfigMap` 對象進行修改並映射到容器中，但過程中要通知 `API Server` 進行同步，因此需要一些時間。當中進入容器查看那些檔案，結果使用 `ls -al` 可以發現他是一個 `link`。

>ConfigMap 要比 POD 還要先建立否則會出現錯誤，除非使用 optional 字段。


## Secret 

用來存放敏感訊息，像是密碼、私鑰等，操作方式相似於 `ConfigMap`。它的內容是以編碼方式進行進行儲存，當在容器的環境數時將會自動解碼變成明文。


### 建立 Secret

同樣的使用 `create` 也能建立，其方式與 `ConfigMap` 差不多，也能使用檔案方式引用。
```shell
$ kubectl create secret generic mysql-root-passwd --from-literal=pwd=itachi
secret/mysql-root-passwd created

$ kubectl get secret # 查看 secret 建立的資源
NAME                  TYPE                                  DATA   AGE
default-token-gjdn4   kubernetes.io/service-account-token   3      8d
mysql-root-passwd     Opaque                                1      12s

$ kubectl describe secret mysql-root-passwd # 查看建立的 mysql-root-passwd 
Name:         mysql-root-passwd
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data # 不同於 ConfigMap 這邊部顯示數據
====
pwd:  6 bytes
```

相較於 `configmap` 在 `describe` 上 secret 不顯示數據。但用 `get` 方式並使用 `yaml` 輸出，則會顯示數據只不過 `secret` 將數據 `base64` 編碼

```shell
$ kubectl get secret mysql-root-passwd -o yaml
apiVersion: v1
data:
  pwd: aXRhY2hp
kind: Secret
metadata:
  creationTimestamp: "2020-08-28T00:47:46Z"
  name: mysql-root-passwd
  namespace: default
  resourceVersion: "2814321"
  selfLink: /api/v1/namespaces/default/secrets/mysql-root-passwd
  uid: 48cc0930-6005-449e-b035-87a2814abf02
type: Opaque
```

```shell
$ echo aXRhY2hp | base64 -d # 確實是 base64
itachi
```

以資源清單方式建立

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-demo
stringData:
  username: test
  password: passw0rd
type: Opaque
```

```shell
$ kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-nnghk   kubernetes.io/service-account-token   3      21d
mysql-root-passwd     Opaque                                1      19h
secret-demo           Opaque                                2      13s # this
```


同樣的使用 `env` 方式進行引用，下面是一個 `yaml` 範例檔，`volume` 方式則不演示，用法語 `ConfigMap` 相同。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret-1
  namespace: default
  labels:
    app: myapp
    tier: frontend
spec:
  containers:
  - name: myapp
    image: nginx:1.18
    ports:
    - name: http
      containerPort: 80
    env:
    - name: NGINX_SERVER_PORT
      valueFrom:
        configMapKeyRef:
          name: nginx-config
          key: nginx_port
    - name: NGINX_SERVER_NAME
      valueFrom:
        configMapKeyRef:
          name: nginx-config
          key: nginx_host_name
    - name: MYSQL_ROOT_PASSWD
      valueFrom:
        secretKeyRef:
          name: mysql-root-passwd
          key: pwd
```

使用 `apply` 布署，接著查看有無引用至環境變數。

```shell
$ kubectl exec pod-secret-1 -- printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=pod-secret-1
NGINX_SERVER_PORT=8090
NGINX_SERVER_NAME=cch.lab.com
MYSQL_ROOT_PASSWD=itachi # this
KUBERNETES_PORT_443_TCP=tcp://10.127.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.127.0.1
KUBERNETES_SERVICE_HOST=10.127.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.127.0.1:443
NGINX_VERSION=1.18.0
NJS_VERSION=0.4.2
PKG_RELEASE=1~buster
HOME=/root
```


## 參考資源

- [](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#running-a-command-in-a-shell)
- [環境變數 - 關於 POD 資訊注入](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/)

