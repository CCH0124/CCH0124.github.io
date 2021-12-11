---
title: 30 天學習歷程-day04
date: 2020-08-20
description: "資料的相異度與相似度"
tags: [school, Machine Learning]
draft: false
---

# 資料的相異性與相似性

在群集、離群值與最近鄰居等演算法應用中，需要比較兩個物件並評估兩物件之間的相似、相異性。

群集是指資料物件的集合，同一群集內的資料物件是彼此相似，不同群集間則是相異。離群值是辨識出那些物件與其它資料物件有著高度相異並將其認定為離群值。最近鄰居是對一個資料物件指定類別標籤，根據它與分類模型中其他物件的相似度決定。

## 資料矩陣和相異度矩陣

這邊的特徵向量將以二維或多維度屬性組成。

### 資料矩陣

使用 $n$ 筆物件乘上 $p$ 個屬性，來儲存 $n$ 筆資料物件。也稱*屬性結構。*

$\begin{bmatrix} 
    x_{11} & ... & x_{1f} & ... & x_{1p} \\
    ... & ... & ... & ... & ... \\
    x_{i1} & ... & x_{if} & ... & x_{ip} \\
    ... & ... & ... & ... & ... \\
    x_{n1} & ... & x_{nf} & ... & x_{np} \\
\end{bmatrix}$

### 相異度矩陣
儲存 $n$ 筆物件集合中，每一對物件的鄰近值，通常用一個 $n \times n$ 矩陣來表示。

$\begin{bmatrix} 
    0 \\
    d(2,1) & 0 \\
    d(3,1) & d(3,2) & 0\\
    . & . & . & .  \\
    . & . & . & .  \\
    . & . & . & .  \\
    d_{n, 1} & d(n, 2) & ... & ... & 0
\end{bmatrix}$

其中 $d(i,j)$ 代表物件 $i$ 與 $j$ 之間的相異度或差異度測量。其越接近 $0$ 表示越高相似度；越大則相反。

### 量測名目屬性資料的鄰近值

名目屬性狀態屬性為 $M$ 個，這些狀態可用文字、符號或整數表示，這些並沒任何特定順序。
如何計算由名目屬性所描述的資料物件之間的相異性呢? 兩個物件之間的相異度可根據他們的*匹配失敗率*計算，$m$ 式匹配成功數目（物件之間具有相同狀態的數目），$p$ 描述資料物件的屬性總數，其中 $m$ 可指定權重。

$$d(i,j)=\frac{p-m}{p}$$

*similarity* 計算：$$sim(i,j)=1-d(i,j)=\frac{m}{p}$$

##### Example
![](https://i.imgur.com/4bhmZQX.png)
![](https://i.imgur.com/aaHRbIr.png)

### 量測二元屬性資料的鄰近值

如何計算兩屬性之間的相異度呢 ?

![](https://i.imgur.com/YPincie.png)

上圖中，假設所有屬性都有*相同權重*，$p=q+r+s+t$。對於對稱二元屬性所算的相異度稱為**symmetric binary dissimilarity**。上圖假設 $i$、$j$ 為**symmetric binary dissimilarity**，則之間相異度為 $d(i,j)=\frac{r+s}{q+r+s+t}$。

在非對稱二元屬性，兩個狀態不是等同重要，基於此屬性計算的相異度稱為**asymmetric binary dissimilarity**，其中負匹配數量 $t$ 被視為不重要，在計算時*可省略*。

透過相異度計算相似度 $sim(i, j) = \frac{q}{q+r+s}=1-d(i,j)$(非對稱二元相似度)，其中 $sim(i,j)$ 也稱為 **Jaccard coefficient**。

##### Example
![](https://i.imgur.com/8Dq5Iig.png)
其中性名是識別碼，型別是對稱二元屬性，其餘都是非對稱二元屬性。

對於非對稱二元屬性，令狀態$Y$(yes)與$P$(陽性)被設定值為 1，而狀態 $N$(no 或是陰性)被設為 0，假設兩個病患之間的距離是以非對稱二元屬性為基礎，根據 $d(i,j)=\frac{r+s}{q+r+s}$，三個病患中，每一對之間的距離
![](https://i.imgur.com/xAaz6sY.png)

越接近 0 則相似度越高。

### 數值屬性資料的相異度

計算數值屬性所描述的資料間的相異度測量

- Euclidean distance

令 $i=(x_{i1},x_{i2},...,x_{ip})$ 和 $j=(x_{j1},x_{j2},...,x_{jp})$，之間距離為 $d(i,j)=\sqrt{(x_{i1}-x_{j1})^2 + (x_{i2}-x_{j2})^2 + ... + (x_{ip}-x_{jp})^2}$
- Manhattan distance or city block distance
$d(i,j)=|x_{i1}-x_{j1}| + |x_{i2}-x_{j2}| + ... + |x_{ip}-x_{jp}|$

- Minkowski distance

為 Euclidean distance 與 Manhattan distance 通式，$d(i,j)=\sqrt[h]{|x_{i1}-x_{j1}|^h + |x_{i2}-x_{j2}|^h + ... + |x_{ip}-x_{jp}|^h}$，其中 $h \ge 1$ 為實數，該距離也稱 $L_p$**範數(norm)**，其 $p$ 等同於 $h$，$h=1$ 表示 **Manhattan distance** 即 $L_1$ 範數，$h=2$ 即為 Euclidean distance 即 $L_2$ 範數。

- supremum distance 或 Chebyshev distance

也稱為 $L_\infty$(uniform norm) 或 $L_{max}$ 範數，也是 Minkowski distance 在 $L \rightarrow \infty$ 的推廣，計算時，要先找出某屬性 $f$，使得兩物件屬性之值的差距為最大的，該差距即為*supremum distance*，更正是定義為 $d(i,j)=lim_{h \rightarrow \infty} \lgroup \sum_{f=1}^{p}|x_{if} - x_{jf}|^h \rgroup^{\frac{1}{h}}=max^p_f|x_{if} - x_{jf}|$

- weighted Euclidean distance

增加權重值。$d(i,j)=\sqrt{w_1(x_{i1}-x_{j1})^2 + w_2(x_{i2}-x_{j2})^2 + ... + w_p(x_{ip}-x_{jp})^2}$

Euclidean distance 與 Manhattan distance 階滿足下列數學性值：

- 非負性：$d(i,j)>0$

- 同一性：$d(i,j)=0$

- 對稱性：$d(i,j)=d(j,i)$

- 三角不等式：$d(i,j)\le d(i,k)+ d(k,j)$，在空間

中從物件 $i$ 直線走到物件 $j$ 的距離，會小於透過物件 $k$ 繞道的距離


##### Example
![](https://i.imgur.com/VdzPSAr.png)


## 餘弦相似度

可用來比較檔案相似度的量測方法，令 $x$ 與 $y$ 為兩個要比較的向量，使用餘弦量測相似度。

$sim(x,y) = \frac{x \centerdot y}{\|x\| \|y\|}$


## 結論

透過使用相似度方式去量測資料之間是否有關聯，這對於像是商場上的行銷都是有很大的幫助，在 [scikit](https://scikit-learn.org/stable/search.html?q=distance) 這套件中有許多相似度量測方法。



## 參考資料

- 文章中圖片以及文字都摘要於 Data Mining. Concepts and Techniques, 3rd Edition