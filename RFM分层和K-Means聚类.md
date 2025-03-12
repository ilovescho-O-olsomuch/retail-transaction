# 数据分层以及聚类

## 为什么选取RFM+K-means对客户进行分群标签
### RFM的意义
### R代表着Recency，表示客户多久没有购买东西，值越小则客户越活跃，F则代表着Frequency，表示客户的购买次数，M代表着Monetary，总的消费金额
### 这三个指标里R可以依赖于TransactionDate字段，F则通过customerID分组后计数count()可得结果，M则可以依赖于TotalAmount字段
### RFM进行客户分群的优势是简单直观，可清晰描述客户的价值

## 为什么选择K-Means进行聚类
### K-Means主要用于寻找数据中的自然分类，计算更加高效，适合大规模数据集，可以自动划分群组。

## 实现RFM分层代码
``` python
import pandas as pd
import numpy as np
from datetime import datetime
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans
```
### 导入python库

``` python
# 读取 CSV 文件，确保 CustomerID 作为字符串处理，TransactionDate 解析为日期格式
file_path = r"C:\Users\32165\Desktop\cleaned_retail.csv"
df = pd.read_csv(file_path, dtype={"CustomerID": str}, parse_dates=["TransactionDate"])
# 记录 CustomerID 原始顺序（用于后续恢复）
df["OriginalOrder"] = df.index  # 记录原始索引，便于后续排序恢复
```
### 后续的groupby函数会自动将customerID分类后的结果按customerID大小进行排序，所以要固定输出按照原始顺序

``` python
# 获取数据中的最大交易日期
max_date = df["TransactionDate"].max()

if max_date.month == 12:
    cutoff_date = datetime(max_date.year + 1, 1, 1)
else:
    cutoff_date = datetime(max_date.year, max_date.month + 1, 1)

# 输出基准日期
print(f"\n基准日期（次月1日）：{cutoff_date.strftime('%Y-%m-%d')}")
```
### 进行R指标的基准日期计算，基准日期选择了数据中最大交易日期的次月一日，还预留了最大交易日期是12月的处理代码。

``` python
rfm = df.groupby("CustomerID", as_index=False).agg(
    Recency=("TransactionDate", lambda x: (cutoff_date - x.max()).days),  # 最近购买天数
    Frequency=("TransactionDate", "count"),  # 购买次数
    Monetary=("TotalAmount", "sum")  # 总消费金额
)
···
### RFM指标的计算全部是以单个不同的customerID为单位进行的

``` python
# 使用 StandardScaler 对数据进行标准化
scaler = StandardScaler()
rfm_scaled = scaler.fit_transform(rfm[["Recency", "Frequency", "Monetary"]])

# 转换为 DataFrame 
rfm_scaled_df = pd.DataFrame(rfm_scaled, columns=["Recency", "Frequency", "Monetary"])
```
### 这里的标准化数据主要是为了后续的K-Means处理

``` python
kmeans = KMeans(n_clusters=4, random_state=42, n_init=10)

# 进行聚类，并将聚类标签赋值给 RFM 数据
rfm["Cluster"] = kmeans.fit_predict(rfm_scaled_df)

# 定义分群标签
cluster_labels = {
    0: "流失客户",
    1: "一般客户",
    2: "高价值客户",
    3: "潜力客户"
}

# 映射分群标签
rfm["Cluster_Label"] = rfm["Cluster"].map(cluster_labels)
···
### 确定好K-Means分类后标签有四个

``` python
# 通过合并恢复 CustomerID 原始顺序
rfm = df[["CustomerID", "OriginalOrder"]].drop_duplicates().merge(rfm, on="CustomerID", how="left")

# 按原始顺序排序，并删除辅助列
rfm = rfm.sort_values("OriginalOrder").drop(columns=["OriginalOrder"])
```
### 恢复原始的数据排列顺序
### ![RFM+K-means输出效果图](https://github.com/ilovescho-O-olsomuch/retail-transaction/blob/main/RFM%2BKMEANS.png)

``` python
# 原始分群统计
original_counts = rfm["Cluster_Label"].value_counts()

# 去重后的分群统计
unique_counts = rfm.drop_duplicates(subset=["CustomerID"])["Cluster_Label"].value_counts()

print("\n🔹 原始统计（可能有重复 CustomerID）：")
print(original_counts)

print("\n🔹 去重后统计（每个 CustomerID 仅计算一次）：")
print(unique_counts)

# 检查去重前后的总人数是否一致
print("\n✅ 总人数对比（去重前 vs. 去重后）：")
print(f"去重前总人数：{original_counts.sum()}")
print(f"去重后总人数：{unique_counts.sum()}")
```
### 最后进行分群统计
###  
原始统计（可能有重复 CustomerID）：
Cluster_Label
潜力客户     35057
流失客户     34958
一般客户     20579
高价值客户     9406
Name: count, dtype: int64

去重后统计（每个 CustomerID 仅计算一次）：
Cluster_Label
潜力客户     35057
流失客户     34958
一般客户     20579
高价值客户     4621
Name: count, dtype: int64

总人数对比（去重前 vs. 去重后）：
去重前总人数：100000
去重后总人数：95215


