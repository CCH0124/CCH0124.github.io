---
title: 30 天學習歷程-day11
date: 2020-08-29
description: "特徵選取"
tags: [school, Machine Learning]
draft: false
---

特徵選取是指在開發一個機器學習模型時，減少輸入特徵數量的過程。這過程不但能減少計算上的成本，有時還能因為特徵選取減少了聲噪的影響因而建構出一個良好的模型。特徵選擇可分為以下

- Unsupervised
    - 移除多餘的特徵
    - Correlation
- Supervised
    - 移除無關連特徵
    - Wrapper
        - RFE
    - Filter
        - 依照特徵集合和目標的關係選擇特徵集合
        - Statistical 方法
            - SelectKBest
            - SelectPercentil
        - Feature Importance 方法
    - Intrinsic
        - 訓練過程中執行自動特徵選取的演算法
        - Decision Tree
- Dimensionality Reduction
    - 將數據投影到低維度的特徵空間中


## 統計的特徵選取方法
通常在輸入和輸出變量之間使用 `correlation` 統計作為過濾器特徵選擇的基礎。統計量測選擇高度依賴於可變數據類型，如下
- 數值
    - Integer 
    - Floating
- 分類
    - Boolean
    - Ordinal
    - Nominal

從數據類型來看的話數值是屬於 `Regression` 問題，分類是 `Classification` 問題。通常過濾器特徵選擇中使用的統計測量與目標變數一次計算一個輸入變數。因此，它們被稱為單變量統計(univariate statistical)測量。

以下是基於過濾器特徵選擇的單變量統計測量方法
![](https://3qeqpr26caki16dnhd19sv6by6v-wpengine.netdna-ssl.com/wp-content/uploads/2019/11/How-to-Choose-Feature-Selection-Methods-For-Machine-Learning.png) from https://machinelearningmastery.com/

### 數值輸入與數值輸出
- Pearson’s correlation coefficient (linear)
- Spearman’s rank coefficient (nonlinear)

### 數值輸入與分類輸出
- ANOVA correlation coefficient (linear).
- Kendall’s rank coefficient (nonlinear).

>Kendall 假設分類變數是 `Ordinal`

### 分類輸入與分類輸出
- Chi-Squared test (contingency tables).
- Mutual Information.

## scikit-learn 的特徵選取
在 `scikit-learn` 中提供了許多的[統計測量](https://scikit-learn.org/stable/modules/classes.html#module-sklearn.feature_selection)，如下

- Pearson’s Correlation Coefficient: `f_regression()`
- ANOVA: `f_classif()`
- Chi-Squared: `chi2()`
- Mutual Information: `mutual_info_classif()` and `mutual_info_regression()`