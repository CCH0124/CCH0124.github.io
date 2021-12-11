---
title: supervised vs unsupervised Learning
date: 2019-12-31
description: "監督與非監督比較"
tags: [Machine Learning]
draft: false
---

## What is supervised ?
有存在正確答案的的數據，簡而言之就是有"Label"。監督式學習從有 Label 的數據學習建立出可預測的模型。例如：天氣狀況、物件辨識等。

流程：
```
Training Data -> Features Selection -> Algorithm -> Model
```

### Type
- Regression
    - 預測一個數值
- Classification
    - 將輸出分組到一個類中

![](https://miro.medium.com/max/798/1*4sixxtuD8unWceZ-yp9TgQ.jpeg)
from : [http://www.slideshare.net/datascienceth/machine-learning-in-image-processing]

## What is unsupervised ?
從無 Label 中的數據，自行找出資料的結構建立模型。相比監督式學習，無監督更加不可預測。

### Type
- Clustering
    - 從數據中找出結構或模式，並將其自然地作出聚類
- Association
    - 在大型的數據資料中找出數據對象之間的關聯
    - 例子
        - 購物的人瀏覽和購買物品的組合

![](https://miro.medium.com/max/1200/1*VACikYaZIHb2OctTRohn8A.jpeg)
from: [https://medium.com/data-science-by-heart/types-of-learning-93b721d5af91]
