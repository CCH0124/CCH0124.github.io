---
title: Network Troubleshotting
date: 2018-01-28
description: "網路不通，DNS 也連不上該怎麼辦...."
tags: [Network, ping, dns, ss, arp, tcpdump]
draft: false
---

通常如果實體設備能正常工作，接下來會藉由一些網路相關的指令或工具進行網路上的除錯。像是確認 A 能否抵達 B 等。下面描述的是一些除錯的指令與工具。

## ping 

[先前有提到過](https://cch0124.github.io/ping/)，這邊就不提了

## DSN Troubleshotting

### dig
dig 驗證 DNS 主機地址、MX 記錄和所有其他 DNS 記錄，方便了解 DNS 部署情況。

```shell
$ dig www.nkust.edu.tw

; <<>> DiG 9.10.3-P4-Ubuntu <<>> www.nkust.edu.tw
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 462
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; MBZ: 0005 , udp: 1280
;; QUESTION SECTION:
;www.nkust.edu.tw.              IN      A

;; ANSWER SECTION:
www.nkust.edu.tw.       5       IN      A       140.133.78.31

;; Query time: 220 msec
;; SERVER: 192.168.137.2#53(192.168.137.2)
;; WHEN: Sat Sep 07 09:52:09 CST 2019
;; MSG SIZE  rcvd: 61
```

針對 MX 紀錄

```shell
$ dig MX www.nkust.edu.tw

; <<>> DiG 9.10.3-P4-Ubuntu <<>> MX www.nkust.edu.tw
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41967
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; MBZ: 0005 , udp: 1280
;; QUESTION SECTION:
;www.nkust.edu.tw.              IN      MX

;; AUTHORITY SECTION:
nkust.edu.tw.           5       IN      SOA     ns1.nkust.edu.tw. hostmaster. 876 900 600 86400 3600

;; Query time: 49 msec
;; SERVER: 192.168.137.2#53(192.168.137.2)
;; WHEN: Sat Sep 07 09:52:34 CST 2019
;; MSG SIZE  rcvd: 95
```

所有紀錄

```shell
$ dig ANY nkust.edu.tw

; <<>> DiG 9.10.3-P4-Ubuntu <<>> ANY nkust.edu.tw
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 34785
;; flags: qr rd ra; QUERY: 1, ANSWER: 9, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; MBZ: 0005 , udp: 1280
;; QUESTION SECTION:
;nkust.edu.tw.                  IN      ANY

;; ANSWER SECTION:
nkust.edu.tw.           5       IN      NS      ns4.nkust.edu.tw.
nkust.edu.tw.           5       IN      NS      ns3.nkust.edu.tw.
nkust.edu.tw.           5       IN      NS      ns2.nkust.edu.tw.
nkust.edu.tw.           5       IN      NS      ns1.nkust.edu.tw.
nkust.edu.tw.           5       IN      MX      10 ALT3.ASPMX.L.GOOGLE.COM.
nkust.edu.tw.           5       IN      MX      1 ASPMX.L.GOOGLE.COM.
nkust.edu.tw.           5       IN      MX      5 ALT1.ASPMX.L.GOOGLE.COM.
nkust.edu.tw.           5       IN      MX      5 ALT2.ASPMX.L.GOOGLE.COM.
nkust.edu.tw.           5       IN      MX      10 ALT4.ASPMX.L.GOOGLE.COM.

;; Query time: 46 msec
;; SERVER: 192.168.137.2#53(192.168.137.2)
;; WHEN: Sat Sep 07 09:53:21 CST 2019
;; MSG SIZE  rcvd: 231
```

反向搜尋獲取 DNS 資訊

```shell
$ dig -x 8.8.8.8

; <<>> DiG 9.10.3-P4-Ubuntu <<>> -x 8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 44451
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; MBZ: 0005 , udp: 1280
;; QUESTION SECTION:
;8.8.8.8.in-addr.arpa.          IN      PTR

;; ANSWER SECTION:
8.8.8.8.in-addr.arpa.   5       IN      PTR     dns.google.

;; Query time: 31 msec
;; SERVER: 192.168.137.2#53(192.168.137.2)
;; WHEN: Sat Sep 07 09:54:44 CST 2019
;; MSG SIZE  rcvd: 73
```

dig 利用 `/etc/resolv.conf` 中列出的 server 進行查詢。

### host

很像 dig 的指令，可以用來針對 DNS 偵錯。

```shell
$ host -a nkust.edu.tw
Trying "nkust.edu.tw"
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12012
;; flags: qr rd ra; QUERY: 1, ANSWER: 12, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;nkust.edu.tw.                  IN      ANY

;; ANSWER SECTION:
nkust.edu.tw.           5       IN      TXT     "v=spf1 ip4:140.127.113.0/24 ip4:140.133.78.0/24 ip4:163.18.1.10 ip4:120.119.140.0/24 ipv4:61.216.112.118 ip4:163.18.2.113 include:_spf.google.com ~all"
nkust.edu.tw.           5       IN      TXT     "google-site-verification=OhgxCcOmuGjTkMf_11whQSN4SYxrcEVJkbm6WFLDMZY"
nkust.edu.tw.           5       IN      MX      10 alt4.aspmx.l.google.com.
nkust.edu.tw.           5       IN      MX      1 aspmx.l.google.com.
nkust.edu.tw.           5       IN      MX      5 alt1.aspmx.l.google.com.
nkust.edu.tw.           5       IN      MX      5 alt2.aspmx.l.google.com.
nkust.edu.tw.           5       IN      MX      10 alt3.aspmx.l.google.com.
nkust.edu.tw.           5       IN      SOA     ns1.nkust.edu.tw. hostmaster. 876 900 600 86400 3600
nkust.edu.tw.           5       IN      NS      ns2.nkust.edu.tw.
nkust.edu.tw.           5       IN      NS      ns1.nkust.edu.tw.
nkust.edu.tw.           5       IN      NS      ns4.nkust.edu.tw.
nkust.edu.tw.           5       IN      NS      ns3.nkust.edu.tw.

Received 510 bytes from 192.168.137.2#53 in 50 ms
```

```shell
$ host 8.8.8.8
8.8.8.8.in-addr.arpa domain name pointer dns.google.
```

```shell
$ host -T stu.edu.tw
stu.edu.tw has address 120.119.28.13
stu.edu.tw has address 120.119.28.14
stu.edu.tw has address 172.30.0.84
stu.edu.tw has IPv6 address 2001:e10:c41:1::13
stu.edu.tw has IPv6 address 2001:e10:c41:1::14
stu.edu.tw mail is handled by 5 alt2.aspmx.l.google.com.
stu.edu.tw mail is handled by 1 aspmx.l.google.com.
stu.edu.tw mail is handled by 10 aspmx2.googlemail.com.
stu.edu.tw mail is handled by 10 aspmx3.googlemail.com.
stu.edu.tw mail is handled by 30 aspmx4.googlemail.com.
stu.edu.tw mail is handled by 30 aspmx5.googlemail.com.
stu.edu.tw mail is handled by 5 alt1.aspmx.l.google.com.
```

## traceroute
可以從此指令得知本地到目標要經過多少 hop。訊息中可以得知每一 hop 的設備名稱和標示甚至可以知道該延遲來自哪個設備。

```shell
$ traceroute www.google.com 
# -n 可避免 DNS 反查
# -I 傳 ICMP Packet
traceroute to www.google.com (216.58.200.228), 30 hops max, 60 byte packets
 1  192.168.137.2 (192.168.137.2)  0.416 ms  0.773 ms  0.640 ms
 2  * * *
 3  * * *
 4  * * *
 5  * * *
 6  * * *
 7  *^C
```

## mtr
算是比 traceroute 更先進，在預設中 ubuntu 並無 traceroute。有著即時的數據。

```shell
$ mtr www.nkust.edu.tw
```

## ss
socket statistics 命令 `ss` 是 `netstat` 的取代未來可能會將 `netstat` 移除，它比 `netstat` 更快，並提供更多訊息。

ss 直接從 kernel 獲取其訊息，不是像 netstat 依賴 `/proc`。

```shell
Netid  State      Recv-Q Send-Q Local Address:Port                 Peer Address:Port
u_str  ESTAB      0      0       * 19250                 * 19251
u_str  ESTAB      0      0      /run/systemd/journal/stdout 22575                 * 22571
u_str  ESTAB      0      0       * 15802                 * 15803
u_str  ESTAB      0      0       * 15803                 * 15802
u_str  ESTAB      0      0      /run/systemd/journal/stdout 15935                 * 15868
u_str  ESTAB      0      0       * 15512                 * 15514
u_str  ESTAB      0      0       * 22571                 * 22575
u_str  ESTAB      0      0      /run/systemd/journal/stdout 15936                 * 15930
u_str  ESTAB      0      0       * 12423                 * 13118
u_str  ESTAB      0      0       * 19603                 * 19604
u_str  ESTAB      0      0       * 15930                 * 15936
u_str  ESTAB      0      0      /run/systemd/journal/stdout 13118                 * 12423
u_str  ESTAB      0      0      /run/systemd/journal/stdout 17241                 * 17240
u_str  ESTAB      0      0      /run/systemd/journal/stdout 15514                 * 15512
u_str  ESTAB      0      0       * 15868                 * 15935
u_str  ESTAB      0      0      /run/systemd/journal/stdout 17640                 * 17461
u_str  ESTAB      0      0      /run/containerd/containerd.sock 19604                 * 19603
u_str  ESTAB      0      0       * 17240                 * 17241
u_str  ESTAB      0      0      /run/systemd/journal/stdout 13120                 * 12543
u_str  ESTAB      0      0       * 16127                 * 16130
u_str  ESTAB      0      0      /run/systemd/journal/stdout 16130                 * 16127
u_str  ESTAB      0      0       * 19251                 * 19250
u_str  ESTAB      0      0       * 15448                 * 15450
u_str  ESTAB      0      0      /run/systemd/journal/stdout 15450                 * 15448
u_str  ESTAB      0      0       * 12543                 * 13120
u_str  ESTAB      0      0       * 14588                 * 14619
u_str  ESTAB      0      0      /run/systemd/journal/stdout 15754                 * 15734
u_str  ESTAB      0      0      /run/systemd/journal/stdout 16129                 * 16066
u_str  ESTAB      0      0       * 16066                 * 16129
u_str  ESTAB      0      0       * 15734                 * 15754
u_str  ESTAB      0      0       * 15741                 * 15804
u_str  ESTAB      0      0      /run/systemd/journal/stdout 17834                 * 17819
u_str  ESTAB      0      0       * 19599                 * 19600
u_str  ESTAB      0      0       * 19410                 * 19411
:
```

顯示 TCP sockets 使用 `-t`，UPD 使用 `-u`，unix socket 使用 `-x`

```shell
$ ss -at
State       Recv-Q Send-Q                                                     Local Address:Port                                                                      Peer Address:Port
LISTEN      0      128                                                                    *:ssh                                                                                  *:*
LISTEN      0      128                                                            127.0.0.1:mysql                                                                                *:*
LISTEN      0      128                                                                    *:http                                                                                 *:*
ESTAB       0      0                                                        192.168.137.142:ssh                                                                      192.168.137.1:11326
LISTEN      0      128                                                                   :::ssh                                                                                 :::*
LISTEN      0      128                                                                   :::http                                                                                :::*
```

查看已建立的 TCP socket 

```shell
$  ss -t4 state established
Recv-Q Send-Q                                                          Local Address:Port                                                                           Peer Address:Port
0      308                                                           192.168.137.142:ssh                                                                           192.168.137.1:11326
```

## arp
IP 位址對應的 MAC 位址，稱為 ARP table。如果連接到 IP 位址，路由器將檢查 MAC 位址。如果已被緩存，則不使用 ARP table。

```shell
$ arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.137.2            ether   00:50:56:ef:e2:19   C                     ens33
192.168.137.254          ether   00:50:56:ef:fe:84   C                     ens33
192.168.137.1            ether   00:50:56:c0:00:08   C                     ens33
```

## tcpdump
可用來蒐集流量並分析，但功能似乎沒 wireshark 齊全。

```shell
$ tcpdump -i <Iface>
$ tcpdump -i <Iface> tcp # 針對 TCP 蒐集
$ tcpdump -i <Iface> port 80 # 針對 port 80 蒐集
$ tcpdump -i <Iface> src <IP> # 針對來源 IP 抓取
$ tcpdump -c <Num> -i <Iface> # -c 表示要抓取的 packet 數量
$ tcpdump -w [PATH] -i <Iface>  # -w 存檔
$ tcpdump -r [PATH] # -r 讀檔
```