---
title:  logstash Multiple Pipelines
date: 2018-12-22
description: "數據分離"
tags: [ELK, logstash]
draft: false
---
# logstash Multiple Pipelines
遇到的問題是，使用 `filebeat` 將 json 數據傳遞給 `logstash` 處裡並建立索引 pcap 與使用 logstash 讀取 CSV 數據建立 CSV 索引。
兩者都是不同的索引。但是 `kibana` 上的 pcap 索引卻讀取 csv 索引數據。這造成分析上的錯誤。以下會先了解 `logstash` 運作以及解決方式。

Logstash 事件處理管道有三個階段：inputs → filters → outputs。
- inputs
    - 生成事件
- filters
    - 修改它們，輸出將它們發送到其他地方

輸入和輸出支援編解碼，能夠在數據進入或退出管道時對數據進行編碼或解碼，而無需使用單獨的 filters。

## inputs
將輸入的數據導入至 logstash。
常用的輸入：
- file
    - 從文件系統上的文件讀取，與UNIX命令非常相似 tail -0F
- syslog
    - 在已知端口514上偵聽syslog消息並根據RFC3164格式進行解析
- redis
    - 使用 redis 通道和 redis 列表從 redis 服務器讀取。
    - Redis 通常用作集中式 Logstash 安裝中的"broker"，該安裝將 Logstash 事件從遠程 Logstash "shippers" 排隊。
- beats
    - 處理 Beats發送的事件

[plugins](https://www.elastic.co/guide/en/logstash/current/input-plugins.html)

## Filters
是 Logstash 管道中的中間處理機制。可以將過濾與條件組合，以便在事件滿足特定條件時對其執行操作。
常用的過濾：
- grok
    - 解析並構造任意數據
    - 目前 Logstash 中將**非結構化** log 數據解析為結構化和可查詢內容的最佳方式
    - Logstash 內置了 120 種模式
- mutate
    - 對事件字段執行常規轉換
    - 可以重命名、刪除、替換和修改事件中的字段
- drop
    - 完全刪除事件，例如 *debug* 事件
- clone
    - 製作事件的副本，可能添加或刪除字段
- geoip
    - 添加有關 IP 地址的地理位置的信息

[plugins](https://www.elastic.co/guide/en/logstash/current/filter-plugins.html)

## Outputs
輸出是 Logstash 管道的最後階段。事件可以傳遞多個輸出，但是一旦所有輸出處理完成，事件就完成了它的執行。
常用的輸出：
- elasticsearch
    - 將事件數據發送給 Elasticsearch。
    - 計劃以高效、方便且易於查詢的格式保存數據...... Elasticsearch 是最佳選擇。
- file
    - 將事件數據寫入 Disk 上的 file。
- [graphite](http://graphite.readthedocs.io/en/latest/)
    - 將事件數據發送到 graphite，這是一種用於存儲和繪製指標的流行開源工具。
- statsd
    - 將事件數據發送到 statsd，這是一種"偵聽統計信息，如計數器和定時器，通過 UDP 發送並將聚合發送到一個或多個可插入後端服務"的服務。

## Codecs
編解碼器基本上是串流過濾器，可以作為輸入或輸出的一部分運行。使用編解碼器可以輕鬆地將消息傳輸與序列化過程分開。流行的編解碼器包括 json、msgpack 和 plain。

- json
    - 以 JSON 格式編碼或解碼數據
- multiline
    - 將多行文字事件（例如 java 異常和 stacktrace 消息）合併到一個事件中。

[plugins](https://www.elastic.co/guide/en/logstash/current/codec-plugins.html)


## 問題解決
藉由 Multiple Pipelines 來實現。這裡需再 logstahs 的 config 目錄裡新增 `pipelines.yml` 檔案，
這邊在 docker-compose 已經做一個掛載的動作，不須再而外掛載。

### pipelines.yml
透過 `pipelines.yml` 的配置文件中讀取它們的定義。
其中 `logstash.yml` 中的 `path.config: /usr/share/logstash/pipeline` 必須註解，未在 `pipelines.yml` 檔案中寫入設置的值將回參考 `logstash.yml` 設置檔案中指定的默認值。

```shell=
logstash$ vi config/pipelines.yml
- pipeline.id: csv
  queue.type: persisted
  path.config: "/usr/share/logstash/pipeline/readCSV.conf"
- pipeline.id: pcap
  queue.type: persisted
  path.config: "/usr/share/logstash/pipeline/pcap.conf"
- pipeline.id: metricbeat
  queue.type: persisted
  path.config: "/usr/share/logstash/pipeline/metricbeat.conf"

```

## 參考資料
[logstash-multiple-pipelines](https://www.elastic.co/blog/logstash-multiple-pipelines)
[discuss](https://discuss.elastic.co/t/different-files-are-entered-into-logstash-kibana-fields-show-the-same/161243/8)