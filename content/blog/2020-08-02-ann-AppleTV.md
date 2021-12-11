---
title: wireshark analysis Ann AppleTV
date: 2020-08-02
description: "Puzzle #3 Solution Ann AppleTV"
tags: [wireshark, puzzle]
draft: false
---

## 題目

[題目和內容](http://forensicscontest.com/2009/12/28/anns-appletv)，其中 Ann 的靜態 IP 是 192.168.1.10

1. What is the MAC address of Ann’s AppleTV?
2. What User-Agent string did Ann’s AppleTV use in HTTP requests?
3. What were Ann’s first four search terms on the AppleTV (all incremental searches count)?
4. What was the title of the first movie Ann clicked on?
5. What was the full URL to the movie trailer (defined by “preview-url”)?
6. What was the title of the second movie Ann clicked on?
7. What was the price to buy it (defined by “price-display”)?
8. What was the last full term Ann searched for?

### 1. What is the MAC address of Ann’s AppleTV ?
將 `View -> Name Resolution -> Resolve Physical Address` 打勾，其 wireshark 能夠將 MAC 做解析，從第二層鏈結層可以看到 MAC 地址對應的廠商，00:25:00 對應的是 Apple 廠商。

![](https://i.imgur.com/8fYyeD4.png)

`00:25:00:fe:07:c4`

### 2. What User-Agent string did Ann’s AppleTV use in HTTP requests?
使用 `http.user_agent` 過濾封包，再點選一個封包查看 HTTP 協定裡的協定，即可看到表頭。
`AppleTV/2.4`

### What were Ann’s first four search terms on the AppleTV (all incremental searches count)?
使用 `http.request.uri.query` 關鍵字過濾封包，並查看 `q` 關鍵字，可以得知在 AppleTV 上搜尋的第一個四個字的搜索詞為 `hack`。通常 `q` 關鍵都用於搜索相關。

![](https://i.imgur.com/c0hIIo6.png)

### What was the title of the first movie Ann clicked on?
接續上面的過濾結果其第 307 個封包顯示出請求 `viewMovie?id=333441649&s=143441`，其 `id` 是點擊看的電影，而 `s` 猜測是和 `X-Apple-Store-Front` 表頭有關其表是國家等資訊。但過濾的封包有點多，這邊使用 `http.request.uri contains viewMovie` 作為過濾的封包的條件，可以發現 Ann 應該是看了 2 部電影。

但是要如何知道它們電影名稱? 我們知道過濾後第 307 封包是請求封包，接著我們查看回應它的封包，該封包為第 312 個，其中它回應的是一個 XML，而從該 XML 可得知她看的為 Hackers。該 XML 包含了有關金額或者上映日期、可觀看年齡等資訊。

```xml=
 ...
 <key>title</key> 
       <string>Hackers</string> 
       <key>store-version</key> 
       <string>1.0</string> 
...
```

### What was the full URL to the movie trailer (defined by “preview-url”)?

接著我們將 312 封包使用 `http stream` 方式來查看，並尋找 `preview-url` 相關資訊。

```xml=
...
<key>preview-url</key><string>http://a227.v.phobos.apple.com/us/r1000/008/Video/62/bd/1b/mzm.plqacyqb..640x278.h264lc.d2.p.m4v</string>
...
```

###  What was the title of the second movie Ann clicked on?
使用和找第一個看電影的方式去完成。使用 `http.request.uri contains viewMovie` 關鍵字過濾，第 1181 個請求封包，接著我們找它的回應封包為第 1186 個封包。同樣使用 `http stream` 方式來查看，該第二個點擊的電影為 `Sneakers`。



### What was the price to buy it (defined by “price-display”)?

使用上一個 XML 查找。
SDVOD 為 2.99
STDQ 為 9.99


### What was the last full term Ann searched for?

最後一次搜尋我們使用 `http.request.uri.query.parameter contains q && http.request.uri contains "/WebObjects/MZSearch.woa/wa/incrementalSearch"` 過濾

![](https://i.imgur.com/x22uZP0.png)

這一個為最後一次搜尋。