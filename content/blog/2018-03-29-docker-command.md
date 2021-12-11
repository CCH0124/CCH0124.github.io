---
title: basic command of docker
date: 2018-03-29
description: "Docker 基本指令使用"
tags: [Docker]
draft: false
---

## Table of content
- [Basic command](#Basic-command)
- [Image command](#Image-command)
- [Containers Command](#Containers-Command)

## Basic command 
1. 查看 Docker 訊息
```shell=
docker info
```
2.  登入 hub.docker.com
```shell=
docker login
```

## Image command
1. 查看本地端以下載的 image
```shell=
docker images
```
2. 移除 image
```shell=
docker rmi  -f {image ID} # -f 強制，直接將運行中的 container 刪除
```
3. 取得 container 的資訊
```shell=
docker inspect {ImageID}
```
4. 取得 image 的歷史紀錄
```shell=
docker history  {ImageName}
```

## Containers Command
1. 取得運行的 container
```shell=
docker ps -a
```
2. 運行 container 
```shell=
docker run -it {imageName}
```
3. 取得該 container 的 log
```shell=
docker logs {CONTAINER}
```
4. 啟動 container
```shell=
docker start {ConatainerName}
```
5. 停止 container
```shell=
docker stop {ConatainerName}
```
6. 暫停 container
```shell=
docker pause  <ConatainerName>
```
7. 取消暫停 container
```shell=
docker unpause  {ConatainerName}
```
8. kill 運行中的 container
```shell=
docker kill {ConatainerName}
```
9. 移除 container
```shell=
docker rm {ConatainerName}
```
10. 進入 container
```shell=
docker exec -it {ConatainerName}
```