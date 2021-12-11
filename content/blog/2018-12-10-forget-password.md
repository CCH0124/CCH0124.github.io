---
title: 前人沒留下資料，需要通靈
date: 2018-12-10
description: "ubuntu 忘記登入密碼"
tags: [Ubuntu, school]
draft: false
---

# unix 忘記密碼

1. 在開機時，按下 `Esc`。開啟 GRUB 開機選單
2. 按下鍵盤上的「e」鍵
3. 會看一個參數編輯的視窗
4. 找有 linux /boot/vmlinuz-X.XX.X 在最後一行加上 single 參數
5. 編輯好參數之後，按下 Ctrl + x 或是 F10 開機，接著就會進入單機模式。
6. 在單機模式之下，使用 passwd 更改一般使用者（或 root）的密碼
7. 最後執行 reboot 重新開機，就可以用新的密碼登入了。

## Ref

[gtwang](https://blog.gtwang.org/linux/linux-grub-change-root-password/)