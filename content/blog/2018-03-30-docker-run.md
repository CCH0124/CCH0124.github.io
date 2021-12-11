---
title: Docker 資源詳解
date: 2018-03-30
description: "Docker 運行時 port、network、volume 設定"
tags: [Docker]
draft: false
---

# Table of content
- [Docker image](#Docker-image)
- [Docker Port mapping](#Docker-Port-mapping)
- [Docker Volume mapping](#Docker-Volume-mapping)
- [Docker Networking](#Docker-Networking)

## Docker image

`Dockerfile` 是建構 `image` 的檔案，建構 `image` 時會讀取 `Dockerfile` 的指令一層一層的建構。而 `container` 是由 image 建構出來的實例。

![](https://i.imgur.com/RsXurEd.png)

### image 特點
##### 分層儲存
當需要修改 image 內的某個檔案時，只會對上方的讀寫層進行改動，不會覆蓋下層既有檔案系統內容。

![](https://i.imgur.com/ix2XQlW.png)

##### Copy-on-Write
從分層儲存可以知道，在建立 `container` 之後我們會在一個可寫層上並進行操作，但是在 `image layers` 的檔案能否修改 ? 是可以的，會複製該檔案至可寫層。

![](https://i.imgur.com/QBHKTL0.png)

### Dockerfil Arg

![](https://i.imgur.com/7WsR2E0.png "網管人")

by [網管人](https://www.netadmin.com.tw/netadmin/zh-tw/technology/7BD73E2A172C4847A3F72D238ACA5148)

##### CMD vs ENTRYPOINT

在 `Dockerfile` 中，只能有一個 `ENTRYPOINT` 或 `CMD` 指令，如果有多個 `ENTRYPOINT` 或 `CMD` 指令則以最後一個為準。

- ENTRYPOINT
    - 往往用於設置容器啟動後的第一個命令，這對一個容器來說往往是固定的。
    - 執行 `docker run` 如果帶有其他命令參數，不會覆蓋 `ENTRYPOINT` 指令
    - `docker run` 的 **--entrypoint** 可以覆蓋 `Dockerfile` 中 `ENTRYPOINT` 設置的命令。
- CMD 
    - 往往用於設置容器啟動的第一個命令的默認參數，這對一個容器來說可以是變化的。`docker run <command>` 往往用於給出替換 `CMD` 的臨時參數。
    - `docker run` 如果帶有其他命令參數，將會覆蓋 `CMD` 指令。
    - 如果在 `Dockerfile` 中，還有 `ENTRYPOINT` 指令，則 `CMD` 指令中的命令將作為 `ENTRYPOINT` 指令中的命令的參數。

### Example
##### Nginx Dockerfile

```shell=
FROM ubuntu:14.04

RUN apt-get update && apt-get install -y nginx
COPY . /var/www/html/
EXPOSE 80
#ENTRYPOINT ["nginx"]
CMD ["nginx", "-g", "daemon off;"]

```
##### Build Dockerfile

使用 `docker build -t {ImageName}:{tag} .` 建構

使用 `docker images` 查看建構的 image

```shell=
REPOSITORY                                          TAG                 IMAGE ID            CREATED             SIZE
nginx                                               v0                  55077273116a        20 minutes ago      222MB

```


## Docker Port mapping

當運行一個 Docker 實例時

```shell
$ docker run
```

，此 `container(實例)` 會在 `Docker host` 上。假設該 `container` 為一個 web 應用程式，在 `container` 中 port 號為 5000，那用戶該如何訪問存取該 web 應用程式，以客戶端來說會輸入 IP 或 Domain，但 `container` 預設給予的 IP 為私有 IP，因此客戶端無法與私有 IP 連接，可以透過 port 映射方式，將 `container` 中 web 應用程式的 5000 port 號，映射到外部主機 80 port，

```shell
$ docker run -p 80:5000 
```

所以客戶端瀏覽時透過外部主機連線，外部主機在路由至 port 5000 的 `container`。

![](https://i.imgur.com/6e9bP4z.png "示意圖")

##  Docker Volume mapping

依照上面的描述，當用 `container` 建立 mysql 時並在 `container` 生命週期內，其資料儲存都是存在的。不過，只要做了**刪除**該 mysql 的 `container` 動作，則該儲存在 mysql 上的資料全都一併消失。因此可藉由 `volume` 或 `bind Mounts` 等進行持續的儲存。

以 `volumn` 來說會有這些優勢
- 遷移與備份較容易
- 可多個 `container` 共享
- 允許應用程式在遠端主機或雲端儲存 `volume`
- 不會增加 `container` 的大小
- 不依賴 `Docker` 的生命週期

![](https://i.imgur.com/nqDLfnx.png "volumn")


![](https://i.imgur.com/lnjEz4a.png)

>Linux 系統中 Docker 的數據預設在 `/var/lib/docker` 中

##### Command

```shell
docker volume create
docker volume inspect
docker volume ls
docker volume prune
docker volume rm
```

## Docker Networking
在 `Docker` 的網路隔離技術，用到了
- namespace
  - 會建立一個擁有網路介面、路由、防火牆等網路資源的環境
- Linux Bridge
  - 將不同主機的網路介面連接
  - `Docker` 安裝完後會有一個預設 `docker0` 的 `Linux Bridge`
- Veth Pair
  - 將 `container` 與本機網路、外部網路達到通訊

在 `Docker` 會有四種模式

##### None
- `container` 擁有自己的 `namespace`，但不幫 `container` 進行任何的網路配置
- `container` 只包含 `loopback` 介面卡，需要使用者自行配置

![](https://i.imgur.com/7Hi5LR4.png)

##### Bridge
- 是利用 `Iptables` 進行 `NAT` 和 `port` 映射
- 此模式下同一主機上的 `container` 可以互相通訊
- `container` 的 `IP` 地址從主機外部不能訪問

![](https://i.imgur.com/3RK6Zgc.png)

##### Host
- 不會為 `container` 建立隔離的網路環境，
- `container` 共享本機的網路命名空間（`/var/run/docker/netns/default`），網路配置與本機一致
- `container` 透過本機的網路介面卡，實現與外部的通訊

![](https://i.imgur.com/NsKKUnP.png)

##### Container
- 新創建的容器和已經存在的某個容器共享同一個 `namespace`，該容器不會擁有自己的網卡
- 此模式下的 `container` 只有網路方面共享數據，文件系統、行程列表等其它方面還是隔離的

![]("https://i.imgur.com/MDefWlA.png")

##### Command

```shell
docker network connect
docker network create
docker network diconnect
docker network inspect
docker network ls
docker network prune
docker network rm
```

## Ref
- [networking 圖來源](https://k2r2bai.com/2016/01/05/container/docker-network/)
- [Docker Tutorial for Beginners - A Full DevOps Course on How to Run Applications in Containers - freeCodeCamp](https://www.youtube.com/watch?v=fqMOX6JJhGo)