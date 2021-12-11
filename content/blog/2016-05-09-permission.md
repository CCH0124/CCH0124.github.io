---
title: "permission setting"
date: 2016-05-09
tags: ["Ubuntu"]
categories: ["Ubuntu"]
description: "原來要這樣管理國家大事"
draft: false
---

# 何謂權限
保證該使用者、部門的有效履行。必須具備對某事項進行決策的范圍和程度。不論在哪裡，都會有這種控管，如：學生對老師；老師對主任甚至校長，這過程不是一個人說的算，總不能學生說我要下課，就可以下課。這當然要往上一個層級詢問，直到擁有該權限的人才說的算。這樣子的控管會有一定的保護。

## 基本權限認知

![](https://i.imgur.com/n9SuMQ9.png)

## 了解基本權限
三種身分
- owner
- group
- other

三種權限
- read
- write
- execute

![](https://i.imgur.com/scatrEO.png)

```shell
-rw-rw-r-- 1 cch  cch   198 Aug 19 17:37 podman.sh
# r 讀
# w 寫
# x 執行
# _ 無權限
```

## 練習

![](https://i.imgur.com/i6i5ym9.png)

penguin 來說 fred 帳號可讀寫執行，但對有 group 為 fred 的帳號少了寫的權限，非該帳號和 group 的帳號則只能讀。
redhat 來說 mary 帳號可讀寫，對有 group 為 admin 的帳號只有讀，非該帳號和 group 的帳號則只能讀。
tuxedo 帳號 root 可讀寫，對有 group 為 staff 的帳號只有讀寫，非該帳號和 group 的帳號則只能讀。

## 變更帳號的擁有者和群組

- chown
    - 變更擁有者
- chgro
    - 變更群組

## 權限變更
- chmod
    - 權限變更

```shell
$ chmod u=rwx,g=rx,o=x filename
# u 使用者
# g 群組
# o 其他人
# a 所有
$ chmod 751 filename

```