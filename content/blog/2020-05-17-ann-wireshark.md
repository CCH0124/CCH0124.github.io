---
title: wireshark analysis Ann’s Bad AIM
date: 2020-05-17
description: "Puzzle #1 Solution: Ann’s Bad AIM 解題"
tags: [wireshark, puzzle]
draft: false
---

[題目網站](http://forensicscontest.com/2009/09/25/puzzle-1-anns-bad-aim)

Ann’s computer, (192.168.1.158) sent IMs over the wireless network to this computer. 這一句話可發現 Ann 的 IP 為 `192.168.1.158`，和使用的應用程式。

該應用程式為[IM 協定](https://zh.wikipedia.org/wiki/%E5%8D%B3%E6%99%82%E9%80%9A%E8%A8%8A)，`tcp port` 為 `5190`

## 題目

1. What is the name of Ann’s IM buddy?
2. What was the first comment in the captured IM conversation?
3. What is the name of the file Ann transferred?
4. What is the magic number of the file you want to extract (first four bytes)?
5. What was the MD5sum of the file?
6. What is the secret recipe?

### 1. What is the name of Ann’s IM buddy?

```shell=
ip.dst == 192.168.1.158
```

透過上面指令將 `192.168.1.158` 先過濾，發現協定有 `SSL`、`TCP` 等，其中 `SSL` 和 `TCP` 較重要，因此可透過 `stream` 方式查看。但由於使用的是非標準端口，該流量 `TCP` 的 `port` 為 `443` 有誤認的情形，須使用`Decode As`，將當前協定修正為 `AIM`，`Protocol` 欄位會將 `AIM` 解析出來，

![](https://i.imgur.com/lqhvi3X.png)

其中 ANN Buddy Name 是 `sec558user1`

### What was the first comment in the captured IM conversation?

先前透過 IP 過濾出封包，因此過濾出來的第一個 `AIM` 協定，即可得知。

![](https://i.imgur.com/wcbYKt7.png)

`ValueMessage: Here's the secret recipe... I just downloaded it from the file server. Just copy to a thumb drive and you're good to go &gt;:-) `

### 3. What is the name of the file Ann transferred?

先猜想是否用 `AIM` 方式傳遞檔案，畢竟 `AIM` 可傳遞檔案。

```shell=
ip.addr == 192.168.1.158 && tcp.port == 5190

```
上述方式過濾出 `5190 port` 封包。用 `TCP stream` 查看，發現 `recipe.docx` 檔案。

### 4. What is the magic number of the file you want to extract (first four bytes)?

透過上面找到的封包，使用 TCP stream 觀看。並 GOOGLE 找一下 DOCX 的魔術數字...

![](https://i.imgur.com/DVYqw8b.png)

使用 Hex 查看 ![](https://i.imgur.com/Oy9nIiE.png)

`pk` 為關鍵字，也就是 `50 4B 03 04`，

- [魔術數字](https://www.garykessler.net/library/file_sigs.html)

### 5. What was the MD5sum of the file?

docx 的標頭分別為 `50 4B 03 04` 或 `50 4B 03 04 14 00 06 00`。
接著開啟 `TCP Stream` 並將編碼轉成 `Raw`，並將其下載並命名`recipe.docx`，並計算該檔案的 `hash`

```shell=
$ md5sum recipe.docx
52c13d8c0a99ac0d3210e8e8edb046bf *recipe.docx
```

`52c13d8c0a99ac0d3210e8e8edb046bf` 為該檔案的 `hash` 值

### 6. What is the secret recipe?

透過上提下載的檔案，打開後裡面內容為

![](https://i.imgur.com/KEIhIeT.png)

##### 補充

`stream` 功能從多個封包裡，把分散的資料在重整一致、容易判讀的格式，稱之封包文本（packet transcript），這樣不必在用戶端與伺服器之間往來的封包片段中檢閱。

- TCP stream
    - 從使用 TCP 的協定中把資料組合起來
- SSL stream
    - 從已加密的協定中把資料組合起來
    - 需有密鑰才能解
- 查看時，紅色是送出內容，藍色是接收內容

## 參考

- [AIM](https://wiki.wireshark.org/AIM)
- [TCP 和 SSL](https://crypto.stackexchange.com/questions/53786/why-is-ssl-on-top-of-tcp)
