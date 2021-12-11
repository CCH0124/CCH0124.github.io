---
title: kubernetes - day20
date: 2020-09-10
description: "Service Discovery and DNS"
tags: [kubernetes]
draft: false
---

前面章節講過的 `Service`，提供了一個穩定可讓客戶端取得想要的服務接口，這之間 `POD` 如何知道某特定服務的 `IP` 和 `Port` 呢？這就需要服務發現(Service Discovery) 的幫助。我們也知道在 `K8s` 環境中，`POD` 不能依靠網路，當服務重新部署時，會導致 `IP` 不同，因此需要借助服務發現。服務發現簡單來說就是是弄清楚如何連接到服務的實際過程。


## 環境變數的服務發現

### Service 環境變數

只要創建 `Service` 都會使用以下的環境變數，當在同一 `namespace` 的 `POD` 會自動擁有這些資訊。可參考[官網 container-environment](https://kubernetes.io/docs/concepts/containers/container-environment/)

- {SVCNAME}_SERVICE_HOST
- {SVCNAME}_SERVICE_PORT

```shell
$ kubectl exec httpd-7765f5994-97hq5 -- printenv
PATH=/usr/local/apache2/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=httpd-7765f5994-97hq5
MYAPP_SVC_PORT_80_TCP_ADDR=10.8.2.52
MYAPP_SVC_SERVICE_HOST=10.8.2.52 # this
KUBERNETES_PORT_443_TCP_ADDR=10.8.0.1
MYAPP_SVC_PORT_80_TCP_PORT=80
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
MYAPP_SVC_PORT_80_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.8.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.8.0.1:443
MYAPP_SVC_SERVICE_PORT=80 # this
MYAPP_SVC_PORT=tcp://10.8.2.52:80
MYAPP_SVC_PORT_80_TCP=tcp://10.8.2.52:80
...
```

### Docker Link 方式環境變數

這邊我在 GCP 上的 `Compute Engine` 拿一個節點的容器用 `docker inspect` 觀察。

```shell
$ docker inspect 203e907bf5ba
"Env": [
                "MYAPP_SVC_PORT=tcp://10.8.2.52:80",
                "MYAPP_SVC_PORT_80_TCP_PROTO=tcp",
                "KUBERNETES_SERVICE_PORT=443",
                "KUBERNETES_SERVICE_PORT_HTTPS=443",
                "KUBERNETES_PORT=tcp://10.8.0.1:443",
                "KUBERNETES_PORT_443_TCP=tcp://10.8.0.1:443",
                "KUBERNETES_SERVICE_HOST=10.8.0.1",
                "KUBERNETES_PORT_443_TCP_PROTO=tcp",
                "MYAPP_SVC_SERVICE_PORT=80",
                "MYAPP_SVC_PORT_80_TCP=tcp://10.8.2.52:80",
                "MYAPP_SVC_PORT_80_TCP_PORT=80",
                "KUBERNETES_PORT_443_TCP_PORT=443",
                "KUBERNETES_PORT_443_TCP_ADDR=10.8.0.1",
                "MYAPP_SVC_SERVICE_HOST=10.8.2.52",
                "MYAPP_SVC_PORT_80_TCP_ADDR=10.8.2.52",
                "PATH=/usr/local/apache2/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "HTTPD_PREFIX=/usr/local/apache2",
                "HTTPD_VERSION=2.4.46",
                "HTTPD_SHA256=740eddf6e1c641992b22359cabc66e6325868c3c5e2e3f98faf349b61ecf41ea",
                "HTTPD_PATCHES="
            ],
```
但是，在不同 `namespace`、或是 `POD` 與 `Service` 部署順序不同時，環境變數方式將是一個缺點，然而 `DNS` 可以解決這個問題。

## DNS 服務發現

`Kubernetes` 上在安裝環境時，有一個 `CoreDNS`，是一個非常重要的元件之一，它會負責將 `Service` 資源等轉成資源紀錄(Resource Record)。在預設下 `POD` 的資源紀錄也會被自動的配置，當中資源紀錄會包含 `namespace` 等資訊。下面是從官方[Kubernetes DNS 規格定義](https://github.com/kubernetes/dns/blob/master/docs/specification.md)，所節錄。

- `Service` 類型為 `ClusterIP` 的資源紀錄
    - A Record
        - `<service>.<ns>.svc.<zone>. <ttl> IN A <cluster-ip>`
    - SRV Record
        - `_<port>._<proto>.<service>.<ns>.svc.<zone>. <ttl> IN SRV <weight> <priority> <port-number> <service>.<ns>.svc.<zone>`
    - PTR Record
        - `<d>.<c>.<b>.<a>.in-addr.arpa. <ttl> IN PTR <service>.<ns>.svc.<zone>`
- `Service` 類型為 `Headless` 的資源紀錄
    - A Record
        - `<service>.<ns>.svc.<zone>. <ttl> IN A <endpoint-ip>`
    - SRV Record
        - `_<port>._<proto>.<service>.<ns>.svc.<zone>. <ttl> IN SRV <weight> <priority> <port-number> <hostname>.<service>.<ns>.svc.<zone>`
    - PTR Record
        - `<d>.<c>.<b>.<a>.in-addr.arpa. <ttl> IN PTR <hostname>.<service>.<ns>.svc.<zone>`
- `Service` 類型為 `ExternalName` 的資源紀錄
    - CNAME Record
        - `<service>.<ns>.svc.<zone>. <ttl> IN CNAME <extname>`

> SRV 相較於其它資源類型它記錄了 Port

只要我們創建 `Service` 資源，`DNS` 會做服務解析和服務註冊的動作，這樣可讓 `POD` 使用 `DNS` 方式去請求 `Service`。每個 `Service` 會包含以下兩個資源紀錄。在安裝 `K8s` 環境時可設定 `DNS` 相關資訊，否則默認把 `cluster.local.` 和主機所在的域名做為本地用。

- {SVCNAME}.{NAMESPACE}.{CLUSTER_DOMAIN}
- {SVCNAME}.{NAMESPACE}.svc.{CLUSTER_DOMAIN}

這邊觀察 GKE 上部署的一個 `POD` 它的 `DNS` 相關設置。也就是說服務的查詢都會導向該 `CoreDNS` 的 `Service` 上。

```shell
$ kubectl exec hello-kubernetes-5b89dbbc4b-t7bvr -- cat /etc/resolv.conf
nameserver 10.8.0.10 
search default.svc.cluster.local svc.cluster.local cluster.local us-central1-c.c.sunny-catwalk-286908.internal c.sunny-catwalk-286908.internal google.internal
options ndots:5
```
```shell
$ kubectl get svc -n kube-system # 系統上的 DNS 系統 
NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
...
kube-dns               ClusterIP   10.8.0.10     <none>        53/UDP,53/TCP   18d
...
```

- nameserver
    - 為 `kube-dns` 的 `Service` 資源
- default.svc.cluster.local
    - {NAMESPACE}.svc.{CLUSTER_DOMAIN}
- svc.cluster.local
    - svc.{CLUSTER_DOMAIN}

我們進到一個可交互的 `POD` 並使用 `nslookup` 查詢 `myapp-svc` 的域名 `myapp-svc.default.svc.cluster.local`，發現解析出來的就是該 `Service` 的 `IP`。

```shell
$ kubectl exec -it hello-kubernetes-5b89dbbc4b-t7bvr /bin/sh
~ $ nslookup myapp-svc.default.svc.cluster.local
Server:         10.8.0.10
Address:        10.8.0.10:53

Name:   myapp-svc.default.svc.cluster.local
Address: 10.8.2.52
$ kubectl get svc myapp-svc
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
myapp-svc   ClusterIP   10.8.2.52    <none>        80/TCP    2d1h
```

這是基於 `SRV` 做查詢格式為 `_PORT-NAME._PROTOCOL.SERVICE-NAME.NAMESPACE.svc.cluster.local`

```shell
$ nslookup _http._tcp.myapp-svc.default.svc.cluster.local
Server:         10.8.0.10
Address:        10.8.0.10:53

_http._tcp.myapp-svc.default.svc.cluster.local  canonical name = myapp-svc.default.svc.cluster.local
Name:   myapp-svc.default.svc.cluster.local
Address: 10.8.2.52
```

對於 `POD` 解析格式為 `POD-IP-ADDRESS.NAMESPACE.pod.cluster.loscal`，這邊是看[官網](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#a-records-1)一些用法都在裡面
```shell
$ nslookup 10-4-2-53.default.pod.cluster.local
Server:         10.8.0.10
Address:        10.8.0.10:53


Name:   10-4-2-53.default.pod.cluster.local
Address: 10.4.2.53
```

## POD DNS

因為 `POD` 對於 `DNS` 可能有不同的需求，因此也提供設定方式，這邊可直接參考[官方](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy)。

- Default
    - 不需設定，同步於節點的 `nameserver`
- ClusterFirst
    - 預設值
    - 使用 cluster DNS 做為設定，資訊如下

```shell
$ kubectl exec hello-kubernetes-5b89dbbc4b-t7bvr -- cat /etc/resolv.conf
nameserver 10.8.0.10 
search default.svc.cluster.local svc.cluster.local cluster.local us-central1-c.c.sunny-catwalk-286908.internal c.sunny-catwalk-286908.internal google.internal
options ndots:5
```

- ClusterFirstWithHostNet
    - `hostNetwork` 為與主機的網路 `namespace` 共享時，又想使用 cluster DNS 做為設定
- None
    - 不設定 `DNS` 相關資訊
    - 需使用 `dnsConfig` 字段來設定 `DNS` 相關資訊

## 參考資源

- [platform9 - kubernetes-service-discovery-principles-in-practice](https://platform9.com/blog/kubernetes-service-discovery-principles-in-practice/)
- [GKE - service-discovery](https://cloud.google.com/kubernetes-engine/docs/concepts/service-discovery?hl=zh-tw)
- [jimmysong - service discovery and loadbalancing](https://jimmysong.io/kubernetes-handbook/practice/service-discovery-and-loadbalancing.html)
- [SRV](https://www.cloudflare.com/learning/dns/dns-records/dns-srv-record/)