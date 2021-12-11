---
title: wireshark analysis Ann Skips Bail
date: 2020-05-18
description: "Puzzle #2 Solution Ann Skips Bail 解題"
tags: [wireshark, puzzle]
draft: false
---

[內容](http://forensicscontest.com/2009/10/10/puzzle-2-ann-skips-bail)

## 題目

1. What is Ann’s email address?
2. What is Ann’s email password?
3. What is Ann’s secret lover’s email address?
4. What two items did Ann tell her secret lover to bring?
5. What is the NAME of the attachment Ann sent to her secret lover?
6. What is the MD5sum of the attachment Ann sent to her secret lover?
7. In what CITY and COUNTRY is their rendez-vous point?
8. What is the MD5sum of the image embedded in the document?

### 1. What is Ann’s email address?

通常 mail 傳輸會使用 `SMTP` 協定，為了確定可透過 `statistics -> protocol Hierarchy` 查看該 pcap 流量的協定統計。

![](https://i.imgur.com/NugJbAg.png)

藉由 `smtp.auth.username` 的 filter 可得到經過編碼的 username `c25lYWt5ZzMza0Bhb2wuY29t` 經過解碼為 `sneakyg33k@aol.com`。

### 2. What is Ann’s email password?

透過 `smtp.auth.password` filter 即可知道，並經過解碼後得到 `558r00lz`

我們知道應用層的資訊會透過 socket 寫進傳輸層的緩存並將它封裝成 segment，在透過網路層傳輸至另一個端點。因此題目 1 與 2 從 `TCP Stream` 也可以去解析。藉由 `ip.addr == 192.168.1.159 && tcp` 進行過濾，查看過濾後第一個封包的 `TCP Stream` 如下圖

![](https://i.imgur.com/7bzhQeX.png)

### 3. What is Ann’s secret lover’s email address?

得先知道這所有封包中 Ann 寄過的人使用 `smtp.req.parameter contains "TO"` 方式或 `smtp.req.command contains "RCPT"` 過濾都可以看到，其 72 與 132 封包的收件人分別是 `sec558@gmail.com` 和`mistersecretx@aol.com`。

查看封包後 `mistersecretx@aol.com` 應該是答案。因為內容有這樣的內容

Hi sweetheart! Bring your fake passport and a bathing suit. Address =
attached. love, Ann

### 4. What two items did Ann tell her secret lover to bring?

從上提的封包內容提到 fake passport 和 bathing suit

### 5. What is the NAME of the attachment Ann sent to her secret lover?

從第 3 題封包可以看見 secretrendezvous.docx 的附件。

![](https://i.imgur.com/dvYirrm.png)

### 6. What is the MD5sum of the attachment Ann sent to her secret lover?

從上題已經知道傳輸檔案了，並從該封包中可發現 `Content-Transfer-Encoding: base64`，該檔案會使用 base64 編碼做處理。

以下是檔案做 base64 的編碼
```shell=
UEsDBBQABgAIA...
...
AA0ADQBEAwAA9CYDAAAA
```
將編碼的檔案還原，將第5題的封包使用 Raw 並載下，將檔案的編碼前後雜訊給刪除，留下該檔案的編碼。
```shell=
# 還原
$ openssl base64 -d < temp > secretrendezvous.docx
$ md5sum secretrendezvous.docx
9e423e11db88f01bbff81172839e1923  secretrendezvous.docx
```
`9e423e11db88f01bbff81172839e1923` 為該檔案 `MD5sum`

### 7. In what CITY and COUNTRY is their rendez-vous point?

上題將檔案還原，其內容為 `Meet me at the fountain near the rendezvous point. Address below. I’m bringing all the cash.`

並附上一個地圖，該地圖顯示的名稱就是答案 `Playa del Carmen`。


### 8. What is the MD5sum of the image embedded in the document?

使用 `unzip` 將 word 裡的內容解析。
```shell=
$ unzip secretrendezvous.docx
Archive:  secretrendezvous.docx
  inflating: [Content_Types].xml
  inflating: _rels/.rels
  inflating: word/_rels/document.xml.rels
  inflating: word/document.xml
 extracting: word/media/image1.png
  inflating: word/theme/theme1.xml
  inflating: word/settings.xml
  inflating: word/webSettings.xml
  inflating: word/styles.xml
  inflating: docProps/core.xml
  inflating: word/numbering.xml
  inflating: word/fontTable.xml
  inflating: docProps/app.xml
  
$ md5sum word/media/image1.png
aadeace50997b1ba24b09ac2ef1940b7  word/media/image1.png
```
`aadeace50997b1ba24b09ac2ef1940b7` 為答案

## 參考

- [SMTP filter](https://www.wireshark.org/docs/dfref/s/smtp.html)
- [SMTP use](https://blog.mailtrap.io/smtp-commands-and-responses/#DATA)
- [linux 取 word 圖片](https://www.opencli.com/linux/linux-get-docx-images)
