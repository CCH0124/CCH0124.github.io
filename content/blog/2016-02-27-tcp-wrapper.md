---
title: "TCP wrapper"
date: 2016-02-27
tags: ["Ubuntu", "security", "TCP wrapper"]
categories: ["Ubuntu"]
description: "security tool"
draft: false
---

# TCP wrapper
- 簡易 TCP 連線存取限制工具
- 修改設定檔即時生效，因此不必重啟服務
- 設定檔
    - `/etc/hosts.allow`
    - `/etc/hosts.deny`
- 限制多，須檢查服務是否支援
```shell
$ ldd /usr/sbin/sshd
...
libwrap.so.0 => /lib/x86_64-linux-gnu/libwrap.so.0 (0x00007fd754b92000)
...
$ ldd /usr/sbin/nginx # 無 libwrap 因此不支援
```

格式
```shell
執行檔名:來源位置:規則
```

##### Example
讓 sshd 只信任 192.168.221.0 和 192.168.18.0 全網段連線，其餘阻擋
```shell
$ vi /etc/hosts.allow
ALL:127.0.0.1 [::1]
sshd:192.168.221. 192.168.18.
$ vi /etc/hosts.deny
sshd:ALL
```