---
title: kubernetes 網路溝通基礎
date: 2021-12-12
description: "kubernetes 網路溝通基礎"
tags: [kubernetes]
draft: false
---

以 Kubernetes 角度來看，網路大致有四種通訊類型

1. Container to Container
Pod 內的容器共享同一網路，通常由 `pause`  基礎架構容器提供。彼此間可透過 lo 接口交互。

![](https://miro.medium.com/max/1400/1*oyGbXt7kStLd85ZT4it3oQ.png )from 此篇[文章](https://medium.com/google-cloud/understanding-kubernetes-networking-pods-7117dd28727)

2. Pod  to Pod 
每個 Pod 擁有一個在集群中全域唯一的 IP 地址，可直接與其它 Pod 通訊。如上圖所示，其節點也可以透過同一網路的橋接設備(docker0)與 Pod 進行通訊。

3. Service to Pod 
Service 資源也稱 *Cluster network*，啟動 `kube-apiserver` 時由 `--service-cluster-ip-range` 進行指定。當我們對 Service 資源進行變動時，會藉由 Api Server 觸發節點上的 `kube-proxy`，並設置相對應的 `iptables` 或 `ipvs` 規則，方便實現 Cluster-IP 與 Pod-IP 之間進行轉發。

可參考此篇[文章](https://medium.com/google-cloud/understanding-kubernetes-networking-services-f0cb48e4cc82)

4. WAN to Pod 
這須透過 nodePort、LoadBalancer 類型的 Service 資源實現。

## Pod 網路
在配置時，`kubelet` 會在預設的 `/etc/cni/net.d/` 目錄中搜尋 CNI 的 json 配置檔，並藉由 type 查找 `/opt/cni/bin` 下相關插件


```bash
cat 05-cilium.conf
{
  "cniVersion": "0.3.1",
  "name": "cilium",
  "type": "cilium-cni",
  "enable-debug": false
}
```

```bash
master in /opt/cni/bin
○ → ls
bandwidth  bridge  cilium-cni  dhcp  firewall  flannel  host-device  host-local  ipvlan  loopback  macvlan  portmap  ptp  sbr  static  tuning  vlan
```

CNI 僅是一個規範，其提供類型分為三類
- main
    - 實作特定網路功能，`loopback`、`bridge`、`ipvlan` 等
- meta
    - 本身不提供網路的實現，而是調用其他套件
- ipam
    - 僅分配 IP 地址。不提供其他網路實現  