---
title: kubernetes network policy
date: 2021-12-12
description: "kubernetes network policy"
tags: [kubernetes]
draft: false
---

`Network policy` 用於控制分組的 Pod 資源彼此之間如何進行通訊，獲分組的 Pod 資源如何與其它網路端點進行通訊的規範。它是一種更為精細的流量控制，實現租戶隔離機制。

Kubernetes 中，封包流入和流出的地點是 Pod 資源，所以 `NetworkPolicy` 資源也使用標籤選擇器選出一組 Pod 資源作為控管對象。內容會定義入的流量 *ingress 規則*，或是出的流量 *Egress 規則*。是否為部分或是全部生效取決於 `policyTypes`。

![](https://img.uj5u.com/2021/01/05/212012050607101.png)

在還沒進行 `Network policy` 資源綁定時，Pod 是可接受四面八方的流量，並做出回應。而當綁定 `Network policy` 資源後，可以將某些流量進行隔離。

- podSelector
    - 透過策略來選擇特定一群的 Pod，可藉由 `macthLabel` 或是 `matchExpression` 獲取
- Egress
    - 出站流量，通常由流量的網路端點(to)和 port 進行定義
- Ingress
    - 入站流量，通常由流量的網路端點(from)和流量目標 port 所定義
- 端點 to 或 from
    - 其可以是 CIDR 格式的 IP 地址(ipBlock)`、namespaceSelector` 或是 `podSelector`

在 `Egress` 或 `Ingress` 都可使用網路端點，其通常是某 namespace 中的一或一組 Pod 資源，會經由  `namespaceSelector`  選定特定 `namespace` 後，經由 `ipBlock` 或 `podSelector` 進行指定。在未定義 `Egress` 或 `Ingress` 默認是非隔離狀態。一旦在 `networkpolicy.spec` 中給出 `Ingress` 或 `Egress`，則它們的 `from` 或 `to` 字段的值就成了白名單列表，而空值意味著所有端點，即不限制訪問。

>NetworkPolicy 資源屬於 `namespace` 級別。

## Ingress 入站
`Ingress` 字段，主要有以下
- `from`
    - 白名單
- `to`
    - 白名單

當 from 和 to 同時定義表示 `and` 的邏輯關係，它將會放行滿足 from 和 to 的流量。

`from` 可以使用以下，當定義了兩個以上，則會有 `or` 邏輯關係。
- ipBlock
    - 根據 IP 地址選擇流量來源端點
- namespaceSelector
    - 挑選 namespace
    - 空值表示所有 namespace
- podSelector
    - 於 `NetworkPolicy` 所在的 namespace 內基於標籤選擇器挑選的 Pod 資源
    - 空值表示所有 Pod 資源，在當前 namespace 下
- ports 透過以下兩個定義流量目標端口
    - port
    - protocol

## Egress 出站
`Egress` 字段，主要有以下
- to
    - 當有給定限制時，只有目標地址匹配列表中主機地址的出站流量被放行
- ports
    - 目標端口列表

期內嵌的字段與 Ingress 是相同，區別是在於流量方向。