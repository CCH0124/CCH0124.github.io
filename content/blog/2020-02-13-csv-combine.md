---
title: csv combine
date: 2020-02-13
description: "多個 CSV 合併"
tags: [Machine Learning, python, awk]
draft: false
---


資料集有分為幾十種的種類，每一種類的資料大小都起碼 1G，由於深度學習時資料只有一個種類會不大理想，因此要將這些多個類型的檔案合併成一個。
我使用了 `pandas`，程式碼如下

```python
import pandas as pd
import os
import glob
csv_file_path = 'DATA_ROOT_PATH'
os.chdir(csv_file_path)
list_of_file = [file for file in blob.glob('DrDos_*.csv')] 
combined_csv = pd.concat([pd.read_csv(f) for f in list_of_file[:-1]])
combined_csv.to_csv('SAVE_PATH', index=True)
```

個人配置是 32GB，但在處裡 2 個 4G 檔案做合併時就會發生記憶體不足的問體。問題點可能是

1. 檔案真的太大
2. 該檔案裡的資料型態佔據的記憶體空間。因為有時資料只有 0 或 1，卻用 `int64` 來儲存，那豈不是佔據空間


之後嘗試用，shell `awk`和管道來處理，該機器有 22G 記憶體空間，但在執行時，看不出記憶體變化程度，這優化不知道是怎麼回事...。
之所以用 `awk` 是因為檔案中的 `header` 是一樣的，在合併時需要將該 `header` 省略，在配合著 `>>` 附加作用，即可完成。當然這種導向的輸出 `cat`、`paste` 等都可以做到，但因為沒省略 `header` 的方式，所以就沒用了。

```shell
awk 'FNR > 1' file.csv file2.csv >> output.csv
```

