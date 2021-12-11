---
title: Docker 基礎介紹
date: 2019-10-05
description: "Docker 基礎介紹"
tags: [Docker, container]
draft: false
---

## Why do you need Docker
從先前的架構來看，建置一個網站需要有 `Apache`、`MySQL`、`PHP` 等。其中它們都依賴於當前的作業系統或著有些服務因為版本而依賴於不同的函式庫。因此這讓架構會變得有點凌亂導致新進的工程師必須去了解此凌亂的架構才能進行工作，而 `Docker` 技術能夠幾乎解決這樣的問題。


## What can I do 

從下圖看出，在作業系統上裝一個 `Docker`，並透過 `docker run` 去建置容器。`Docker` 能夠將每個運行服務的每一個組件，丟至該服務容器（container）中，讓每個服務依賴於該容器中被定義的 `image`。

![](https://i.imgur.com/MA3Ek1u.png)
by docker.com

那這又跟 VM 又有麼差異呢 ? 在資源使用率上會有[差異](https://www.inwinstack.com/2017/10/13/vm-container-difference/)。

## What is container

一種隔離環境的技術， 可以擁有自己的 `process`、`network`、`mount`，但是容器會共用 `kernel`。在又 `Docker` 此技術之前，容器的概念先前就有，像是 `lkx`、`chroot` 等。但整合方面的完整性都歸功於 `Docker`。
`container` 應該要說是一種概念，`Docker` 是一個實作 `container` 的技術。`container` 竟然是一種概念，它必定有規範，稱作 `Open Container Initiative (OCI)`。藉由此標準可以提升 `container` 不同解決方案的相容性。
然而，`Open Container Initiative (OCI)` 定義兩大的標準
- [Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/master/spec.md)
- [Image Specification](https://github.com/opencontainers/image-spec/blob/master/spec.md)

對於 `Docker` 來說，`image` 相關的操作 `images/pull/push` 等甚至是藉由 `Dockerfile` 來建立自己的工作環境，這些都牽扯到了 `image`。`Image Specification` 就是來規範 `Image`。
相對的 `Runtime Specification` 是控制 `container` 的生命週期 `create/delete/start/stop` 或是運行 `container` 時透過 `exec/attach` 與 `container` 的互動。對於 `container` 中，隔離會使用 `namespace` 來完成，`namespace` 操作會有 `pid`、`network`、`ipc`、`mount` 等。

## Run container
`Dockerfile` 是建構 `image` 的檔案，而 `container` 就是運行 `image` 的結果。

![](https://geekflare.com/wp-content/uploads/2019/07/dockerfile.png)
by geekflare.com

### Example
```shell
/test$ tree
.
└── Dockerfile
```

##### Define nginx Dockerfile

```shell
FROM ubuntu:14.04

RUN apt-get update && apt-get install -y nginx

COPY . /var/www/html/

EXPOSE 80

#ENTRYPOINT ["nginx"]
CMD ["nginx", "-g", "daemon off;"]
```

##### Build

使用 `docker build -t {ImageName}:{tag01} .` 建構

使用 `docker images` 查看建構的 image

```shell
REPOSITORY                                          TAG                 IMAGE ID            CREATED             SIZE
nginx                                               v0                  55077273116a        20 minutes ago      222MB
```

##### Fix Dockerfile
`image` 原則上應該越小越好

```shell
FROM ubuntu:14.04

RUN apt-get update && apt-get install -y nginx \
        && rm -rf /var/lib/apt/lists/*

COPY . /var/www/html/

EXPOSE 80

#ENTRYPOINT ["nginx"]
CMD ["nginx", "-g", "daemon off;"]

```

```shell
REPOSITORY                                          TAG                 IMAGE ID            CREATED             SIZE
nginx                                               v0.1                d17658937ea9        2 minutes ago       209MB
```

上面就成功運行了一個 `container`。

## ref
- [官網 what-container](https://www.docker.com/resources/what-container)
- [淺談 Container 設計原理(I)](https://www.hwchiu.com/container-design-i.html)
- [dockerfile-tutorial](https://geekflare.com/dockerfile-tutorial/)