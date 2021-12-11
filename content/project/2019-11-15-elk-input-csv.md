---
title: input csv to elasticsearch
date: 2019-11-15
description: "將 CSV 檔透過 logstash 到 Elasticsearch"
tags: [ELK]
draft: false
---

## 想法
實驗透過 `logstash` 傳遞 `CSV` 檔至 `Elasticsearch`。這邊 CSV 檔是來自於[這邊的數據集](https://www.unb.ca/cic/datasets/ddos-2019.html)，因為很懶得用 `python` 來分析資料，且該數據集太肥大，所以想藉由 `ELK` 了力量 XD。

目錄架構
```shell
$ tree 
.
├── docker-compose.yml
├── elasticsearch
│   ├── config
│   │   ├── elasticsearch-node2.yml
│   │   ├── elasticsearch-node3.yml
│   │   └── elasticsearch.yml
│   └── Dockerfile
├── kibana
│   ├── config
│   │   ├── kibana.crt
│   │   ├── kibana.key
│   │   └── kibana.yml
│   └── Dockerfile
├── logstash
│   ├── config
│   │   ├── logstash.yml
│   │   └── pipelines.yml
│   ├── DataSet
│   │    └── DDoS
│   │       ├── 01-12
│   │       │   ├── DrDoS_DNS.csv
│   │       │   ├── DrDoS_LDAP.csv
│   │       │   ├── DrDoS_MSSQL.csv
│   │       │   ├── DrDoS_NetBIOS.csv
│   │       │   ├── DrDoS_NTP.csv
│   │       │   ├── DrDoS_SNMP.csv
│   │       │   ├── DrDoS_SSDP.csv
│   │       │   ├── DrDoS_UDP.csv
│   │       │   ├── Syn.csv
│   │       │   ├── TFTP.csv
│   │       │   └── UDPLag.csv
│   │       └── 03-11
│   │           ├── LDAP.csv
│   │           ├── MSSQL.csv
│   │           ├── NetBIOS.csv
│   │           ├── Portmap.csv
│   │           ├── Syn.csv
│   │           ├── UDP.csv
│   │           └── UDPLag.csv
│   ├── Dockerfile
│   └── pipeline
│       ├── ddos.conf
│       ├── ddos_csv.conf
│       ├── filebeat.conf
│       └── logstash.conf
├── nginx
│   ├── auth
│   ├── cert
│   │   ├── nginx.crt
│   │   └── nginx.key
│   ├── conf.d
│   │   ├── elasticsearch.conf
│   │   └── kibana.conf
│   ├── nginx.conf
│   └── security_header.conf
└── README.md
```

## logstash 配置

##### pipeline 配置
在對於多個不同資料源，無此配置的話在使用 logstash 傳遞至 elasticsearch 時，不同的來源會相互影響到，可利用此方式來進行這些不同資料源的隔離。
```shell=
~/Desktop/docker-elk/logstash/config$ vi pipelines.yml
- pipeline.id: ddos
  queue.type: persisted
  path.config: "/usr/share/logstash/pipeline/ddos_csv.conf"
  pipeline.workers: 1
```
##### 定義傳遞至 Elasticsearch 配置
```shell=
~/Desktop/docker-elk/logstash$ vi pipeline/ddos_csv.conf
input {
  file {
    path => "/usr/share/logstash/DataSet/DDoS/**/*.csv"
    start_position => "beginning"
  }
}

filter {
        csv {
                separator => ","
                autodetect_column_names => true
                autogenerate_column_names => false
                skip_empty_columns => true
                skip_empty_rows => true
                skip_header => true
                id => "ddos"
        }

}

output {
  elasticsearch {
    hosts => "elasticsearch:9200"
    index => "packets-ddos-%{+YYYY-MM-dd}"
    document_type => "csv"
    manage_template => false
  }
}

```

## kiban 視覺化

![](https://i.imgur.com/hqrLIrA.png)

自行新增圖表，這邊使用 `Terms` 聚合方式，`Field` 使用 `Label`。單純從圖表查看說數據集裡 `Label` 的分部有哪些。
![](https://i.imgur.com/0QUnXFG.png)

## 遇到問題
在 CSV 檔 `header` 地方不能有空白，例如 "Apple Type" `Elasticsearch` 會有 `mapping` 錯誤，要變成 "Apple_Type" 之類的，其中 `.` 符號也不能夠出現。因此利用 `sed -i '1 s/./_/g' *.csv` 方式去變更 CSV 檔的 header。

## 結論
效果感覺很可以畢竟記憶體有擴充到 23 GB，但是硬碟有點小可再擴充。會這樣做也是筆電跑這些數據集也是出現記憶體的問題...。未來如果能夠配上 `spark` 來處理資料或許是一個更好的辦法。

## Ref
- [pipelines](https://www.elastic.co/guide/en/logstash/current/multiple-pipelines.html#multiple-pipelines)
- [csv filters](https://www.elastic.co/guide/en/logstash/current/plugins-filters-csv.html)