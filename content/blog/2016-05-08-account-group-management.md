---
title: "Ubuntu 上的帳號和群組管理"
date: 2016-05-08
tags: ["Ubuntu"]
categories: ["Ubuntu"]
description: "要如何分配這些部門的人呢..."
draft: false
---
## 帳號命名原則

管理者總是讓帳號有一定的統一規則。如：
- 魯夫就 roof 或 luff
- 組織單位的分層 itachi_jp
- 區域分層 zoro_mis
等。

這些都是為了維護和管理阿!!

## Account 帳號
在 ubuntu 中，每個使用者都有獨立的 UID（ID Number），系統對帳號的識別以 UID 為準，**帳號的名稱僅用在登錄系統上**。UID 就像身分證一樣必須唯一（UID 0 以外），有著一對一關係。

- UID 為 0 表示為最高權限（root）。
- 服務帳號會自動建立，**UID 通常介於 1~99**
- 手動定義 UID 建議以 1000 開始

```shell
# 系統的設定
$ grep UID /etc/login.defs
UID_MIN                  1000
UID_MAX                 60000
#SYS_UID_MIN              100
#SYS_UID_MAX              999
```
- 使用者的名稱和 UID 存在 `/etc/passwd`
- 使用者會有**家目錄**
- 該使用者無適當的權限，則無法讀寫甚至執行檔案

## Group 群組
帳號與群組類似，簡稱 GID（Group ID）。

- GID 和群組名稱必須唯一（GID 0 例外）
- GID 儲存在 `/etc/passwd` 第四欄位；`/etc/group` 是第三欄位

```shell
$ cat /etc/passwd | tail -n 5
cch:x:1000:1000:cch,,,:/home/cch:/bin/bash
naruto:x:1001:1001:Test Account:/home/naruto:/bin/bash
itahi:x:1002:1002:Test Account:/home/itahi:/bin/bash
madara:x:1003:1004::/home/madara:/sbin/nologin
brouto:x:1004:1006::/home/brouto:/sbin/nologin
```
```
$ cat /etc/group | tail -n 5
itahi:x:1002:
ftpgroup:x:1003:naruto,madara
madara:x:1004:
ftpgroup2:x:1005:brouto
brouto:x:1006:naruto
```

## root 
一定是存在系統的，權限為最高的帳號。在 ubuntu 不允許預設 `root` 登入，須以一般帳號登入再提權。`root` 的 `UID` 為 `0`，帳號 `UID` 改為 `0`，則權限與 root 是畫上等號的。

```shell
$ id root
uid=0(root) gid=0(root) groups=0(root)
```

## 帳號管理相關目錄與介紹
```shell
$ /etc/passwd # 紀錄系統帳號資訊
$ /etc/group # 紀錄群組帳號的資訊檔
$ /etc/shadow # 記錄密碼相關的資訊紀錄
```
### passwd 檔案
格式為 `帳號:密碼:UID:GID:註解:家目錄:shell`

- 帳號
    - 系統帳號名稱
- 密碼
    - hash 過的欄位，在 `shadow` 檔為欄位 `x`
- UID
    - 帳號識別碼
- GID
    - 群組碼，對應於 group 檔案
- 註解
    - 備註或介紹等資訊
- 家目錄
    - 帳號家目錄的位置
- shell
    - 登入時所使用的指令有以下
        - sh
        - csh
        - bash
        - zsh
        - /sbin/nologin 無法登入，可應用在 `sftp`

### group 檔案
格式 `群組名稱:密碼:GID:帳號列表`。可用 `groupadd`、`gpasswd` 或 `groupdel` 處裡該檔案，並不會直接修改。

- 群組名稱
    - 取的名字
- 密碼
    - hash 過的欄位，也是以 `x` 顯示
- GID
    - 群組的 ID
- 帳號列表
    - 列出該群組的使用者

### shadow 檔案
![](https://i.imgur.com/5GKMako.png)
- 帳號 
    - 登入系統的名稱
- 加密密碼
    - 第一字元為 `!`，表示已經被鎖定，該帳號無法登入
- 上次密碼變更日期
    - 0 表示下次登入要更改密碼
- 密碼最短使用期限
    - 0 表示關閉此功能
- 密碼最長使用期限
    - 為空值表示關閉此功能
- 密碼失效前多少日提醒警告
    - 0 表示關閉此功能
- 密碼失效前多少日停用帳號
    - 密碼過期，還有幾天寬限時間更改
    - 為空值表示關閉此功能
- 帳號過期時間
    - 為空值表示不會過期

## 帳號和群組相關指令
- passwd
    - 設定密碼
![](https://i.imgur.com/1z9DOBn.png)
- useradd
    - 新增帳號
![](https://i.imgur.com/XNsgPE3.png)
![](https://i.imgur.com/zt5oudF.png)
```shell
# 範例
$ sudo useradd -d /home/roof -u 5000 -s /bin/bash roof
$ id roof
uid=5000(roof) gid=5000(roof) groups=5000(roof)
```
- usermod
    - 修改帳號資訊
![](https://i.imgur.com/NQozKCs.png)
- userdel
    - 刪除帳號
```shell
$ sudo userdel roof # 加上 -r 參數表示連同家目錄一並刪除
$ id roof
id: ‘roof’: no such user
```
- groupadd
    - 新增群組
- gpasswd
    - 管理群組成員
- groupdel
    - 刪除群組

```shell
$ sudo useradd student100
$ sudo useradd student101
$ sudo useradd student102
$ sudo groupadd -g 800 stustaff # -g 給予 GID
$ sudo groupadd -g 900 ta
$ sudo gpasswd -a student100 stustaff # -a 附加要新增的帳號到群組 stustaff；-d 則表示刪除
Adding user student100 to group stustaff
 sudo gpasswd -a student101 ta
Adding user student101 to group ta
$ sudo gpasswd -a student102 ta
Adding user student102 to group ta
$ sudo cat /etc/group | grep -E "(stustaff)|(ta)"
tape:x:26:
www-data:x:33:
staff:x:50:
crontab:x:107:
itahi:x:1002:
stustaff:x:800:student100 ##
ta:x:900:student101,student102 ##
$ id student101
uid=1006(student101) gid=1008(student101) groups=1008(student101),900(ta)
$ sudo groupdel ta # 刪除群組
$ id student101
uid=1006(student101) gid=1008(student101) groups=1008(student101)
$ sudo groupdel stustaff
```

- chage 
    - 設定密碼
![](https://i.imgur.com/uy1NA0V.png)

## 帳號預設設定
- 會從 `/etc/default/useradd` 與 `/etc/login.defs` 的檔案做處裡
- 將 `/etc/skel/.bashrc` 複製到帳號的根目錄，因此帳號會有 `.bash_profile` 和 `.bashrc` 檔案

### useradd 檔案基本的設定值
```shell
GROUP=500 # 帳號預設 GID
HOME=/home # 目錄位置
INACTIVE=-1 # 是否停用，-1 表啟用
EXPIRE= # 失效日期
SHELL=/bin/sh # 使用 shell
SKEL=/etc/skel # 套用 SKEL 設定值
```

### login.defs 帳號詳細資訊
```shell
MAIL_DIR                /var/main # 帳號郵件存放目錄
MAIL_FILE               .mail   
FAILLOG_ENAB            yes # 紀錄和顯示 /var/log/faillog 紀錄資訊
LOG_UNKFAIL_ENAB        no # 登入錯誤被記錄時，允許顯示不知名帳戶
LOG_OK_LOGINS           no # 開啟正確登入紀錄
SYSLOG_SU_ENAB          yes # 開啟切換用戶紀錄
SYSLOG_SG_ENAB          yes # 開啟切換用戶紀錄
...
PASS_MAX_DAYS           99999 # 密碼可使用天數
PASS_MIN_DAYS           0   # 密碼多少日不得變更
PASS_WARN_AGE           7   # 密碼失效前多少日提供警告
...
```

## 結論
透過 `useradd` 新增使用者，透過 `grouadd` 新增使用者至某個群組，這樣算是可以較清楚的管理系統的使用者了。當然這樣還不夠，適當地將這些 UID、GID 應用在不論是檔案權限的劃分或者服務使用的權限這才是最好的配置。

以上是從學校學到的資訊。如有問題可留言回覆。