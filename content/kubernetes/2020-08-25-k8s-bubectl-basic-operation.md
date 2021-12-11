---
title: kubernetes - day04
date: 2020-08-25
description: "kubectl 基本操作"
tags: [kubernetes]
draft: false
---

操作環境有 GKE 與 kubeadm 自架群集操作，本篇文章將會帶讀者學會使用 `kubectl` 觀看集群相關的資訊。

###  查看預設安裝的 POD 資源

GKE 的環境

```shell
$ kubectl get pods -n kube-system # -n 表示 namespace，kube-system 為指定的 namespace
NAME                                                        READY   STATUS    RESTARTS   AGE
event-exporter-v0.3.0-5cd6ccb7f7-mp7p4                      2/2     Running   0          13h
fluentd-gcp-scaler-6855f55bcc-mchvv                         1/1     Running   0          13h
fluentd-gcp-v3.1.1-f8wc8                                    2/2     Running   0          13h
fluentd-gcp-v3.1.1-g6mbn                                    2/2     Running   0          13h
fluentd-gcp-v3.1.1-zq4xm                                    2/2     Running   0          13h
heapster-gke-7c7bdf567c-cmqhm                               3/3     Running   0          13h
kube-dns-5c446b66bd-5ltbw                                   4/4     Running   0          13h
kube-dns-5c446b66bd-fqvwk                                   4/4     Running   0          13h
kube-dns-autoscaler-6b7f784798-hr8ck                        1/1     Running   0          13h
kube-proxy-gke-cluster-1-test-default-pool-255d7fb2-1f8l    1/1     Running   0          13h
kube-proxy-gke-cluster-1-test-default-pool-255d7fb2-lbwc    1/1     Running   0          13h
kube-proxy-gke-cluster-1-test-default-pool-255d7fb2-ppnm    1/1     Running   0          13h
l7-default-backend-84c9fcfbb-kwrrs                          1/1     Running   0          13h
metrics-server-v0.3.3-fdc67d4b6-264zw                       2/2     Running   0          13h
prometheus-to-sd-ktwlz                                      2/2     Running   0          13h
prometheus-to-sd-pfxlh                                      2/2     Running   0          13h
prometheus-to-sd-tvgv7                                      2/2     Running   0          13h
stackdriver-metadata-agent-cluster-level-646c549689-cfftk   2/2     Running   0          13h
```

```shell
$ kubectl get namespace 
NAME              STATUS   AGE
default           Active   13h
kube-node-lease   Active   13h
kube-public       Active   13h
kube-system       Active   13h
```

![](https://i.imgur.com/VphvAO2.png) 雲端介面示意圖



kubeadm 預設 namespace 查看。

```shell=
$ kubectl get namespace
NAME              STATUS   AGE
default           Active   4d17h
kube-node-lease   Active   4d17h
kube-public       Active   4d17h
kube-system       Active   4d17h
```


###  Kubernetes 版本查看

GKE

```shell=
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.6", GitCommit:"dff82dc0de47299ab66c83c626e08b245ab19037", GitTreeState:"clean", BuildDate:"2020-07-15T16:58:53Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"15+", GitVersion:"v1.15.12-gke.2", GitCommit:"fb7add51f767aae42655d39972210dc1c5dbd4b3", GitTreeState:"clean", BuildDate:"2020-06-01T22:20:10Z", GoVersion:"go1.12.17b4", Compiler:"gc", Platform:"linux/amd64"}
```
kubeadm

```shell=
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.8", GitCommit:"9f2892aab98fe339f3bd70e3c470144299398ace", GitTreeState:"clean", BuildDate:"2020-08-13T16:12:48Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.8", GitCommit:"9f2892aab98fe339f3bd70e3c470144299398ace", GitTreeState:"clean", BuildDate:"2020-08-13T16:04:18Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"linux/amd64"}
```

###  Kubernetes API 資源

GKE 和 kubeadm 原則上會是相同，當中 `SHORTNAMES` 表示縮寫，可在下指令時用其縮寫表示資源。

```shell=
$ kubectl api-resources
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
bindings                                                                      true         Binding
componentstatuses                 cs                                          false        ComponentStatus
configmaps                        cm                                          true         ConfigMap
endpoints                         ep                                          true         Endpoints
...
```

###  Kubernetes Api 版本



```shell=
$ kubectl api-versions
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
...
```

![](https://zhiweiyin318.github.io/k8s-notes/images/kubernetes-api.png)


一個 `API` 對像在 `Etcd` 裡面完整的資源路徑由：`Group`（API 組），`verison`（API 版本）和 `Resource`（API 資源類型）三個部分組成。

- [官網 API 相關資訊資訊](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)


###  Kubernetes Cluster 資訊

GKE 上
```shell=
$ kubectl cluster-info
Kubernetes master is running at https://35.238.20.43
GLBCDefaultBackend is running at https://35.238.20.43/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
Heapster is running at https://35.238.20.43/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://35.238.20.43/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://35.238.20.43/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

kubeadm 上

```shell=
$ kubectl cluster-info
Kubernetes master is running at https://192.168.134.131:6443
KubeDNS is running at https://192.168.134.131:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```


