---
title: kubernetes 網路
date: 2021-12-12
description: "kubernetes 網路"
tags: [kubernetes]
draft: true
---

以 Kubernetes 角度來看，網路大致有四種通訊類型

1. Container to Container
Pod 內的容器共享同一網路，通常由 `pause`  基礎架構容器提供。彼此間可透過 lo 接口交互
2. Pod  to Pod 

3. Service to Pod 

4. WAN to Pod 