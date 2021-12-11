---
title: 人所習慣的視覺化
date: 2018-01-24
description: "shell 環境下文字和背景顏色設定"
tags: [Ubuntu]
draft: false
---
## Visualize

視覺化能夠增強人類感知，將一些資料中的重要資訊用特別的顏色顯示，有助於提供將重要資訊顯示。

生活中日歷上特殊節日也會以紅色顯示，如國歷 10/10 國慶日、農曆 8/15 中秋節等等。

如果再 ubuntu 文字介面下沒有特別突出的顏色應該是不會去特別注意，像是安裝套件有錯誤或警告幾乎以特別的顏色顯示，以吸引或引導的方式幫助我們解決錯誤。

## Matching styles to numbers
- 樣式
  - 0 一般樣式
  - 1 粗體
  - 4 加底線
  - 5 灰底
  - 7 文字及背景顏色對調

- 文字顏色
  - 30 黑色
  - 31 紅色
  - 32 綠色
  - 33 黃色
  - 34 藍色
  - 35 紫色
  - 36 青綠
  - 37 白色

- 背景顏色
  - 40 黑色
  - 41 紅色
  - 42 綠色
  - 43 黃色
  - 44 藍色
  - 45 紫色
  - 46 青綠
  - 47 白色

## Example

~~~ shell
#!/bin/bash
echo "\033[1m Hello World"
# bold effect
echo "\033[5m Hello World"
# blink effect
echo "\033[0m Hello World"
# back to noraml
echo "\033[31m Hello World"
# Red color
echo "\033[32m Hello World"
# Green color
echo "\033[33m Hello World"
# See remaing on screen
echo "\033[34m Hello World"
echo "\033[35m Hello World"
echo "\033[36m Hello World"
echo -n "\033[0m"
# back to noraml
echo "\033[41m Hello World"
echo "\033[42m Hello World"
echo "\033[43m Hello World"
echo "\033[44m Hello World"
echo "\033[45m Hello World"
echo "\033[46m Hello World"
echo "\033[0m Hello World"
~~~

## Reference
[Linux 技術手札](https://www.phpini.com/linux/shell-script-color-text)