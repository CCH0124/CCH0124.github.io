---
title: "IP Routing"
date: 2016-04-12
tags: ["Network"]
categories: ["Network"]
description: "IP 路由介紹"
draft: false
---
## IP Routing
簡易來說就是我們從台灣主機到美國 google 主機，所經過的路徑過程。過程之間都是透過路由器的決策來完成。路由器會得知目的地 IP，藉由路由表決定下一跳的路徑，接著轉發封包。

![](https://i.imgur.com/OiSnvjA.png)

假設 TW 想要取得 Google 的資訊，但是 Google 在跟 TW 不同的網路上。TW 會發送封包至路由器，路由起接收到封包後，會得知目的地 IP 位址，並對照路由表做決策再將 TW 發送至路由器的封包轉發到跟目標網路關聯的介面，最後抵達 Google。

## Gateway
本地端主機要和遠方的主機通訊。當主機沒有遠方的主機路由資訊時，會發送至 Gateway。該 Gateway 的路由器會再做決策。

如上面的圖

TW 設置有路由器其中一個介面的 IP，TW 嘗試與遠方不同網路的 google 通訊。TW 在路由表中查找目標網路是否有路徑可到達。如果未找到該路徑，則 TW 將封包發送至路由器上的 TW Gateway IP。路由器接收風包後並將封包轉發給 google。

## Routing Table
路由器會維護一個路由表並儲存在 RAM 中。路由器使用路由表來確定到目標網路的路徑。
每個路由表包含以下資訊：

```shell=
R1#show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       + - replicated route, % - next hop override

Gateway of last resort is not set

      192.168.1.0/24 is variably subnetted, 8 subnets, 5 masks
R        192.168.1.0/25 [120/1] via 192.168.1.226, 00:00:14, Serial6/0
C        192.168.1.128/26 is directly connected, Ethernet5/0
L        192.168.1.129/32 is directly connected, Ethernet5/0
R        192.168.1.192/27 [120/1] via 192.168.1.230, 00:00:19, Serial6/1
C        192.168.1.224/30 is directly connected, Serial6/0
L        192.168.1.225/32 is directly connected, Serial6/0
C        192.168.1.228/30 is directly connected, Serial6/1
L        192.168.1.229/32 is directly connected, Serial6/1

```

- Codes
    - 路由器的來源
    - R 表示 RIP 路由協定
    - C 表示直連網路
- Network
    - 192.168.1.0/25
    - 不同網路的網路位址和遮罩，當封包進來時為了要將封包送達至目的地，會查看目的地 IP 落在哪一條紀錄，找到後就使用該紀錄去轉發封包。
- AD/Metric
    - [120/1]
    - AD 是 Administrative Distance，這是一種權值的表現，越低表示權重越高。
    - Codes 是 C 權重最高其次是靜態路由。
- Next Hop
    - via 192.168.1.226
    - 要轉發至下一跳的位址，不一定是 IP 可以是介面

寫入路由表方式有
- directly connected subnets
- static routing
- dynamic routing