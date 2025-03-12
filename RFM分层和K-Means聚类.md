# 数据分层以及聚类

## 为什么选取 RFM + K-means 对客户进行分群标签？

### RFM 的意义
R 代表着 **Recency**，表示客户多久没有购买东西，值越小则客户越活跃。  
F 代表着 **Frequency**，表示客户的购买次数。  
M 代表着 **Monetary**，表示客户的总消费金额。

在这三个指标里：
- **R** 可依赖 `TransactionDate` 字段计算  
- **F** 通过 `CustomerID` 分组后 `count()` 计算  
- **M** 依赖 `TotalAmount` 字段

RFM 进行客户分群的优势在于 **简单直观**，可清晰描述客户的价值。

---

## 为什么选择 K-Means 进行聚类？
K-Means 主要用于寻找数据中的自然分类，**计算高效**，适合大规模数据集，并能 **自动划分群组**。

---

## 实现 RFM 分层代码

### 导入 Python 库
```python
import pandas as pd
import numpy as np
from datetime import datetime
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans
```

### 读取 CSV 文件
```python
file_path = r"C:\Users\32165\Desktop\cleaned_retail.csv"
df = pd.read_csv(file_path, dtype={"CustomerID": str}, parse_dates=["TransactionDate"])

# 记录 CustomerID 原始顺序（用于后续恢复）
df["OriginalOrder"] = df.index  # 记录原始索引，便于后续排序恢复
```

### 计算 R 指标的基准日期
```python
# 获取数据中的最大交易日期
max_date = df["TransactionDate"].max()
if max_date.month == 12:
    cutoff_date = datetime(max_date.year + 1, 1, 1)
else:
    cutoff_date = datetime(max_date.year, max_date.month + 1, 1)

# 输出基准日期
print(f"\n基准日期（次月1日）：{cutoff_date.strftime('%Y-%m-%d')}")
```

### 计算 RFM 指标
```python
rfm = df.groupby("CustomerID", as_index=False).agg(
    Recency=("TransactionDate", lambda x: (cutoff_date - x.max()).days),  # 最近购买天数
    Frequency=("TransactionDate", "count"),  # 购买次数
    Monetary=("TotalAmount", "sum")  # 总消费金额
)
```

### 数据标准化
```python
scaler = StandardScaler()
rfm_scaled = scaler.fit_transform(rfm[["Recency", "Frequency", "Monetary"]])

# 转换为 DataFrame 
rfm_scaled_df = pd.DataFrame(rfm_scaled, columns=["Recency", "Frequency", "Monetary"])
```

### K-Means 聚类
```python
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
```

### 恢复原始数据排列顺序
```python
# 通过合并恢复 CustomerID 原始顺序
rfm = df[["CustomerID", "OriginalOrder"]].drop_duplicates().merge(rfm, on="CustomerID", how="left")

# 按原始顺序排序，并删除辅助列
rfm = rfm.sort_values("OriginalOrder").drop(columns=["OriginalOrder"])
```

### RFM+K-means 输出效果
![RFM+K-means 输出效果](https://github.com/ilovescho-O-olsomuch/retail-transaction/blob/main/RFM%2BKMEANS.png)

### 统计分析
```python
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

### 统计结果示例
```
🔹 原始统计（可能有重复 CustomerID）：
潜力客户：35057  
流失客户：34958  
一般客户：20579  
高价值客户：9406  

🔹 去重后统计（每个 CustomerID 仅计算一次）：
潜力客户：12035  
流失客户：11875  
一般客户：8543  
高价值客户：4120  

✅ 总人数对比（去重前 vs. 去重后）：
去重前总人数：100000  
去重后总人数：36573  
