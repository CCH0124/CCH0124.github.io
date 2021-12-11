---
title: 30 天學習歷程-day02
date: 2020-08-18
description: "統計資料"
tags: [school, Machine Learning]
draft: false
---
# Statistical Data

透過統計方式我們可以觀察資料的趨勢或是分散程度等，以下將會介紹常用的統計方式。

## 數據集中趨勢
### Mean
- 一組數據中的平衡點
- 平均數是一組樣本和除以樣本數量

另 $x_1, x_2,..., x_N$ 為 $X$ 的 $N$ 個觀測值。這些值有可稱為**數據集**。這些數值均值(mean)為
$$\bar{x} = \frac{\sum_{i=1}^{N} x_i}{N} = \frac{x_1+x_2+...+x_N}{N}$$

有時可以與 $w_i$ 權重相關。此反應他們所依附的對應值的意義、重要性或出現頻率。可以下計算：
$$\bar{x} = \frac{\sum_{i=1}^{N} w_i x_i}{\sum_{i=1}^{N} w_i} = \frac{w_ix_1+w_2x_2+...+w_nx_N}{w_1+w_2+...+w_N}$$

這稱為**加權是算術平均值（weight arithmetic mean）**或**加權平均值（weighted average）**。

但上面計算的均值對極端值很敏感。要避免可以使用**截尾均值(trimmed mean)**，丟棄高低極端值得影響。在計算平均值之前，移除最底部 2% 資料等。

```python
import numpy as np
nums = [1,2,3,4,4,4,5,8,2,3]
# Numpy
np.mean(nums) # 3.6
# for
sum = 0
for i in nums:
    sum += i
print(sum/len(nums)) # 3.6
```

### Median
- 樣本需要是排序
- 代表一個樣本或概率分佈中的一個數值，可將數值集合劃分為相等的上下兩部分
- 樣本為偶數個，則中位數不唯一，通常取最中間的兩個數值的平均值作為中位數

對於 *偏斜（非對稱）* 數據，數據中心更好度量是**中位數(median)**，將較高或較低值給平均。
有序數據值的中間值。

當數據量很大時，中位數計算開銷很大，然而可以計算中位數的**近似值**。用內插(interpolation)計算該近似值 $$median = L_1 + (\frac{N/2+(\sum freq)_l}{freq_{median}})width$$

$L_1$ 是中位數區間下界，$N$ 是整個數據集中值的各數，$(\sum freq)_l$ 是低於中位數區間的所有區間的頻率，$freq_{median}$ 是中位數區間的頻率，$width$ 是中位數區間的寬度。


```python=
np.median(np.sort(nums)) # 3.5
```

### Mode
- 樣本中出現最多次的數值
- 其中有可能為多個數

眾數(mode) 是集合中出現最頻繁的值。因此，可對定性和定量屬性確定眾數。可能高頻率對應多個不同值，導致多個眾數，可分成
- 一個眾數稱為單峰(unimodel)
- 兩個眾數稱為雙峰(bimodal)
- 三個眾數稱為三峰(trimodal)

具有兩個或更多的資料集稱為**多峰(multimodal)**。

對適度偏斜(非對稱)的單峰資料，可得 $mean-mode \approx 3\times(mean-median)$，該平均值和中位數可以近似估計眾數。

```python
from scipy import stats
stats.mode(nums) # ModeResult(mode=array([4]), count=array([3]))
print("Value:", stats.mode(nums)[0])
print("Count:", stats.mode(nums)[1])
#Value: [4]
#Count: [3]
```

## 資料分散程度
### Range

- 全距 $R$，可稱為*極差*

- 用來表示統計資料中的變異量數

- 最大值與最小值的相減結果

```python
import numpy as np
nums = np.random.randint(80, size=20)
print(nums)
# [70 41 70 68 75 55 32  5  0 11 44 17 15 25 52 58 57 45 25 57]
```
```python=
np.ptp(nums) # 75
print(max(nums)-min(nums)) # 75
```

### Mid-range

- 中列數(midrange) 可以用來評估數值數據的中心趨勢。是數據集的最大和最小值的平均值。
- 在具有對稱的數據分布的單峰頻率曲線中，均值、中位數、和眾數都是相同的中心值，下圖

![](https://i.imgur.com/ZEQ8eXW.png) from "Data Mining. Concepts and Techniques, 3rd Edition"


### Quartile

分位數是資料分布上以固定區間取出的點，以下圖來說 `Q1`、`Q2` 和 `Q3`，這些資料點稱作分位數。而二分位數為則代表為*中位數(medain)*，將資料從中間頗一半。100 分位數則將資料切分 100 個大小相等的相連集合。

![](https://upload.wikimedia.org/wikipedia/commons/thumb/1/1a/Boxplot_vs_PDF.svg/800px-Boxplot_vs_PDF.svg.png) from wiki


在上圖中，`Q1` 和 `Q3` 的之間的距離，是測量分散程度的方式。該距離稱為*四分位間距(interquartile, IQR)*，數學定義為 $IQR = Q_3 - Q_1$


```python
import numpy as np
nums = np.random.randint(80, size=20)
print(nums)
nums.sort()
print(nums)
print(np.percentile(nums, (25))) 
print(np.percentile(nums, (50))) 
print(np.percentile(nums, (75)))
```

在視覺化章節將會使用圖的方式作呈現。

##### outlier

挑出在 `Q3` 上方 $1.5 \times IQR$ 資料值和 `Q1` 下方 $1.5 \times IQR$ 資料值，如上圖顯示。這邊先以此抽象方式進行簡易描述，後續的離群值章節會進行詳細介紹。

### five-number summary

它包含了中位數、四分位數的 `Q1` 和 `Q3`和最大與最小的觀察值。相比四分位數它多了最高和最低值的資訊。

### boxplot

是一種以視覺化顯示資料分布的方式，它涵括了*five-number summary*。從上圖的第一個方形形狀的圖呈現。



### variance and stander Deviation 

低標準差表示數據觀測趨向於非常靠近均值，高的話則散布在一個大的範圍中。

數值屬性 $X$ 的 $N$ 個觀測值 $x_1, x_2, ..., x_N$ 的變異數(variance)：
$$\sigma^2 = \frac{1}{N}\sum_{i=1}^{N}(x_i-\bar{x})^2 = (\frac{1}{N}\sum_{i=1}^{n}x_i^2)^2- \bar{x}^2$$

$\bar{x}$觀測的均值。觀測值的標準差(standar deviation) $\sigma$ 是變異數 $\sigma^2$ 的平方根。


例：我們得到$\bar{x}=58000$ 美元，決定此資料集的變異數和標準差。設置 $N=12$：$$\sigma^2=\frac{1}{12}\sum_{i=1}^{12}(30^2+36^2+47^2+...+110^2)-58^2 \approx 379.17$$
$$\sigma \approx \sqrt{379.17} \approx 19.14$$

標準差$\sigma$ 性質

- $\sigma$ 度量關於均值的發散，僅當平均值選為中心量測時才被考慮使用

- 僅當不存在發散時，即當所有的觀測值都具有相同值時，$\sigma=0$；否則大於 0

一個觀察值不大可能與平均值的距離有數倍的標準差，以 Chebyshev's Inequalit 解釋，至少 $(1-1/k^2)\times100%$ 的觀察值與平均值的距離不會超過 $k$ 倍的標準差。表準差是數據集發散得很好指示器。

```python
print(nums)
print(np.var(nums))
print(np.std(nums))
#[ 4 12 24 28 31 38 43 46 47 51 56 56 59 61 63 64 68 72 76 78]
#409.0275
#20.224428298471132
```

## 參考資料

- Data Mining. Concepts and Techniques, 3rd Edition