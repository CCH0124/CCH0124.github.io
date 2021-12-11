---
title: 30 天學習歷程-day06
date: 2020-08-22
description: "資料清理應用"
tags: [school, Machine Learning]
draft: false
---

## Data cleaning

將使用 `pandas` 工具和 `CICDDOS` 數據集進行資料清理的練習。下面會學習到 `pandas` 的使用。

###　Drop Miss Value
```python
import pandas as pd
import numpy as np
ddos = pd.read_csv("D:\\DataSet\\CICDDOS2019\\01-12\\CSV\\DrDoS_LDAP.csv")
missing_values_count = ddos.isnull().sum() # 檢查缺失值
missing_values_count[20:25]
##############################
#Bwd Packet Length Std     0
#Flow Bytes/s              12
# Flow Packets/s            0
# Flow IAT Mean             0
# Flow IAT Std              0
#dtype: int64
##############################3
# Flow Bytes/s 有 12 個缺失值
ddos.shape
# (2181542, 88)

# 計算缺失值比例
total_miss_values = missing_values_count.sum()
total_cells = np.product(ddos.shape)
print(total_miss_values)
print(total_cells)
print((total_miss_values/total_cells) * 100)

#12
#191975696
#6.250791245991888e-06

columns_with_na_dropped = ddos.dropna(axis=1) # 刪除缺失值欄位
columns_with_na_dropped.shape
# (2181542, 87) Flow Bytes/s 被刪除
```

### Filling in missing values automatically

由於數據集無 `NaN` 值，因此無實作。但這邊提供方法，在用 pandas 匯入檔案時，當有發現 `NaN` 值時，可以用 pandas 中 `fillna` 的方式去填補 `NaN` 值。該方法有提供以下方式填補 `backfill`、`bfill`、`pad` 和 `ffill`。而官網有提供[範例](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.fillna.html)。


## Scaling and Normalization

### min-max

使用 [scikit-learn 套件實作](https://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.MinMaxScaler.html)

```python
ddos = pd.read_csv("D:\\DataSet\\CICDDOS2019\\01-12\\CSV\\DrDoS_LDAP.csv")
data_temp = ddos.iloc[:3, 8:11]
data_temp2 = ddos.iloc[:8, 8:11]
data_temp.values

#array([[9141643,   85894,      28],
#       [      1,       2,       0],
#       [      2,       2,       0]], dtype=int64)

data_temp2.values
#array([[9141643,   85894,      28],
#       [      1,       2,       0],
#       [      2,       2,       0],
#       [      1,       2,       0],
#       [      2,       2,       0],
#       [      2,       2,       0],
#       [      2,       2,       0],
#       [      2,       2,       0]], dtype=int64)

from sklearn.preprocessing import MinMaxScaler
scaler = MinMaxScaler()
scaler.fit(data_temp.values)

print(scaler.data_max_) # [9.141643e+06 8.589400e+04 2.800000e+01]
print(scaler.data_min_) # [1. 2. 0.]

print(scaler.transform(data_temp2.values))
'''
[[1.00000000e+00 1.00000000e+00 1.00000000e+00]
 [0.00000000e+00 0.00000000e+00 0.00000000e+00]
 [1.09389539e-07 0.00000000e+00 0.00000000e+00]
 [0.00000000e+00 0.00000000e+00 0.00000000e+00]
 [1.09389539e-07 0.00000000e+00 0.00000000e+00]
 [1.09389539e-07 0.00000000e+00 0.00000000e+00]
 [1.09389539e-07 0.00000000e+00 0.00000000e+00]
 [1.09389539e-07 0.00000000e+00 0.00000000e+00]]
'''
```
### StandardScaler

使用 [scikit-learn 套件實作](https://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.StandardScaler.html)

```python
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler()
scaler.fit(data_temp.values)

print(scaler.mean_) # [3.04721533e+06 2.86326667e+04 9.33333333e+00]
print(scaler.var_) # [1.85710243e+13 1.63943015e+09 1.74222222e+02]

print(scaler.transform(data_temp2.values))
'''
[[ 1.41421356  1.41421356  1.41421356]
 [-0.7071069  -0.70710678 -0.70710678]
 [-0.70710667 -0.70710678 -0.70710678]
 [-0.7071069  -0.70710678 -0.70710678]
 [-0.70710667 -0.70710678 -0.70710678]
 [-0.70710667 -0.70710678 -0.70710678]
 [-0.70710667 -0.70710678 -0.70710678]
 [-0.70710667 -0.70710678 -0.70710678]]
'''
```

scikit-learn 提供了多種數值縮放的方式，如下

- sklearn.preprocessing.MaxAbsScaler
- sklearn.preprocessing.MinMaxScaler
- sklearn.preprocessing.RobustScaler
- sklearn.preprocessing.StandardScaler
- ...

## Data Parse

```python
ddos[' Timestamp']
'''
0          2018-12-01 11:22:40.254769
1          2018-12-01 11:22:40.255361
2          2018-12-01 11:22:40.255568
3          2018-12-01 11:22:40.256113
4          2018-12-01 11:22:40.256285
                      ...            
2181537    2018-12-01 11:32:32.914224
2181538    2018-12-01 11:32:32.914273
2181539    2018-12-01 11:32:32.914438
2181540    2018-12-01 11:32:32.915067
2181541    2018-12-01 11:32:32.915361
Name:  Timestamp, Length: 2181542, dtype: object
'''
ddos['date_parsed'] = pd.to_datetime(ddos[' Timestamp'], format="%Y-%m-%d %H:%M:%S.%f") # parse

ddos['date_parsed'].dt.day[:3]
'''
0    1
1    1
2    1
Name: date_parsed, dtype: int64
'''
print(ddos['date_parsed'].dt.month[:3])
'''
0    12
1    12
2    12
Name: date_parsed, dtype: int64
'''
```

pandas 能夠對日期進行一些的計算，如相差幾天等。pandas 對於數據的處裡提供很多強大的 API 使用。


`format` 補充

```
%a  Locale’s abbreviated weekday name.
%A  Locale’s full weekday name.      
%b  Locale’s abbreviated month name.     
%B  Locale’s full month name.
%c  Locale’s appropriate date and time representation.   
%d  Day of the month as a decimal number [01,31].    
%f  Microsecond as a decimal number [0,999999], zero-padded on the left
%H  Hour (24-hour clock) as a decimal number [00,23].    
%I  Hour (12-hour clock) as a decimal number [01,12].    
%j  Day of the year as a decimal number [001,366].   
%m  Month as a decimal number [01,12].   
%M  Minute as a decimal number [00,59].      
%p  Locale’s equivalent of either AM or PM.
%S  Second as a decimal number [00,61].
%U  Week number of the year (Sunday as the first day of the week)
%w  Weekday as a decimal number [0(Sunday),6].   
%W  Week number of the year (Monday as the first day of the week)
%x  Locale’s appropriate date representation.    
%X  Locale’s appropriate time representation.    
%y  Year without century as a decimal number [00,99].    
%Y  Year with century as a decimal number.   
%z  UTC offset in the form +HHMM or -HHMM.
%Z  Time zone name (empty string if the object is naive).    
%%  A literal '%' character.
```

## Inconsistent Data Entry

在我們上面範例中，數據集中的表頭有空白地方而我們將解決這一個部分。在數據集中對於一些字串可能也會有全小寫或大寫的需求，而這也能夠透過程式來解決。

```python
header = ddos.columns[:4]
print(header)
'''
Index(['Unnamed: 0', 'Flow ID', ' Source IP', ' Source Port'], dtype='object')
'''

print(header.str.strip()) # 去除頭尾空白
Index(['Unnamed: 0', 'Flow ID', 'Source IP', 'Source Port'], dtype='object')

```