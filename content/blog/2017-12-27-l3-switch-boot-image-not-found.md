---
title: 交換器 image 遺失
date: 2017-12-27
description: "交換器的工作"
project: true
tags: [實習, switch]
draft: false
---
## 狀況
今天在弄設備時，我做 erase startup-config 接著 reload。恩... 成功，但是 VLAN 存在 flash ，所以我就把 erase flash，接著 OH NO NO NO 飄眼淚 QQ，因為 image 配我刪了，又是下班時間，立刻請教 google ，最後有回來了，感謝神明保佑!!!

## 方法
用 `xmodem` 救援。

![](https://i.imgur.com/gsiRF53.png)

## 參考資料
[cisco](https://www.cisco.com/c/en/us/support/docs/switches/catalyst-2950-series-switches/41845-192.html)