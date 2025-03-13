# RFM 分层及 K-Means 聚类数据导入 SQL 存储

## 1. RFM 分层与 K-Means 聚类数据导入 MySQL

```python
import pymysql
import pandas as pd

host = "localhost"  # MySQL 服务器地址
user = "root"       # MySQL 用户名
password = "123456"  # MySQL 密码
database = "retailDB"  # 数据库名称
table_name = "customer_segments"

csv_file = r"C:\Users\32165\Desktop\retail\customer_segments.csv"
df = pd.read_csv(csv_file, sep=",", dtype=str)  

conn = pymysql.connect(host=host, user=user, password=password, database=database, charset='utf8mb4')
cursor = conn.cursor()

# 创建表
create_table_query = f"""
CREATE TABLE IF NOT EXISTS {table_name} (
    CustomerID INT PRIMARY KEY,
    Recency INT,
    Frequency INT,
    Monetary DECIMAL(10,2),
    Cluster INT,
    Cluster_Label VARCHAR(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci
);
"""
cursor.execute(create_table_query)

# 转换数据类型
df["CustomerID"] = df["CustomerID"].astype(int)
df["Recency"] = df["Recency"].astype(int)
df["Frequency"] = df["Frequency"].astype(int)
df["Monetary"] = df["Monetary"].astype(float)
df["Cluster"] = df["Cluster"].astype(int)

# 插入数据 SQL 语句
insert_query = f"""
INSERT INTO {table_name} (CustomerID, Recency, Frequency, Monetary, Cluster, Cluster_Label)
VALUES (%s, %s, %s, %s, %s, %s)
ON DUPLICATE KEY UPDATE 
    Recency=VALUES(Recency), 
    Frequency=VALUES(Frequency), 
    Monetary=VALUES(Monetary), 
    Cluster=VALUES(Cluster), 
    Cluster_Label=VALUES(Cluster_Label);
"""

# 批量插入数据
data = [tuple(row) for row in df.values]
cursor.executemany(insert_query, data)

# 提交并关闭连接
conn.commit()
cursor.close()
conn.close()

print("数据导入完成！")
```

---

## 2. ABC 分类分析
## 选择ABC分类的理由
1. RFM & K-Means 聚类：按照 客户行为模式（最近购买、购买频率、消费金额）进行分类，识别不同客户群体（如 VIP、流失客户、潜力客户）。
2. ABC 分类：仅按照 销售金额占比 来划分客户，确保业务重点关注高贡献客户。
3. RFM & K-Means 用于 客户细分，如区分“高价值但低频客户”和“低价值高频客户”。
4. ABC 分类 用于 市场决策，例如 A 类客户需要 VIP 服务，C 类客户可以做低成本营销。

### 2.1 读取数据并确保正确格式
```python
import pandas as pd

csv_file = r"C:\Users\32165\Desktop\retail\customer_segments.csv"
df = pd.read_csv(csv_file, sep=",", dtype={'CustomerID': str})  
df.columns = df.columns.str.strip()
df["Monetary"] = df["Monetary"].astype(float)
```

### 2.2 计算累积销售额并排序
```python
df = df.sort_values(by="Monetary", ascending=False)
df["Cumulative_Sum"] = df["Monetary"].cumsum()
df["Cumulative_Percent"] = df["Cumulative_Sum"] / df["Monetary"].sum()
```

### 2.3 ABC 分类标准
ABC 分类用于根据销售额贡献度对客户进行分层，分为 A、B、C 三类：

- **A 类客户**：贡献了前 70% 的销售额，最重要的核心客户。
- **B 类客户**：贡献 70% - 90% 的销售额，次重要客户。
- **C 类客户**：贡献最后 10% 的销售额，普通客户。

```python
def classify_abc(percent):
    if percent <= 0.7:
        return "A类客户"
    elif percent <= 0.9:
        return "B类客户"
    else:
        return "C类客户"

df["ABC_Category"] = df["Cumulative_Percent"].apply(classify_abc)
```

### 2.4 导出 ABC 分类结果
```python
output_file = r"C:\Users\32165\Desktop\retail\abc_classification.csv"
df.to_csv(output_file, index=False, encoding="utf-8-sig")  
print(f"ABC 分类结果已导出至：{output_file}")
```

### 2.5 统计分析（不去重）
```python
total_sales = df["Monetary"].sum()
total_orders = len(df)
abc_stats = df.groupby("ABC_Category")["Monetary"].agg(["sum", "count"]).rename(columns={"sum": "销售金额", "count": "订单数"})
abc_stats["销售金额占比"] = (abc_stats["销售金额"] / total_sales * 100).round(2).astype(str) + "%"
abc_stats["订单数占比"] = (abc_stats["订单数"] / total_orders * 100).round(2).astype(str) + "%"
```

### 2.6 统计分析（去重后按客户统计）
```python
df_unique = df.drop_duplicates(subset="CustomerID")  # 按 CustomerID 去重
total_sales_unique = df_unique["Monetary"].sum()
total_customers = len(df_unique)
abc_unique_stats = df_unique.groupby("ABC_Category")["Monetary"].agg(["sum", "count"]).rename(columns={"sum": "销售金额", "count": "客户数"})
abc_unique_stats["销售金额占比"] = (abc_unique_stats["销售金额"] / total_sales_unique * 100).round(2).astype(str) + "%"
abc_unique_stats["客户数占比"] = (abc_unique_stats["客户数"] / total_customers * 100).round(2).astype(str) + "%"
```

### 2.7 导出统计数据
```python
stats_output_file = r"C:\Users\32165\Desktop\retail\abc_statistics.csv"
abc_stats.to_csv(stats_output_file, encoding="utf-8-sig")
abc_unique_stats.to_csv(stats_output_file, mode="a", encoding="utf-8-sig")  
print(f"ABC 分类统计已导出至：{stats_output_file}")
```

### 2.8 统计结果示例
![统计结果](https://github.com/ilovescho-O-olsomuch/retail-transaction/blob/main/%E5%90%8C.png)
