---
title: NAT
data: 2019-10-12
description: "私網要怎麼連上外面"
tags: [NAT, Network, GNS3]
draft: false
---
## 什麼是 NAT

因為 IPv4 的地址將用盡，如果沒有 `NAT` 技術的話將導致某一些人無法存取網路。在 `ISP` 提供的 IP 可分為，動態 IP 與靜態 IP。動態 IP 可以想成從 `ISP` 那邊自動獲取 IP 類似於 `DHCP`；靜態 IP 就是給一個 Global IP 位置且它是收費的。隨著物聯網裝置的需求想必 IPv4 是不夠用，因此就有 NAT 去提升效率，藉由一個 Global IP 來對其下的組織或客戶端實現 `NAT` 並存取網際網路。

`NAT` 通常會配置兩個網路，該 `NAT` 可以隱藏內部訊息，其流程大致上是把私有 IP 轉換為 Global IP，在將其封包轉發至目的地，這樣也就提供對內的安全性。

`NAT` 可以幫忙解決 IPv4 不足問題也可以有隱藏 IP 功能提高安全性。但它也會有一些缺點，像是要保留傳入與傳出的 IPv4 這將對記憶體或 CPU 帶來負擔，因為多了 `NAT` 轉換想必延遲一定會有，除此之外它變得不可追溯，因此在做除錯時會帶來麻煩。


## 環境設置

下圖為實驗環境，兩台客戶端和一台可通往 Cloud1 的 ISP。

<!-- ![](../assets/img/GNS3/NAT.png) -->
{{< figure src="/images/GNS3/NAT.png" width="auto" height="auto">}}


兩台客戶端使用 Vmware 虛擬機網路都為同一個 `host only`，接著用 `netplan` 配置虛擬機 `IP` 和 `gateway`，這 `gateway` 將會是 R1 的 `f0/0` 接口的 IP，這邊有 `netplan` 教學[鏈結](https://blog.gtwang.org/linux/ubuntu-linux-1804-configure-network-static-ip-address-tutorial/)。圖中的 Cloud1 使用 Vmware 提供的可上網的*網路適配器*，記住上圖的 IP 設置應當依照實驗環境而設置。

## Cloud1 的適配器配置

右鍵 -> Configure，接著會有下圖視窗。因為預設似乎是藍芽，這邊要勾選左下角的 `Show special Etherenet interface`，之後在藍芽網路連線的下拉式選單選擇要的適配器，在點選 `Add` 這樣 Cloud1 就會有配置的網路適配器。

![](https://i.imgur.com/RADCvwf.png)


## 配置 R1

如果客戶端都配置完成後，我們將對 R1 進行如下配置。

### 配置接口 f0/0
```
R1#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)#interface fa0/0
R1(config-if)#ip address 192.168.245.254 255.255.255.0
R1(config-if)#no shutdown
R1(config-if)#ip nat inside
```
### 配置接口 f1/0

f1/0 應當要和 Cloud1 同一網段，因此這邊使用 `DHCP` 方式對 Cloud1 進行 IP 請求。

```
R1#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)#interface fastEthernet 1/0
R1(config-if)#ip address dhcp 
R1(config-if)#ip nat outside
R1(config-if)#no shutdown
```

> inside 的配置通常是指內部組織，outside 則是連接網際網路

### 查看是否配置 IP 至接口上

```shell
#do sh ip int brief
Interface                  IP-Address      OK? Method Status                Protocol
FastEthernet0/0            192.168.245.254 YES manual up                    up
FastEthernet1/0            192.168.134.138 YES DHCP   up                    up
NVI0                       192.168.245.254 YES unset  up                    up # nat 接口
```

### 配置相關 NAT
使用 `access-list` 方式限制可存取的 `LAN`。再建立一個 NAT 的 IP `pool`，該 `pool` 是配置 Cloud1 分配的 IP 地址範圍。當 LAN 進行外網的訪問時將會透過 NAT 機制將 IP 替換 `pool` 中的 IP 以進行外網存取。

```shell
R1(config)#access-list 1 permit 192.168.245.0 0.0.0.255
R1(config)#ip nat pool DYNAMICNAT 192.168.134.100 192.168.134.120 netmask 255.255.255.0
R1(config)#ip nat inside source list 1 pool DYNAMICNAT
```

>動態 NAT 與靜態的差異是在於控制一個 pool 中多個 IP

## 實驗

R1 嘗試對 google DNS 進行 `ping`，結果如下是通的。同時間虛擬機也嘗試 ping `gateway` 也就是 R1 的 f0/0，如果是失敗的需要檢查虛擬機網路配置。

```shell
R1#ping 8.8.8.8

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
..!!!
Success rate is 60 percent (3/5), round-trip min/avg/max = 92/98/104 ms
```

虛擬機嘗試 `ping 8.8.8.8` 如果是成功的話表示說有藉由 NAT 方式進行轉址，並藉由 Cloud1 的網段出去至 8.8.8.8，結果應當如下圖

```shell
ping 8.8.8.8
```

![](https://i.imgur.com/W6AlAsE.png)

### NAT 資訊觀察

我們在虛擬機嘗試用 `ping` 對 8.8.8.8 做請求。透過 R1 的 NAT 資訊來驗證是否有做 NAT 動作，下面結果顯示說虛擬機的 IP 被轉為 `Inside global` IP。也就是虛擬機使用 `DYNAMICNAT` pool 中所定義的閒置 IP，以下面結果來說是將 `192.168.245.8` 轉成 `192.168.134.101`。

```shell
R1#show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
udp 192.168.134.101:59162 192.168.245.8:59162 8.8.8.8:44446  8.8.8.8:44446
udp 192.168.134.101:59162 192.168.245.8:59162 8.8.8.8:44447  8.8.8.8:44447
udp 192.168.134.101:59162 192.168.245.8:59162 8.8.8.8:44448  8.8.8.8:44448
udp 192.168.134.101:59162 192.168.245.8:59162 8.8.8.8:44449  8.8.8.8:44449
udp 192.168.134.101:59162 192.168.245.8:59162 8.8.8.8:44450  8.8.8.8:44450
udp 192.168.134.101:59162 192.168.245.8:59162 8.8.8.8:44451  8.8.8.8:44451
udp 192.168.134.101:59162 192.168.245.8:59162 8.8.8.8:44452  8.8.8.8:44452
udp 192.168.134.101:59162 192.168.245.8:59162 8.8.8.8:44453  8.8.8.8:44453
--- 192.168.134.101    192.168.245.8      ---                ---
--- 192.168.134.100    192.168.245.10     ---                ---
```

在做 NAT 時 IP 會有前後因此可用 `local` 或是 `global` 來定義。當企業內部組織的主機位於 `Inside`，連上網路的則為 `Outside`，其做轉換前會被記錄至 `Inside local`，被轉換後則是 `Inside global`。這邊可以想一下說在 R1 `Inside` 位置會先轉換在路由，而 `Outside` 則是相反。

下面是 NAT 統計訊息，`cisco` 文章說名此統計訊息會有延遲問題，這是隨著 NAT 轉址的數量而變化。

```shell
R1#show ip nat statistics
Total active translations: 31 (0 static, 31 dynamic; 29 extended)
Peak translations: 31, occurred 00:02:48 ago
Outside interfaces:
  FastEthernet1/0
Inside interfaces:
  FastEthernet0/0
Hits: 67  Misses: 0
CEF Translated packets: 66, CEF Punted packets: 1
Expired translations: 3
Dynamic mappings:
-- Inside Source
[Id: 1] access-list 1 pool DYNAMICNAT refcount 31
 pool DYNAMICNAT: netmask 255.255.255.0
        start 192.168.134.100 end 192.168.134.120
        type generic, total addresses 21, allocated 2 (9%), misses 0
Appl doors: 0
Normal doors: 0
Queued Packets: 0
```

## 參考資源

- [cisco nat](https://www.cisco.com/c/en/us/support/docs/ip/network-address-translation-nat/8605-13.html)
- [cisco nat book](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/configuration/xe-16/nat-xe-16-book/iadnat-addr-consv.html)