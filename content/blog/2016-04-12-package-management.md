---
title: "Ubuntu 上的 package 管理"
date: 2016-04-12
tags: ["Ubuntu"]
categories: ["Ubuntu"]
description: "就像是主管一樣"
draft: false
---

## ubuntu package 說明

- deb 為 ubuntu/debian 套件檔案的附檔名
    - 類似 redhat/centos 的 `rpm`
    - 是個包裹檔案，包含著`配置檔`、`執行檔`和`暫存檔`
- 命名格式
    - "套件名稱_版本_更新次數_適合的硬體平台".deb
        - postfix_2.11.0-1_i386.deb
- 硬體平台
    - i386
        - 適合 x86 平台
    - amd64
        - 適合 64 位元平台
    - noarch
        - 無任何硬體限制

## dpkg 操作
dpkg 為 debian PacKaGe 簡稱，為 debian/ubuntu 主要套件管理指令（安裝、升級、移除）

```shell
dpkg -c # 列出套件檔案的內容
dpkg -f # 列出套件的控制訊息
dpkg -l # 列出套件的詳細資訊
dpkg -L # 列出套件所安裝的檔案
dpkg -i # 安裝指定套件
dpkg -r # 移除套件
dpkg -P # 完全移除套件，不保留設定檔
```

```shell
dpkg-reconfigure # 重新設定套件安裝
-a # 重新設定所有使用 debconf 安裝的套件
-u # 預設會在畫面上顯示問題
-force # 強迫重新設定套件
```

## 轉換 rpm 與 deb 套件
```shell
alien # 轉換 rpm 與 deb 套件
--to-deb（-d）  # 產生 debian 套件
--to-rpm（-r）  # 產生 rpm 套件
--to-tgz（-t）  # 產生 tgz 套件
--test（-T）    # 測試 deb 套件
```

[套件安裝](https://cch0124.github.io/apt/)