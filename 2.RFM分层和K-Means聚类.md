## 数据分层以及聚类

### 为什么选取 RFM + K-means 对客户进行分群标签？

#### RFM 的意义
R 代表着 **Recency**，表示客户多久没有购买东西，值越小则客户越活跃。  
F 代表着 **Frequency**，表示客户的购买次数。  
M 代表着 **Monetary**，表示客户的总消费金额。

在这三个指标里：
- **R** 可依赖 `TransactionDate` 字段计算  
- **F** 通过 `CustomerID` 分组后 `count()` 计算  
- **M** 依赖 `TotalAmount` 字段

RFM 进行客户分群的优势在于 **简单直观**，可清晰描述客户的价值。

---

### 为什么选择 K-Means 进行聚类？
K-Means 主要用于寻找数据中的自然分类，**计算高效**，适合大规模数据集，并能 **自动划分群组**。

---

## 实现 RFM 分层代码
# RFM 分析完整代码及讲解

## 1. 导入 Python 库
```python
import pandas as pd
import numpy as np
from datetime import datetime
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans
```

本部分导入了必要的 Python 库：
- `pandas` 用于数据处理。
- `numpy` 用于数值计算。
- `datetime` 处理时间数据。
- `StandardScaler` 进行数据标准化。
- `KMeans` 进行 K-Means 聚类。

## 2. 读取 CSV 文件
```python
file_path = r"C:\Users\32165\Desktop\retail\cleaned_retail.csv"
df = pd.read_csv(file_path, dtype={"CustomerID": str}, parse_dates=["TransactionDate"])
df["OriginalOrder"] = df.index  # 记录原始索引，便于后续排序恢复
```

- `dtype={"CustomerID": str}`：确保 `CustomerID` 作为字符串处理，避免数值格式化问题。
- `parse_dates=["TransactionDate"]`：将 `TransactionDate` 列解析为日期格式。
- 记录 `OriginalOrder`，以便后续恢复原始顺序。

## 3. 计算 R 指标的基准日期
```python
max_date = df["TransactionDate"].max()
if max_date.month == 12:
    cutoff_date = datetime(max_date.year + 1, 1, 1)
else:
    cutoff_date = datetime(max_date.year, max_date.month + 1, 1)
print(f"\n基准日期（次月1日）：{cutoff_date.strftime('%Y-%m-%d')}")
```

- `max_date` 获取数据集中最新的交易日期。
- `cutoff_date` 设定为 `max_date` 次月 1 号，作为 `Recency` 计算的基准日期。

## 4. 计算 RFM 指标
```python
rfm = df.groupby("CustomerID", as_index=False).agg(
    Recency=("TransactionDate", lambda x: (cutoff_date - x.max()).days),
    Frequency=("TransactionDate", "count"),
    Monetary=("TotalAmount", "sum")
)
```

- `Recency` 计算客户最近一次交易距离基准日期的天数。
- `Frequency` 统计每个客户的交易次数。
- `Monetary` 计算客户的总消费金额。

## 5. 数据标准化
```python
scaler = StandardScaler()
rfm_scaled = scaler.fit_transform(rfm[["Recency", "Frequency", "Monetary"]])
rfm_scaled_df = pd.DataFrame(rfm_scaled, columns=["Recency", "Frequency", "Monetary"])
```

- `StandardScaler` 标准化数据，使其均值为 0，标准差为 1，避免特征量纲影响聚类。

## 6. K-Means 聚类
```python
kmeans = KMeans(n_clusters=4, random_state=42, n_init=10)
rfm["Cluster"] = kmeans.fit_predict(rfm_scaled_df)
cluster_means = rfm.groupby("Cluster")[["Recency", "Frequency", "Monetary"]].mean()
print("\n各簇 RFM 平均值：")
print(cluster_means)
```

- `n_clusters=4`：指定 4 个聚类类别。
- `fit_predict()` 进行 K-Means 聚类，并将结果存入 `Cluster` 列。
- `cluster_means` 计算每个聚类的平均 RFM 值。

## 7. 定义分群标签
```python
cluster_labels = {}
sorted_clusters = cluster_means.sort_values(by=["Monetary", "Frequency", "Recency"], ascending=[False, False, True]).index
cluster_labels[sorted_clusters[0]] = "流失客户"
cluster_labels[sorted_clusters[1]] = "一般客户"
cluster_labels[sorted_clusters[2]] = "高价值客户"
cluster_labels[sorted_clusters[3]] = "潜力客户"
rfm["Cluster_Label"] = rfm["Cluster"].map(cluster_labels)
```

- 按 `Monetary`、`Frequency`、`Recency` 进行排序，确定群组标签。
- `map()` 进行标签映射。

## 8. 恢复原始数据排列顺序
```python
rfm = df[["CustomerID", "OriginalOrder"]].drop_duplicates().merge(rfm, on="CustomerID", how="left")
rfm = rfm.sort_values("OriginalOrder").drop(columns=["OriginalOrder"])
```

- `drop_duplicates()` 确保 `CustomerID` 唯一。
- `merge()` 结合原始索引以恢复顺序。
- `sort_values("OriginalOrder")` 重新按原始顺序排列。

## 9. 统计分析
```python
original_counts = rfm["Cluster_Label"].value_counts()
unique_counts = rfm.drop_duplicates(subset=["CustomerID"])["Cluster_Label"].value_counts()
print("\n🔹 原始统计（可能有重复 CustomerID）：")
print(original_counts)
print("\n🔹 去重后统计（每个 CustomerID 仅计算一次）：")
print(unique_counts)
output_file_path = r"C:\Users\32165\Desktop\rfm_result.csv"
rfm.drop_duplicates(subset=["CustomerID"]).to_csv(output_file_path, index=False, encoding="utf-8-sig")
print(f"\n✅ RFM 结果已保存至: {output_file_path}")
```

- `value_counts()` 计算每个分群的客户数量。
- `drop_duplicates(subset=["CustomerID"])` 确保 `CustomerID` 唯一。
- 结果存入 `rfm_result.csv`。

## 10. 统计结果示例
```yaml
🔹 原始统计（可能有重复 CustomerID）：
潜力客户     35057  
高价值客户    34958  
流失客户     20579  
一般客户      9406  

🔹 去重后统计（每个 CustomerID 仅计算一次）：
潜力客户     35057  
高价值客户    34958  
流失客户     20579  
一般客户      4621  
```
