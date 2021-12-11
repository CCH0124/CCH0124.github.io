---
title: ML-3-meanshift
date: 2020-01-03
description: "非監督式學習 meanshift"
tags: [Machine Learning, meanshift, unsupervised]
draft: false
---

## What is meanshift

資料集的密度為一個隨核密度分佈，能夠在此資料集中找到局部極值，即為一個 `kernel density estimation`（它不需要預先知道樣本數據的概率密度分佈函數，完全能夠對樣本點的計算），因此將資料分群。

## Example

```python
import numpy as np
import matplotlib.pyplot as plt
import pandas as pd
from sklearn.cluster import MeanShift, estimate_bandwidth
from sklearn import datasets
from sklearn import preprocessing
#create datasets
# iris = datasets.load_iris() 
# X = iris.data[:, :4]
url = "https://raw.githubusercontent.com/uiuc-cse/data-fa14/gh-pages/data/iris.csv"
data = pd.read_csv(url)
le = preprocessing.LabelEncoder()
data['species'] = le.fit_transform(data.iloc[:,-1])

X = data.iloc[:, 0:4].to_numpy()
y = data.iloc[:,-1].to_numpy()

plt.scatter(X[:, 0], X[:, 1], c="yellow", marker='o', label='see')  
plt.show()

# estimate bandwidth
bandwidth = estimate_bandwidth(X, quantile=0.3)
# Mean Shift method
model = MeanShift(bandwidth = bandwidth, bin_seeding = True, max_iter=500)
model.fit(X)
labels = model.fit_predict(X)
centers = model.cluster_centers_

colormap = np.array(['Red','green','blue'])

plt.scatter(data.petal_length, data.petal_width, c=colormap[y], s=50)
plt.title('Classification Actually')
plt.show()

plt.scatter(data.petal_length, data.petal_width, c=colormap[model.labels_], s=50)
plt.title('Classification Prediction')
plt.show()
```

其中 `estimate bandwidth` 會影響 `Mean Shift`。當 `bandwidth` 越大，則峰就越平滑，越小則相反，這會導致集群分類的結果。

## Ref

- [ai-academy-taiwan](https://medium.com/ai-academy-taiwan/clustering-method-2-cd9bb883a0cb)
- [mean-shift-clustering](https://spin.atomicobject.com/2015/05/26/mean-shift-clustering/)
- [CSDN](https://blog.csdn.net/google19890102/article/details/51030884)