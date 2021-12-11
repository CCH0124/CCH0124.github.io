---
title: "使用者操作 - Ubuntu"
date: 2015-08-11
tags: ["Ubuntu"]
categories: ["Ubuntu"]
description: "確實做好使用者管理"
draft: false
---
## useradd 選項

|參數 | 說明|
|--- | --- |
|-c 註解 | 指定帳號的註解文字|
|-g 主要群組名稱或 GID | 指定帳號的主要群組|
|-G 附加群組名稱或 GID | 指定帳號所屬的附加群組|
|-d 目錄 | 指定家目錄|
|-e 日期 | 指定新建帳號的使用到期日，過期該帳號無法使用|
|-u UID | 指定帳號的 UID 編號|

## 新增使用者

``` shell
# useradd testuser
# id testuser # id 列出使用者資訊
uid=1002(testuser) gid=1003(testuser) groups=1003(testuser)
```

### 指定群組

``` shell
# useradd -g rd eric # -g 指定群組
# id eric
uid=1003(eric) gid=1001(rd) groups=1001(rd)
```

## 移除使用者

``` shell
# userdel USER #無法刪除該使用者家目錄
# userdel -r USER #刪除使用者和家目錄
```

## usermod 選項

|參數 | 說明|
|--- | ---|
|-c 註解 | 改變帳號的註解文字|
|-g 主要群組名稱或 GID | 改變帳號的主要群組|
|-G 附加群組名稱或 GID | 改變或增加帳號所屬的附加群組|
|-d 目錄 | 改變家目錄|
|-e 日期 | 變更帳號的使用到期日，過期該帳號無法使用|
|-u UID | 變更帳號的 UID 編號|
|-l 帳號名稱 | 變更原帳號的名稱|

## 修改使用者帳號

``` shell
# id tom
uid=1004(tom) gid=1001(rd) groups=1001(rd),1004(manager)
# usermod -G manager,sales tom # usermod 修改使用者帳號， -G 加入附屬群組(已存在的帳號)
# id tom
uid=1004(tom) gid=1001(rd) groups=1001(rd),1004(manager),1005(sales)
```