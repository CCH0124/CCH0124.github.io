---
title: How exclude file ?
date: 2019-12-12
description: "rm 沒有 exclude 的功能"
tags: [Ubuntu]
draft: false
---

## 使用方式

```shell
find . -type f | grep -v ".json" | xargs rm
```

透過 `type` 可指定檔案（f）或是目錄（d），在藉由 `grep` 的 `-v` 反向選取，之後再驅動 `rm`。如果想要多個條件的話要藉由 `grep` 配合規則表示。