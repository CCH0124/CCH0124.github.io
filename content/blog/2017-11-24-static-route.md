---
title: Static routes
date: 2017-11-24
description: "轉發封包方式之一"
tags: [Network, GNS3]
draft: false
---

## Routing table
路由器是一個負責將封包送往目的地的設備，而路由器跟路由器之間必須要分享所學習到的資訊，並且把交換的資訊以到鄰居的成本以和使用什麼路由協定等給記錄至一張表格，而這張表格就是 `Routing table`。

此篇文章範圍是 `Static Route` 的概念。 

## The router decides how to send the packet
- Static Route
    - 手動輸入
        - 網路有變化，需要人員的管理
    - 無須學習
        - 因此速度快
- Dynamic Route
    - 做 `Routing Protocol` 的設定
        - 耗資源
    - 由 router 之間去做協調
    - 網路有變化，會透過 `Routing Protocol` 學習

>Default route （預設路由）所謂的預設路由就是當不知道要將這個封包送往哪裡的時候，就會採用這個預設路由所指定的路徑。
## Example

![](https://i.imgur.com/sIzooXT.png)

這邊不描述 IP address 的設定。

目標：
- 設置 R1 `Static Route`。
- R2 和 R3 都各有一條 `Default route`。

### Set R2 & R3 Default route
##### R2
將來自不存在於路由表的目的地給丟置 10.0.0.1 的 R1 處裡。如：`PC 5` 想與 `PC 8` 溝通，則 `PC 5` 送往 `192.168.2.10` 的目的地不再 `R2` 的路由表因此會將此封包先丟往 `gateway` 再交由 `R1` 做處裡。再者，`R2` 想到 `R3` 必須經由 `R1` 因此要從 `R1` 做 `Static Route` 設定。
```shell
R2(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.1
```
查看 R2 路由表，觀察靜態路由的宣告。
```shell
R2#show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       + - replicated route, % - next hop override

Gateway of last resort is 10.0.0.1 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 10.0.0.1
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.0.0/30 is directly connected, Serial6/0
L        10.0.0.2/32 is directly connected, Serial6/0
      172.24.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        172.24.0.0/16 is directly connected, Ethernet5/0
L        172.24.0.1/32 is directly connected, Ethernet5/0
      172.25.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        172.25.0.0/16 is directly connected, Ethernet5/1
L        172.25.0.1/32 is directly connected, Ethernet5/1
      172.26.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        172.26.0.0/16 is directly connected, Ethernet5/2
L        172.26.0.1/32 is directly connected, Ethernet5/2
      172.27.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        172.27.0.0/16 is directly connected, Ethernet5/3
L        172.27.0.1/32 is directly connected, Ethernet5/3

```
`S` 表示 `static route` 設定，`AD` 值為 1，比其它 `Dynamic Route` 優先權來得高。

##### R3
將來自不存在於路由表的目的地給丟置 10.0.0.5 的 R1 處裡。
```shell
R3(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.5
```
### Set R1 Static Route
靜態路由指令的關鍵字是 

`ip route`
`ip route [目的地的網路 IP 位址][目的地子網路的網路遮罩][到達目的地而要經過的下一台路由器設備的 IP 位址]`
##### Next Hop IP
當上述的封包抵達 R1 做轉發時，會看著路由表是否有到達目的地的條目，如果有則會進行轉發。`PC 5` 傳到 `PC 8` 封包到達 `R1` 時發現有可以達到 `PC 8` 網段的路徑（即第二條宣告）。
R1 `Static Route` 宣告
```shell
R1(config)# ip route 172.0.0.0 255.224.0.0 10.0.0.2
R1(config)# ip route 192.168.0.0 255.255.252.0 10.0.0.6
```
R1 路由表
```shell
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

      10.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
C        10.0.0.0/30 is directly connected, Serial6/0
L        10.0.0.1/32 is directly connected, Serial6/0
C        10.0.0.4/30 is directly connected, Serial6/1
L        10.0.0.5/32 is directly connected, Serial6/1
S     172.0.0.0/11 [1/0] via 10.0.0.2
S     192.168.0.0/22 [1/0] via 10.0.0.6
      192.168.64.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.64.0/24 is directly connected, Ethernet5/0
L        192.168.64.1/32 is directly connected, Ethernet5/0
      192.168.65.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.65.0/24 is directly connected, Ethernet5/1
L        192.168.65.1/32 is directly connected, Ethernet5/1
```
##### Next Hop Interface
第二種宣告方式

`ip route [目的地的網路 IP 位址][目的地子網路的網路遮罩][到達目的地而要經過的下一台路由器設備的介面][到達目的地而要經過的下一台路由器設備的 IP]`
```shell
R1(config)# ip route 172.0.0.0 255.224.0.0 Serial6/0 10.0.0.2
R1(config)# ip route 192.168.0.0 255.255.252.0 Serial6/1 10.0.0.6
```

### PC 觀察

##### PC5 to PC8
```shell
PC-5> trace 192.168.2.10
trace to 192.168.2.10, 8 hops max, press Ctrl+C to stop
 1   172.26.0.1   697.380 ms  19.832 ms  9.425 ms
 2   10.0.0.1   226.608 ms  30.747 ms  41.659 ms
 3   10.0.0.6   165.173 ms  40.168 ms  61.269 ms
 4   *192.168.2.10   152.268 ms (ICMP type:3, code:3, Destination port unreachable)
```
`PC5` 從 `R2` `gateway` 出去（透過 `Default route`）到達 `R1` 再經由 `R1` `Static Route` 抵達 `R3`
##### PC9 to PC1

```shell
PC-9> ping 192.168.64.10
84 bytes from 192.168.64.10 icmp_seq=1 ttl=62 time=32.737 ms
84 bytes from 192.168.64.10 icmp_seq=2 ttl=62 time=39.684 ms
84 bytes from 192.168.64.10 icmp_seq=3 ttl=62 time=39.186 ms
```

## conclusion
`Static Route` 比較適合在 `stub Network`（透過單一路由路徑連接的網路）下設置，當然也適合用在預設閘道上，因為這些地方不易變化。
