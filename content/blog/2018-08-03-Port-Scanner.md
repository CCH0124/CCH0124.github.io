---
title: port scanner
data: 2018-08-03
description: "模擬 NMAP Port 掃描"
tags: [scapy, python]
draft: false
---

# Scapy 安裝及實作程式碼介紹
## env
- ubuntu
    - scapy 套件
    - python3

### scapy 套件安裝

```bash
git clone https://github.com/secdev/scapy
cd scapy
sudo python3 setup.py install
```
>因為 windows 沒 fork 功能，所以只能在 unix 環境執行

[scapy install](http://scapy.readthedocs.io/en/latest/installation.html)

## 執行與驗證

#### 執行

```python=
itachi@swarm-master:~/python$ sudo python3 PortScanner.py
[sudo] password for itachi:
[*] Enter Target IP Address: 127.0.0.1
[*] Enter Minumum Port Number: 10
[*] Enter Maximum Port Number: 90

[*] Target is Up, Beginning Scan...
[*] Scanning Started at 10:54:30!

Port 22: Open
Port 80: Open

[*] Scanning Finished!
[*] Total Scan Duration: 0:00:07.653668
```

#### 驗證

```bash
$ netstat -ta
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 *:ssh                   *:*                     LISTEN
tcp        0      1 192.168.15.129:39212    192.168.15.130:2377     SYN_SENT
tcp        0      0 192.168.15.129:ssh      192.168.15.1:62492      ESTABLISHED
tcp6       0      0 [::]:2377               [::]:*                  LISTEN
tcp6       0      0 [::]:http               [::]:*                  LISTEN
tcp6       0      0 [::]:ssh                [::]:*                  LISTEN
```
## scapy 介紹

#### 列出所有協議

```python=
ls()
```
#### 針對 TCP 列出此協議選項

```python=
ls(TCP)
```
#### 獲取所有命令

```python=
lsc()
```

### 發送數據包

#### ping

```python=
send(IP(dst="127.0.0.1")/ICMP())
```
- send（）在第3層（IP）上發送
- sendp（）在layer2（以太網）上發送

#### Layer 3 發送 packet 並等待響應

```python=
(resp, unans) = sr(IP(dst="192.168.1.1")/ICMP(), timeout=3)
```
## TCP Three-Way Handshake

與TCP建立連接時，Server 和 Client 之間會發生 Three-Way Handshake。有許多標記可以在 TCP Packet 中標記。
![Three-Way Handshake](https://d3ojx0qwvsjea2.cloudfront.net/wp-content/uploads/2016/12/24160105/Three-way-Handshake-ex2.png)

## 隱形掃描

隱形掃描中，攻擊者向 Server 發送SYN 標誌。Server 然後用一組SYN和ACK標誌或一組RST和ACK標誌作出回應。如果服務器使用RST和ACK進行回應，則該端口關閉。但是如果 Server 回應SYN和ACK，則端口是開放的。攻擊者然後用RST標誌作出回應，在連接完全建立之前終止連接。
![TCP stealth scan](https://mk0resourcesinfm536w.kinstacdn.com/wp-content/uploads/101613_1123_PortScannin4.jpg)

## 程式

### main

```python=
# TCP flag
SYNACK = 0x12 
RSTACK = 0x14
```
```python=
startTime = getTime() # 掃描開始時間
try:
    targetIP = input("[*] Enter Target IP Address: ")
    minPort = input("[*] Enter Minumum Port Number: ")
    maxPort = input("[*] Enter Maximum Port Number: ")
    ports = checkPort(minPort, maxPort) # 檢查輸入的 port ，Port 範圍 0 ~ 65536
except KeyboardInterrupt:
    print("\n[*] User Requested Shutdown...")
    print("[*] Exiting...")
    sys.exit(1)

#ports = range(int(minPort), int(maxPort)+1)

checkHost(targetIP) # 確認輸入 IP 使否能 ping 
scanStart() 

scanningPrintOpenPort(ports) # 掃描使用者輸入的 Port

endTime = getTime() # 掃描結束時間

print("\n[*] Scanning Finished!")

totalTime(startTime,endTime) # 總共花費時間
```

### module

```python=
def checkPort(minPort, maxPort) :
    try:
        if int(int(minPort) >= 0 and int(maxPort) >= 0 and int(maxPort) >= int(minPort)):

            ports = range(int(minPort), int(maxPort) + 1) # 定義掃描 port 範圍
            return ports
        else:
            print("\n Range of Ports. Error")
            print("[!] Exiting...")
            sys.exit(1)
    except Exception:
         print("\n Range of Ports. Error")
         print("[!] Exiting...")
         sys.exit(1)
```
```python=
def checkHost(targetIP):
    conf.L3socket = L3RawSocket # 是為了讓 lookback 能夠運行
    conf.verb = False # 書初步顯示
    try:
        ping = sr1(IP(dst = targetIP)/ICMP(),timeout=1) # 定義 ping 的封包
        print("\n[*] Target is Up, Beginning Scan...")
    except Exception:
        print("\n[!] Couldn't Resolve Target")
        print("[!] Exiting...")
        traceback.print_exc()
        sys.exit(1)
```
```python=
def scanport(port):
    conf.L3socket = L3RawSocket
    try:
        srcPort = RandShort() # Client 端隨機產生 port 
        conf.verb = False
        SYN_ACK_Packet = sr1(IP(dst = targetIP)/TCP(sport = srcPort, dport = port, flags = "S"),timeout=3) # 定義 TCP SYN/ACK 封包
        packetFlags = SYN_ACK_Packet.getlayer(TCP).flags # 取得 flag
        return isFlags(packetFlags)
        RSTpkt = IP(dst = targetIP)/TCP(sport = srcPort, dport = port, flags = "R") # 定義 TCP　RST 封包
        send(RSTpkt) # 第三層（網路層），發送
    except KeyboardInterrupt:
        RSTpkt = IP(dst = target)/TCP(sport = srcPort, dport = port, flags = "R")
        send(RSTpkt)
        print("\n[*] User Requested Shutdown...")
        print("[*] Exiting...")
        sys.exit(1)

```

```python=
def isFlags(packetFlags): # 比對回應的 TCP flag
    if packetFlags == SYNACK:
        return True
    else:
        return False
        
def getTime():
    return datetime.now() # 取得時間

def totalTime(startTime, endTime):
    print("[*] Total Scan Duration: " + str(endTime - startTime)) # 起始時間跟結束時間，耗費多久

def scanStart():
    print("[*] Scanning Started at " + strftime("%H:%M:%S") + "!\n")

def scanningPrintOpenPort(ports): # 掃描 port 
    for port in ports:
        status = scanport(port)
        isPortOpen(status, port)

def isPortOpen(status, port): # 檢測 port 是否開啟
    if status == True:
        print("Port " + str(port) + ": Open")
```

[本人的實作](https://github.com/CCH0124/STU_Homework/blob/master/python/PortScanner/PortScanner.py)

## 問題解決

###  Warning Message

導入 scapy 時刪除 `找不到IPv6目標的路由` 錯誤
```python=
import logging
logging.getLogger("scapy.runtime").setLevel(logging.ERROR)
from scapy.all import *
import sys 
from datetime import datetime
from time import strftime
```
### 得不到回應

```python=
ping = sr1(IP(dst = "127.0.0.1")/ICMP(),timeout=3)
print(ping)
Begin emission:
.Finished sending 1 packets.
..............................................................................
Received 79 packets, got 0 answers, remaining 1 packets
None
```
一整個沒回應，新增如下即可

```python=
conf.L3socket = L3RawSocket
```
### 未知問題

掃描本機端是可行的，但是針對外面的主機進行掃描，有時掃到某個 port(99、105、2001) 會噴錯，錯誤目前未知。

## 參考資料

[PortScanner](https://null-byte.wonderhowto.com/how-to/build-stealth-port-scanner-with-scapy-and-python-0164779/)
[PortScanner](https://resources.infosecinstitute.com/port-scanning-using-scapy/#gref)
[L3socket](https://scapy.readthedocs.io/en/latest/troubleshooting.html?highlight=L3socket)
