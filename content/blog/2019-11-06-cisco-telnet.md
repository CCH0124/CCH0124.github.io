---
title: cisco telnet
date: 2019-11-06
description: "用虛擬機模擬遠端路由器"
tags: [GNS3, Network, telnet]
draft: false
---

## 拓樸
<!-- ![](../assets/img/GNS3/telnet.png "Telnet") -->
{{< figure src="/images/GNS3/telnet.png" width="auto" height="auto">}}

本實驗將 ubuntu16 與 VMware 的虛擬機做鏈接，因此 `R1` 和 ubuntu16 為同一個網段。本實驗練習設定以及用 `wireshark` 觀察 `telnet`。

## 遠端設備
原則上有以下方式可以連接設備
- Console Terminal
- Remote Terminal
- Telnet
- SSH

## R1 設定
##### IP 配置
```shell
R1(config)#interface fastEthernet 0/0
R1(config-if)#ip address 192.168.6.200 255.255.255.0
R1(config-if)#no shutdown
```
##### 啟用遠端

line vty 用來設定 0 到 4 的介面，藉由 login 設定登入時輸入帳號與密碼。要取消 `telnet` 使用 `no login local` 指令。

```shell
R1(config)#line vty 0 4
R1(config-line)#login
R1(config-line)#exit
R1(config)#username cisco7200 privilege 15 password cisco7200
R1(config)#exit
R1#wr
Building configuration...
[OK]
```

>`login` 和 `login local` 在認證方面有些許差異。

## ubuntu16 PC 遠端測試
```shell
login as: cch
cch@192.168.6.128's password:
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-116-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

210 packages can be updated.
131 updates are security updates.


Last login: Wed Nov 13 17:01:39 2019
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

cch@ubuntu:~$ telnet 192.168.6.200
Trying 192.168.6.200...
Connected to 192.168.6.200.
Escape character is '^]'.


User Access Verification

Username: cisco7200
Password:
R1#sho
R1#show bri
R1#show ip int br
R1#show ip int brief
Interface                  IP-Address      OK? Method Status                Protocol
FastEthernet0/0            192.168.6.200   YES manual up                    up
R1#
```

## wireshark 觀察

![](https://i.imgur.com/J7pQsLx.png)

在做遠端連線時會透過三項交握來做一個連線。