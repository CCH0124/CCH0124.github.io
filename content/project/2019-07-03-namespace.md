---
title: Net Namespaces
date: 2019-07-03
description: "我想要擁有私人空間"
tags: [OVS, Ubuntu]
draft: false
---

# Namespaces
###### tags: `SDN`

`Namespaces` 是 `Linux kernel` 的一項功能，它對 `kernel` 資源進行區分隔離，使得一組行程看到一組資源，而另一組行程看到一組不同的資源。
好比一棟房子有多個房間，一個房間表示一組資源，妹妹住一間、哥哥住一間，彼此之間是相互看不到內部資源的，為每一個人提供隱私。

可分為下面幾列
- pid Namespaces
- net Namespaces
- ipc Namespaces
- mnt Namespaces
- uts Namespaces
- user Namespaces

>`Namespaces` 不限制對 `CPU`、`memory` 和 `disk` 等物理上資源的存取。該存取由 `cgroups` 的 kernel 功能計算和限制。

## Experiment
### Create network namespace
```shell
cch@ubuntu:~$ sudo ip netns add red # create a new named network namespace
cch@ubuntu:~$ sudo ip netns add blue
cch@ubuntu:~$ sudo ip netns list # show all of the named network namespaces
blue
red
```

![](https://i.imgur.com/50TCfDt.jpg)

此圖是利用上面指令達成的構圖。建立 red 和 blue 的 namespace。

>`ip netns list` 其實是顯示 `/var/run/netns` 下 `Namespace`。

### Exec in network namespace
##### Run cmd in the named network namespace
查看該 namespace 的介面卡
```shell
cch@ubuntu:~$ sudo ip netns exec red ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

> `ip netns` 是行程網路 `namespace` 管理

##### Local information
透過比較 `ARP`、`Route` 此方式，可以比較出 namespace 中的環境與本機差異。

```shell
cch@ubuntu:~$ arp
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.137.2            ether   00:50:56:ef:e2:19   C                     ens33
192.168.137.1            ether   00:50:56:c0:00:08   C                     ens33
cch@ubuntu:~$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.137.2   0.0.0.0         UG    0      0        0 ens33
172.17.0.0      *               255.255.0.0     U     0      0        0 docker0
192.168.137.0   *               255.255.255.0   U     0      0        0 ens33
cch@ubuntu:~$ sudo ip netns exec red route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
```
##### Create veth

`veth` 設備是虛擬網路設備。可以充當網路 namespace 之間的隧道，讓彼此藉由此通道進行溝通，但也可以用作獨立的網路設備。

```shell
cch@ubuntu:~$ sudo ip link add veth-red type veth peer name veth-blue # Create veth pair
cch@ubuntu:~$ sudo ip link show type veth
4: veth-blue@veth-red: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 7a:40:fb:ab:41:4c brd ff:ff:ff:ff:ff:ff
5: veth-red@veth-blue: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether b6:ad:e3:c8:8e:a8 brd ff:ff:ff:ff:ff:ff
cch@ubuntu:~$ sudo ip link set veth-red netns red # Move the global namespace to this namespace
cch@ubuntu:~$ sudo ip link set veth-blue netns blue
cch@ubuntu:~$ sudo ip -n red addr add 172.10.10.1/24 dev veth-red # execution cmd
cch@ubuntu:~$ sudo ip -n blue addr add 172.10.10.5/24 dev veth-blue
cch@ubuntu:~$ sudo ip -n red link set veth-red up
cch@ubuntu:~$ sudo ip -n blue link set veth-blue up
```

```shell
cch@ubuntu:~$ sudo ip netns exec red ping -c1 172.10.10.5 # execution cmd
PING 172.10.10.5 (172.10.10.5) 56(84) bytes of data.
64 bytes from 172.10.10.5: icmp_seq=1 ttl=64 time=0.027 ms

--- 172.10.10.5 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.027/0.027/0.027/0.000 ms
cch@ubuntu:~$ sudo ip netns exec red arp
Address                  HWtype  HWaddress           Flags Mask            Iface
172.10.10.5              ether   5a:59:3d:3e:e6:2f   C                     veth-red
```

##### Compare ARP

```shell
cch@ubuntu:~$ sudo ip netns exec red arp
Address                  HWtype  HWaddress           Flags Mask            Iface
172.10.10.5              ether   5a:59:3d:3e:e6:2f   C                     veth-red
cch@ubuntu:~$ sudo ip netns exec blue arp
Address                  HWtype  HWaddress           Flags Mask            Iface
172.10.10.1              ether   6e:ef:a9:4c:44:ce   C                     veth-blue
cch@ubuntu:~$ arp
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.137.254          ether   00:50:56:e3:40:5b   C                     ens33
192.168.137.1            ether   00:50:56:c0:00:08   C                     ens33
192.168.137.2            ether   00:50:56:ef:e2:19   C                     ens33

```

![](https://i.imgur.com/ME1Mbv3.jpg)

透過上面，將 `red` 與 `blue` 建立一個橋梁，讓它們能夠相互溝通。

## OVS

一種虛擬交換器，可用來作為 L2 switch，可切割網域、QoS 或是流量監控，同時支持 openFlow 協定。

![](https://i.imgur.com/Um3k70a.jpg)

如果有多個 namespace，要如何讓它們能夠相互通訊?可以藉由創建虛擬交換機方式實現。


### create virtual switch
```shell
cch@ubuntu:~$ sudo ovs-vsctl add-br v-net-0
cch@ubuntu:~$ sudo ovs-vsctl show
8680f06e-e248-422a-9792-d0f2c5777dd3
    Bridge "v-net-0"
        Port "v-net-0"
            Interface "v-net-0"
                type: internal
    ovs_version: "2.11.90"
```

```shell
cch@ubuntu:~$ ip link | grep v-net-0 -A 3
21: v-net-0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether e2:28:23:77:27:4a brd ff:ff:ff:ff:ff:ff
```

![](https://i.imgur.com/i4LMIou.jpg)

透過上述，用 `OVS` 建立一個虛擬交換器。

##### delete bridge
將先前建立的橋樑給刪除，只要將一端刪除，另一端也會隨著消失。

這邊是將環境還原至一開始只存在兩個 namespace 的地方。

```shell
cch@ubuntu:~$ sudo ip -n red link del veth-red
```

### Configuration environment
建立橋梁，與虛擬交換器環境結合。

```shell
cch@ubuntu:~$ sudo ip link add veth-red type veth peer name veth-red-br
cch@ubuntu:~$ sudo ip link add veth-blue type veth peer name veth-blue-br
cch@ubuntu:~$ sudo ip link set veth-red netns red
cch@ubuntu:~$ sudo ovs-vsctl add-port v-net-0 veth-red-br
cch@ubuntu:~$ sudo ip link set veth-blue netns blue
cch@ubuntu:~$ sudo ovs-vsctl add-port v-net-0 veth-blue-br
cch@ubuntu:~$ sudo ovs-vsctl show
8680f06e-e248-422a-9792-d0f2c5777dd3
    Bridge "v-net-0"
        Port veth-blue-br
            Interface veth-blue-br
        Port "v-net-0"
            Interface "v-net-0"
                type: internal
        Port veth-red-br
            Interface veth-red-br
    ovs_version: "2.11.90"
cch@ubuntu:~$ sudo ip link show type veth
24: veth-red-br@if25: <BROADCAST,MULTICAST> mtu 1500 qdisc noop master ovs-system state DOWN mode DEFAULT group default qlen 1000
    link/ether 76:bb:f6:db:46:05 brd ff:ff:ff:ff:ff:ff link-netnsid 0
26: veth-blue-br@if27: <BROADCAST,MULTICAST> mtu 1500 qdisc noop master ovs-system state DOWN mode DEFAULT group default qlen 1000
    link/ether ce:59:0d:c9:1f:bf brd ff:ff:ff:ff:ff:ff link-netnsid 1
cch@ubuntu:~$ sudo ip -n red addr add 172.10.10.1/24 dev veth-red
cch@ubuntu:~$ sudo ip -n blue addr add 172.10.10.5/24 dev veth-blue
cch@ubuntu:~$ sudo ip -n red link set veth-red up
cch@ubuntu:~$ sudo ip -n blue link set veth-blue up
```

![](https://i.imgur.com/wAkyKdV.jpg)

透過上述，將 red 和 blue 的橋樑與虛擬交換器建立。此時橋樑介面尚未啟動。

### Test connection

##### Start interface
將橋樑介面啟動。

```shell
cch@ubuntu:~$ sudo ifconfig v-net-0 up
cch@ubuntu:~$ sudo ifconfig veth-red-br up
cch@ubuntu:~$ sudo ifconfig veth-blue-br up
```
##### Test ping 
```shell
cch@ubuntu:~$ sudo ip netns exec red ping -c1 172.10.10.5
PING 172.10.10.5 (172.10.10.5) 56(84) bytes of data.
64 bytes from 172.10.10.5: icmp_seq=1 ttl=64 time=0.214 ms

--- 172.10.10.5 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.214/0.214/0.214/0.000 ms
```

##### add namespace
再新增兩個 namespace 進行實驗。

```shell
cch@ubuntu:~$ sudo ip netns add green
cch@ubuntu:~$ sudo ip netns add yellow
cch@ubuntu:~$ sudo ip netns list
yellow
green
blue (id: 1)
red (id: 0)
```

##### Configuration environment
將新增的兩個 namespace 建立環境。

```shell
cch@ubuntu:~$ sudo ip link add veth-green type veth peer name veth-green-br
cch@ubuntu:~$ sudo ip link add veth-yellow type veth peer name veth-yellow-br
cch@ubuntu:~$ sudo ip link set veth-green netns green
cch@ubuntu:~$ sudo ip link set veth-yellow netns yellow
cch@ubuntu:~$ sudo ovs-vsctl add-port v-net-0 veth-green-br
cch@ubuntu:~$ sudo ovs-vsctl add-port v-net-0 veth-yellow-br
cch@ubuntu:~$ sudo ip -n green addr add 172.10.10.10/24 dev veth-green
cch@ubuntu:~$ sudo ip -n yellow addr add 172.10.10.20/24 dev veth-yellow
cch@ubuntu:~$ sudo ip -n green link set veth-green up
cch@ubuntu:~$ sudo ip -n yellow link set veth-yellow up
cch@ubuntu:~$ sudo ifconfig veth-green-br up
cch@ubuntu:~$ sudo ifconfig veth-yellow-br up
cch@ubuntu:~$ sudo ip netns exec red ping -c1 172.10.10.20
PING 172.10.10.20 (172.10.10.20) 56(84) bytes of data.
64 bytes from 172.10.10.20: icmp_seq=1 ttl=64 time=0.611 ms

--- 172.10.10.20 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.611/0.611/0.611/0.000 ms
```
##### View route and ARP

```shell
cch@ubuntu:~$ sudo ip netns exec yellow ip route
172.10.10.0/24 dev veth-yellow  proto kernel  scope link  src 172.10.10.20
cch@ubuntu:~$ sudo ip netns exec yellow arp
Address                  HWtype  HWaddress           Flags Mask            Iface
172.10.10.1              ether   22:0f:19:e1:85:c1   C                     veth-yellow
```


![](https://i.imgur.com/VBEDq3d.jpg)

最終的構圖。

## Advanced

##### Test the local host
原因是 blue 裡路由表沒有 `192.168.137.0/24` 的資訊

```shell
cch@ubuntu:~$ sudo ip netns exec blue ping 192.168.137.133
connect: Network is unreachable
```

```shell
cch@ubuntu:~$ sudo ip netns exec blue route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
172.10.10.0     *               255.255.255.0   U     0      0        0 veth-blue
```

IP 設置為 `private IP`，這裡重新分配。

```shell
cch@ubuntu:~$ sudo ip netns exec red ip addr del 172.10.10.1/24 dev veth-red
cch@ubuntu:~$ sudo ip netns exec blue ip addr del 172.10.10.5/24 dev veth-blue
cch@ubuntu:~$ sudo ip netns exec green ip addr del 172.10.10.10/24 dev veth-green
cch@ubuntu:~$ sudo ip netns exec yellow ip addr del 172.10.10.20/24 dev veth-yellow
cch@ubuntu:~$ sudo ip -n red addr add 172.16.10.1/24 dev veth-red
cch@ubuntu:~$ sudo ip -n blue addr add 172.16.10.5/24 dev veth-blue
cch@ubuntu:~$ sudo ip -n green addr add 172.16.10.10/24 dev veth-green
cch@ubuntu:~$ sudo ip -n yellow addr add 172.16.10.20/24 dev veth-yellow
```

將交換機分配 IP 作為 `gateway`，讓不屬於 red、blue、green、yellow 的封包能夠從此地方出去，再藉由本地 host 轉發。

```shell
cch@ubuntu:~$ sudo ifconfig v-net-0 172.16.10.254 netmask 255.255.255.0
```

##### Setting routeing
設置一條路由，讓 blue 能夠與本機 host 溝通。
```shell
cch@ubuntu:~$ sudo ip netns exec blue ip route add 192.168.137.0/24 via 172.16.10.254
cch@ubuntu:~$ sudo ip netns exec blue route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
172.16.10.0     *               255.255.255.0   U     0      0        0 veth-blue
192.168.137.0   172.16.10.254   255.255.255.0   UG    0      0        0 veth-blue
cch@ubuntu:~$ route # 本機 host 路由
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.137.2   0.0.0.0         UG    0      0        0 ens33
172.16.10.0     *               255.255.255.0   U     0      0        0 v-net-0
172.17.0.0      *               255.255.0.0     U     0      0        0 docker0
192.168.137.0   *               255.255.255.0   U     0      0        0 ens33
cch@ubuntu:~$ sudo ip netns exec blue ping 192.168.137.133
PING 192.168.137.133 (192.168.137.133) 56(84) bytes of data.
64 bytes from 192.168.137.133: icmp_seq=1 ttl=64 time=0.103 ms
64 bytes from 192.168.137.133: icmp_seq=2 ttl=64 time=0.096 ms
64 bytes from 192.168.137.133: icmp_seq=3 ttl=64 time=0.103 ms
^C
--- 192.168.137.133 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2000ms
rtt min/avg/max/mdev = 0.096/0.100/0.103/0.012 ms

```
##### Connect to the external network and the same area network host
```shell
cch@ubuntu:~$ sudo ip netns exec blue ping 8.8.8.8
connect: Network is unreachable
```

在 blue 裡並沒有默認路由讓封包出去，因此在 blue 裡設置默認路由，讓不知道該怎麼處裡的封包丟網默認路由。

```shell
cch@ubuntu:~$ sudo ip netns exec blue ip route add default via 172.16.10.254
```

但是因為 `private IP` 關係，因此無法對外網做一個互相溝通的動作
```shell
cch@ubuntu:~$ sudo tcpdump -i v-net-0 icmp
[sudo] password for cch:
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on v-net-0, link-type EN10MB (Ethernet), capture size 262144 bytes
21:40:48.014037 IP 172.16.10.5 > dns.google: ICMP echo request, id 2610, seq 1, length 64
21:40:49.012938 IP 172.16.10.5 > dns.google: ICMP echo request, id 2610, seq 2, length 64
21:40:50.012489 IP 172.16.10.5 > dns.google: ICMP echo request, id 2610, seq 3, length 64
21:40:51.012496 IP 172.16.10.5 > dns.google: ICMP echo request, id 2610, seq 4, length 64
21:40:52.013086 IP 172.16.10.5 > dns.google: ICMP echo request, id 2610, seq 5, length 64
21:40:53.012835 IP 172.16.10.5 > dns.google: ICMP echo request, id 2610, seq 6, length 64
21:40:54.012351 IP 172.16.10.5 > dns.google: ICMP echo request, id 2610, seq 7, length 64
21:40:55.012690 IP 172.16.10.5 > dns.google: ICMP echo request, id 2610, seq 8, length 64
21:40:56.012742 IP 172.16.10.5 > dns.google: ICMP echo request, id 2610, seq 9, length 64
21:40:57.012232 IP 172.16.10.5 > dns.google: ICMP echo request, id 2610, seq 10, length 64
21:40:58.012571 IP 172.16.10.5 > dns.google: ICMP echo request, id 2610, seq 11, length 64
^C
```
使用 `NAT` 封裝，讓 `private IP` 能夠對外連線。
```shell
cch@ubuntu:~$ sudo iptables -t nat -A POSTROUTING -s 172.16.10.0/24 -o ens33 -j MASQU
ERADE
$ sudo iptables -F 
$ sudo iptables -P FORWARD ACCEPT
$ sudo sysctl -w net.ipv4.ip_forward=1
cch@ubuntu:~$ sudo ip netns exec blue ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=127 time=232 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=127 time=34.1 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=127 time=59.7 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=127 time=194 ms
64 bytes from 8.8.8.8: icmp_seq=5 ttl=127 time=319 ms
```
查看 `NAT` 規則
```shell
cch@ubuntu:~$ sudo iptables -t nat -vnL POSTROUTING --line-number
Chain POSTROUTING (policy ACCEPT 3 packets, 464 bytes)
num   pkts bytes target     prot opt in     out     source               destination 
1        0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
2        2   168 MASQUERADE  all  --  *      ens33   172.16.10.0/24       0.0.0.0/0  
```

透過上述的設定，同區網的主機基本上能夠連線溝通。那其實用了預設路由，就可以不用先前設置與本機 host 連線那段路由，讓封包透過預設路由出去即可。


![](https://i.imgur.com/aTUu68S.jpg)

整個架構上，如上圖

## Additional
### Delete Namespace
```shell
$ sudo ip netns del red
```
### Delete Addr
```shell
cch@ubuntu:~$ sudo ip netns exec red ip addr del 172.10.10.1/24 dev veth-red
```
### Delete Port
```shell
cch@ubuntu:~$ sudo ovs-vsctl del-port v-net-0 red
```
### Delete Bridge
```shell
cch@ubuntu:~$ sudo ovs-vsctl del-br ovs-br
```
### View Routing table
```shell
cch@ubuntu:~$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         _gateway        0.0.0.0         UG    100    0        0 ens33
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-e23b4bf51b20
172.19.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-59d8f43684ef
192.168.137.0   0.0.0.0         255.255.255.0   U     0      0        0 ens33
_gateway        0.0.0.0         255.255.255.255 UH    100    0        0 ens33
```

Flags
- U (route is up)：該路由是啟動的
- H (target is a host)：目標是一部主機 (IP) 而非網域
- G (use gateway)：需要透過外部的主機 (gateway) 來轉遞封包
- R (reinstate route for dynamic routing)：使用動態路由時，恢復路由資訊的旗標
- D (dynamically installed by daemon or redirect)：已經由服務或轉 port 功能設定為動態路由
- M (modified from routing daemon or redirect)：路由已經被修改了
- ! (reject route)：這個路由將不會被接受(用來抵擋不安全的網域)


### Remove Specific Route
```shell
$ sudo ip route del default via 172.16.10.254
```

### Delete NAT 
```shell
cch@ubuntu:~$ sudo iptables -t nat -D POSTROUTING 2 # delete numbet two
```
### View NAT Table
```shell
cch@ubuntu:~$ sudo iptables -t nat -vnL POSTROUTING --line-number

-t nat：選擇 nat 表。
-v：詳細輸出。
-L：列出所選鏈中的所有規則，顯示 nat 表中的所有規則。
-n：數字輸出。IP 地址和端口號將以數字格式打印
--line-number：列出規則時，將行號添加到每個規則的開頭。可以使用行號來刪除 nat 規則。

```
### View Chain
```shell
cch@ubuntu:~$  sudo iptables -L FORWARD --line-numbers
```
### delete Chain
```shell
cch@ubuntu:~$ sudo iptables -D FORWARD 1 # 刪除第一個規則
```

## Ref
[1](https://matthewarcus.wordpress.com/2018/02/04/veth-devices-network-namespaces-and-open-vswitch/)

[2](http://crosbymichael.com/creating-containers-part-1.html)

[3](https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/)

[4](https://www.netadmin.com.tw/netadmin/zh-tw/technology/A54919443A1C442389BD38E531D5B4FB)

[5](https://www.youtube.com/watch?v=j_UUnlVC2Ss&list=PLoLSpL6iW1uCojnvEffg8RSgBEbU3GbgA&index=2&t=170s)