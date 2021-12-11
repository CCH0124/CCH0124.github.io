---
title: kubernetes - day22
date: 2020-09-12
description: "儲存與持久化儲存 part02 - NFS"
tags: [kubernetes]
draft: false
---

接續上一章的實作，這邊將使用 NFS(Network File System) 分散式儲存系統實現永久儲存，它可以讓客戶端像是在本地端取得資料。


## NFS 儲存卷

這邊將不使用 `GKE` 操作而是使用本地端架設的 `K8s` 來操作，將會有四台虛擬機，一台 `NFS`，其它則為 `K8s` 叢集。我們會在一台虛擬機上安裝 `NFS`，並設定要導出的儲存空間，之後再藉由設定檔方式去讓 `POD` 取得 `NFS` 的掛載目錄。在 `POD` 生命週期中，此方式只是會卸載掛載的資訊，並非像 `emptyDir` 一樣是直接刪除。其定義字段如下

- server: NFS Server 的 `IP` 或是能解析的域名
- path: NFS Server 定義的共享檔案路徑
- readOnly: 是否要唯讀，預設是 `false`

先在 `NFS` 的虛擬機上安裝 `NFS` 環境，在 NFS Server 主機安裝 `nfs-kernel-server`，客戶端安裝 `nfs-common`也就是 `K8s` 叢集的節點，安裝完之後在 NFS Server 設定以下
```shell
sudo apt install  -y
sudo mkdir /k8sData
echo "/home/cch/k8sData *(rw,sync,no_root_squash)" | sudo tee /etc/exports # , 不能有空格隔開
sudo exportfs -r # reload
sudo showmount -e # 顯示 NFS 設定的要掛載目錄
```

`K8s` 的叢集節點進行 `NFS` 的掛載

```shell
$ mount NFS_SERVER_IP:/home/cch/k8sData /mnt
$ mount # 觀察有無掛載
```

這是 `yaml` 的測試範例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nfs-demo
  namespace: default
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html/
  volumes:
  - name: html
    nfs:
      path: /home/cch/k8sData
      server: 192.168.134.130
```

接著在 NFS Server 上新增以下，上述完成 `K8s` 叢集上節點的 `/mnt` 下要有檔案
```shell
$ vi index.html # 要在 NFS Server 要掛載的目錄下新增
<h1>NFS Server $(hostname)</h1>
```

將剛定義的 `yaml` 使用 `apply` 或其它方式進行布署，接著觀察布署後的 `POD`，並用 `curl` 驗證是否有讀取 NFS Server 的資料。
```shell
$ kubectl get pods -o wide
NAME           READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
pod-nfs-demo   1/1     Running   0          2m14s   10.244.2.32   node02   <none>           <none>
cch@master:~$ curl 10.244.2.32
<h1>NFS Server $(hostname)</h1>
cch@master:~$ curl 10.244.2.32
<h1>NFS Server $(hostname)</h1>
```

這邊在使用 `ReplicaSet` 實現多副本，用來實現不同節點上能夠有同步的資訊，也實現說不會因為被重新調度而讀取不到資料。

```yaml
kind: ReplicaSet
metadata:
  name: rs-nfs-demo
  namespace: default
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp
      release: canary
  template:
    metadata:
      name: myapp
      namespace: default
      labels:
        app: myapp
        release: canary
    spec:
      containers:
      - name: myapp
        image: ikubernetes/myapp:v1
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html/
      volumes:
        - name: html
          nfs:
            path: /home/cch/k8sData
            server: 192.168.134.130
```

同樣取得 `POD` 資訊，並用 `curl` 驗證
```shell
$ kubectl get pods -o wide -l app=myapp,release=canary
NAME                READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
rs-nfs-demo-479qk   1/1     Running   0          95s   10.244.1.28   node01   <none>           <none>
rs-nfs-demo-8mk2z   1/1     Running   0          95s   10.244.2.35   node02   <none>           <none>
rs-nfs-demo-99z8w   1/1     Running   0          95s   10.244.1.26   node01   <none>           <none>
rs-nfs-demo-bkmbb   1/1     Running   0          95s   10.244.2.36   node02   <none>           <none>
rs-nfs-demo-hz4l9   1/1     Running   0          95s   10.244.1.27   node01   <none>           <none>
cch@master:~$ curl 10.244.1.28
<h1>NFS Server $(hostname)</h1>
cch@master:~$ curl 10.244.2.35
<h1>NFS Server $(hostname)</h1>
cch@master:~$ curl 10.244.1.26
<h1>NFS Server $(hostname)</h1>
cch@master:~$ curl 10.244.2.36
<h1>NFS Server $(hostname)</h1>
```

透過上面的驗證可以說明，`NFS` 可以為 `POD` 達到持久性儲存，`POD` 被重新調度時也能夠有完整的資訊。要將其儲存刪除則是要以 `NFS` 管理員身分去做刪除，並非是 `POD`。


