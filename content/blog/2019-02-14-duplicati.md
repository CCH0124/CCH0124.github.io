---
title: Backup
date: 2019-02-14
description: "資料備份工具"
tags: [backup, duplicati, google drive]
draft: false
---

# duplicati
[duplicati](https://github.com/duplicati/duplicati) 是一個開源的備份軟體，可用在 `cloud storage` 服務或者 `file server`。
`duplicati` 可工作於 `WebDAV`、`Google Cloud Drive`、`MEGA`...等等。

最近會使用原因，因為社群有人寫[文章](http://blog.jason.tools/2019/02/pve-bak-conf.html?fbclid=IwAR14Eemed1oQVlMl1WKD6hfkjvoGKC8Im8ACnco7DrJpKOEBkgDkeeziIDs)分享，我有時也會備份資料去雲端但大多數是開著雲端網頁拉某資料夾的檔案到網頁。因為那篇文章我就嘗試運用 `duplicati` 做定期備份，這樣減少了我的干預。`Duplicati` 也提供強大的`加密`功能，同時備份檔案放在公共網路服務器上比在家中未加密的文件更安全。

## Install
以 ubuntu 為主機安裝 duplicati 此軟體。[載點](https://www.duplicati.com/download)
```shell=
$ wget https://updates.duplicati.com/beta/duplicati_2.0.4.5-1_all.deb
$ sudo dpkg -i duplicati_2.0.4.5-1_all.deb
(Reading database ... 99829 files and directories currently installed.)
Preparing to unpack duplicati_2.0.4.5-1_all.deb ...
Unpacking duplicati (2.0.4.5-1) over (2.0.4.5-1) ...
dpkg: dependency problems prevent configuration of duplicati:
 duplicati depends on mono-runtime (>= 3.0); however:
  Package mono-runtime is not installed.
 duplicati depends on libmono-2.0-1; however:
  Package libmono-2.0-1 is not installed.
 duplicati depends on libmono-system-core4.0-cil; however:
  Package libmono-system-core4.0-cil is not installed.
 duplicati depends on libmono-system-configuration4.0-cil; however:
  Package libmono-system-configuration4.0-cil is not installed.
 duplicati depends on libmono-system-configuration-install4.0-cil; however:
  Package libmono-system-configuration-install4.0-cil is not installed.
 duplicati depends on libmono-system-data4.0-cil; however:
  Package libmono-system-data4.0-cil is not installed.
 duplicati depends on libmono-system-drawing4.0-cil; however:
  Package libmono-system-drawing4.0-cil is not installed.
 duplicati depends on libmono-system-net4.0-cil; however:
  Package libmono-system-net4.0-cil is not installed.
 duplicati depends on libmono-system-net-htt
dpkg: error processing package duplicati (--install):
 dependency problems - leaving unconfigured
Processing triggers for mime-support (3.59ubuntu1) ...
Errors were encountered while processing:
 duplicati

```

在使用 `dpkg` 安裝時出現 Error。這 Error 要求要一些依賴軟體

```shell=
$ sudo apt-get install -f
```

利用下面指令確認
```shell=
$ sudo dpkg -l duplicati
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                                           Version                      Architecture                 Description
+++-==============================================-============================-============================-=================================================================================================
ii  duplicati                                      2.0.4.5-1                    all                          Backup client for encrypted online backups

```

## Start service
```shell=
$ sudo systemctl start duplicati.service
lab702@elk:~$  systemctl status duplicati
● duplicati.service - Duplicati web-server
   Loaded: loaded (/lib/systemd/system/duplicati.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2019-02-13 13:12:28 CST; 25s ago
 Main PID: 74617 (mono)
    Tasks: 17
   Memory: 30.6M
      CPU: 742ms
   CGroup: /system.slice/duplicati.service
           ├─74617 DuplicatiServer /usr/lib/duplicati/Duplicati.Server.exe
           └─74628 /usr/bin/mono-sgen /usr/lib/duplicati/Duplicati.Server.exe

Feb 13 13:12:28 elk systemd[1]: Started Duplicati web-server.
lab702@elk:~$ sudo systemctl enable duplicati.service
Created symlink from /etc/systemd/system/multi-user.target.wants/duplicati.service to /lib/systemd/system/duplicati.service.
```

### Defaul port
`duplicati` 預設 port 為 `8200`
```shell=
$ netstat -ltn | grep 8200
tcp        0      0 127.0.0.1:8200          0.0.0.0:*               LISTEN
```

## Start duplicati
可用以下兩種方式啟動
1. 
```shell=
$ duplicati


(Duplicati.GUI.TrayIcon:97053): libappindicator-WARNING **: Unable to get the session bus: Failed to execute child process "dbus-launch" (No such file or directory)

(Duplicati.GUI.TrayIcon:97053): LIBDBUSMENU-GLIB-WARNING **: Unable to get session bus: Failed to execute child process "dbus-launch" (No such file or directory)

```
2. 
```shell=
$ sudo duplicati-server --webservice-interface=192.168.222.128
$ sudo duplicati-server --webservice-interface=192.168.222.128 & # 背景執行
```
192.168.222.128 為 Host IP address

## Remove duplicati
```shell=
$ sudo dpkg -r duplicati
```
## Duplicati backup data to google drive
1. 
![](https://i.imgur.com/aUY5EkG.png)

2. 
![](https://i.imgur.com/ET42IXt.png)

3. 
![](https://i.imgur.com/8OsC6PI.png)

4. 點選 AuthID 登入 google 服務即可獲得 AuthID
![](https://i.imgur.com/rzOuPgX.png)

5. 勾選要備份到雲端的資料
![](https://i.imgur.com/L7kGxVY.png)

6. 
![](https://i.imgur.com/fg6UIHk.png)

7. 
![](https://i.imgur.com/B9hchxT.png)

8.  首頁會出現剛設定的 backup 
![](https://i.imgur.com/fiJtj07.png)

9. 查看雲端
![](https://i.imgur.com/QOxQh2P.png)


其中在第二步驟輸入的密碼是在還原時，要用到的。

### Data restore
1. 
![](https://i.imgur.com/cz0MMVN.png)

2. 
![](https://i.imgur.com/QbYQ1nO.png)

3. 輸入當備份時所加密的密碼
![](https://i.imgur.com/hWSLlPX.png)

4. 會列出當時您備份的資料，將要還原的資料打勾
![](https://i.imgur.com/RPIlk9a.png)

5. 設定還原時選項，依照個人需求設定
- 您要還原檔案到哪裡？
    - 可以自己選擇新路徑或原本位置
- 您如何處理既有檔案？
    - 覆寫
    - 在檔案名稱中儲存不同版本的時間戳記
- 權限
    - 還原讀/寫權限

6. 成功畫面
![](https://i.imgur.com/FVKWw3i.png)


## Conclusion
我是在 windows 上使用 `duplicati` 做備份，`duplicati` 可以幫備份資料做加密提高了安全性，又可以上傳至 `cloud` 可達成異地備份的任務，是一個很不錯的開原軟體。但有很多功能還在摸索，因此此文章只是描述我如何備份至雲端。

我在學校有管理系統，因此正在研究如何將系統的備份資料透過 `duplicati` 上傳至 NAS。這部分我成功後會修補文章。
## Reference
[github](https://github.com/duplicati/duplicati)

[install-duplicati-ubuntu-server](https://manjaro.site/install-duplicati-ubuntu-server/)

[jasoncheng](https://www.slideshare.net/jasoncheng7115/duplicati-20170521-hexbase)