## 基本代码
``` python
import pandas as pd
import numpy as np
from datetime import datetime
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans

# ================================================
# 1. 读取数据（确保 CustomerID 为字符串）
# ================================================
file_path = r"C:\Users\32165\Desktop\cleaned_retail.csv"
df = pd.read_csv(file_path, dtype={"CustomerID": str}, parse_dates=["TransactionDate"])

# **记录 CustomerID 原始顺序**
df["OriginalOrder"] = df.index  # 添加索引，代表原始顺序

# ================================================
# 2. 计算基准日期（最大交易日期的次月1日）
# ================================================
max_date = df["TransactionDate"].max()
if max_date.month == 12:
    cutoff_date = datetime(max_date.year + 1, 1, 1)
else:
    cutoff_date = datetime(max_date.year, max_date.month + 1, 1)

print(f"\n基准日期（次月1日）：{cutoff_date.strftime('%Y-%m-%d')}")

# ================================================
# 3. 计算 RFM 指标
# ================================================
rfm = df.groupby("CustomerID", as_index=False).agg(
    Recency=("TransactionDate", lambda x: (cutoff_date - x.max()).days),  # 最近购买天数
    Frequency=("TransactionDate", "count"),  # 购买次数
    Monetary=("TotalAmount", "sum")  # 总消费金额
)

# ================================================
# 4. 数据标准化（K-means 必需）
# ================================================
scaler = StandardScaler()
rfm_scaled = scaler.fit_transform(rfm[["Recency", "Frequency", "Monetary"]])
rfm_scaled_df = pd.DataFrame(rfm_scaled, columns=["Recency", "Frequency", "Monetary"])

# ================================================
# 5. K-means 聚类（分成 4 组）
# ================================================
kmeans = KMeans(n_clusters=4, random_state=42, n_init=10)
rfm["Cluster"] = kmeans.fit_predict(rfm_scaled_df)

# 定义分群标签
cluster_labels = {
    0: "流失客户",
    1: "一般客户",
    2: "高价值客户",
    3: "潜力客户"
}
rfm["Cluster_Label"] = rfm["Cluster"].map(cluster_labels)

# ================================================
# 6. 恢复原始顺序
# ================================================
rfm = df[["CustomerID", "OriginalOrder"]].drop_duplicates().merge(rfm, on="CustomerID", how="left")
rfm = rfm.sort_values("OriginalOrder").drop(columns=["OriginalOrder"])  # 按原始顺序排序，并删除辅助列

# ================================================
# 7. 导出分群结果，确保 CustomerID 正确
# ================================================
output_path = r"C:\Users\32165\Desktop\customer_segments.csv"
try:
    rfm.to_csv(output_path, index=False, quoting=1)  # quoting=1 避免 CustomerID 变形
    print("\n✅ RFM 计算完成，分群结果已保存至:", output_path)
    print("\n📊 分群结果示例：")
    print(rfm.head())
except PermissionError:
    print("\n❌ 错误：文件可能被占用，请关闭 Excel 或更换保存路径。")

# ================================================
# 8. 分群结果统计（验证用）
# ================================================
print("\n📌 分群统计：")
print(rfm["Cluster_Label"].value_counts())
