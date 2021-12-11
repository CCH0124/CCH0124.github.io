---
title: basic command of docker-compose
date: 2018-03-29
description: "docker-compose 基本指令使用"
tags: [Docker, docker-compose]
draft: false
---

# docker compose
`Docker compose` 是一個用於定義和運行多個容器的 Docker 的工具。
它使用 yml 檔案定義應用程式服務。

- docker-compose 指令基於每個工作目錄
- 可以在一台電腦上運行多個 Docker 容器


## Table of content
- [run yml file](#run-yml-file)
- [將容器及環境配置移除](#將容器及環境配置移除)
- [查看 container 運行資訊](#查看-container-運行資訊)
- [kill 特定 container](#kill-特定-container)
- [刪除 container](#刪除-container)

### run yml file
```shell=
docker-compose up -d -f {.yml} # -d 後臺運行，-f 指定 docker-compose yml 檔案
```
### 將容器及環境配置移除
```shell=
docker-compose down 
```

### 查看 container 運行資訊
```shell=
docker-compose ps
```

### kill 特定 container
```shell=
docker-compose  kill {service}
```

### 刪除 container
要將刪除的 container 先停止運行
```shell=
docker-compose  rm {service} 
```