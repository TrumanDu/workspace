# python 学习笔记之数据分析

## NumPy

创建

```python
array1 = np.array([1, 2, 3, 4, 5])
array2 = np.arange(0, 20, 2)
# 产生10个$[1, 100)$范围的随机整数
array5 = np.random.randint(1, 100, 10)
array7 = np.array([[1, 2, 3], [4, 5, 6]])
array8 = np.zeros((3, 4))
array9 = np.ones((3, 4))
# 使用eye函数创建单位矩阵
array11 = np.eye(4)
# 通过reshape将一维数组变成二维数组
array12 = np.array([1, 2, 3, 4, 5, 6]).reshape(2, 3)
```

## Pandas

Pandas 核心的数据类型是 Series（数据系列）、DataFrame（数据表/数据框），分别用于处理一维和二维的数据

### 使用场景

获取及修改指定数据

```python
sales_df.loc[index, "name"]
sales_df.loc[0, "name"]='truman'
```
排序
```
df=df.sort_values('price',ascending=True)
```

df数据拼接
```python
all_emp_df = pd.concat([emp_df, emp2_df])
```


遍历数据
```python
# 使用 iterrows() 遍历 DataFrame
for index, row in df.iterrows():
    print(index, row['Name'], row['Age'], row['City'])

# 使用 itertuples() 遍历 DataFrame
for row in df.itertuples():
    print(row.Index, row.Name, row.Age, row.City)

# 使用 iteritems(),按列迭代 遍历 DataFrame
for column, series in df.iteritems():
    print(column)
    for value in series:
        print(value)
```


按数值范围分桶

```python
bins = [0, 100, 200, 300, 400, 500, 1000, 2000, 3000]
labels = ['<100', '100-200', '200-300', '300-400', '400-500', '500-1000', '1000-2000', '>=2000']
price_groups = pd.cut(df['price'], bins=bins, labels=labels, right=False)
```

分组求和

```python
click_count_by_price = df.groupby(price_groups)['click_sum'].sum()
```

## Matplotlib 与 Seaborn

## Scikit-learn

## 参考

1. [数据分析概述](https://github.com/jackfrued/Python-100-Days/blob/master/Day66-80/66.%E6%95%B0%E6%8D%AE%E5%88%86%E6%9E%90%E6%A6%82%E8%BF%B0.md)
