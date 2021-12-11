---
title: throughout
date: 2019-05-21
description: "我能一次吞多少啊"
tags: [Network]
draft: false
---

Throughput（吞吐量）是指系統可以在指定的時間內處理的單元的數量。像是 internet 之類的通訊網路的環境中，吞吐量是訊息成功傳遞的速率。

## 什麼是 Throughput
網路吞吐量通常表示為平均值，以每秒位數（bps）為單位，或者在某些情況下以每秒 `data packets` 為單位。吞吐量是網路連接性能和質量的重要指標。不成功的訊息傳遞的高比率，會導致較低的吞吐量和降低性能的問題。

網路設備通過交換 `data packets` 進行通訊。吞吐量表示從網路上的一個點到另一個點的**成功數據包傳送的級別**。沿途丟棄 `data packets` 會降低吞吐量和網路連接質量。對於某些即時服務要求的吞吐量就會較高。

網路吞吐量受許多因素的影響，這些屬性包括

- 物理硬體的處理能力
- 電纜和路由器
- 網路擁塞和 `data packets` 丟失也會對吞吐量產生影響

## Bandwidth 與 Throughput 差異
Bandwidth（頻寬）是指 internet 通道的大小。internet 通訊通常用稱為 `data packets` 的數據形式產生。頻寬是指這些 `data packets` 的大小以及可以同時通過 internet 通道傳輸的數量。與吞吐量的一個重要區別是**頻寬是指 internet 通道的實際大小或容量**；**吞吐量是指實際傳輸的 `data packets` 的數量**。

使用高速公路的比喻，頻寬將是在一段時間內沿著該高速公路行駛的汽車總數（越多通道所傳送的資料越多）。事故和道路封閉後，吞吐量將是實際通過高速公路長度的汽車數量。

## Throughput 和網際網路速度
大多數人認為 internet 速度是下載或上傳檔案所需的時間。速度也可以指硬體設備或 internet 連接的 "rated speed"。

例如，經常聽到超快 100 Mbps 的 internet 連接。預設情況下，這些速度表示特定 Internet 連接的吞吐量。事實上，這些 internet 連接速度可以更準確的描述為連接的實際帶寬，實際數據傳輸容量或吞吐量可能要低得多。