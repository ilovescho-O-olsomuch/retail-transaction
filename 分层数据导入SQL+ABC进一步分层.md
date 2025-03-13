# 数据导入SQL存储
``` python
import pymysql
import pandas as pd

# MySQL 连接配置
host = "localhost"  # MySQL 服务器地址
user = "root"       # MySQL 用户名
password = "123456"  # MySQL 密码
database = "retailDB"  # 数据库名称
table_name = "customer_segments"

# 读取 CSV 文件，确保数据格式正确
csv_file = r"C:\Users\32165\Desktop\retail\customer_segments.csv"
df = pd.read_csv(csv_file, sep=",", dtype=str)  # 强制所有列为字符串

# 连接 MySQL
conn = pymysql.connect(host=host, user=user, password=password, database=database, charset='utf8mb4')
cursor = conn.cursor()

# 创建表（如果不存在）
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

# 转换数据类型，确保格式正确
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

## ABC分类
``` python
import pandas as pd

# 读取数据
csv_file = r"C:\Users\32165\Desktop\retail\customer_segments.csv"
df = pd.read_csv(csv_file, sep=",", dtype={'CustomerID': str})  # 确保 CustomerID 是字符串

# 去除列名前后空格
df.columns = df.columns.str.strip()

# 确保 'Monetary' 是浮点数
df["Monetary"] = df["Monetary"].astype(float)

# 按销售额排序（降序）
df = df.sort_values(by="Monetary", ascending=False)

# 计算累计销售额占比
df["Cumulative_Sum"] = df["Monetary"].cumsum()
df["Cumulative_Percent"] = df["Cumulative_Sum"] / df["Monetary"].sum()

# 定义 ABC 分类
def classify_abc(percent):
    if percent <= 0.7:
        return "A类客户"
    elif percent <= 0.9:
        return "B类客户"
    else:
        return "C类客户"

df["ABC_Category"] = df["Cumulative_Percent"].apply(classify_abc)

# **导出 ABC 分类结果**
output_file = r"C:\Users\32165\Desktop\retail\abc_classification.csv"
df.to_csv(output_file, index=False, encoding="utf-8-sig")  # 确保支持中文

print(f"ABC 分类结果已导出至：{output_file}")

# **不去重统计**
total_sales = df["Monetary"].sum()
total_orders = len(df)

abc_stats = df.groupby("ABC_Category")["Monetary"].agg(["sum", "count"]).rename(columns={"sum": "销售金额", "count": "订单数"})
abc_stats["销售金额占比"] = (abc_stats["销售金额"] / total_sales * 100).round(2).astype(str) + "%"
abc_stats["订单数占比"] = (abc_stats["订单数"] / total_orders * 100).round(2).astype(str) + "%"

# **去重后统计**
df_unique = df.drop_duplicates(subset="CustomerID")  # 按客户ID去重
total_sales_unique = df_unique["Monetary"].sum()
total_customers = len(df_unique)

abc_unique_stats = df_unique.groupby("ABC_Category")["Monetary"].agg(["sum", "count"]).rename(columns={"sum": "销售金额", "count": "客户数"})
abc_unique_stats["销售金额占比"] = (abc_unique_stats["销售金额"] / total_sales_unique * 100).round(2).astype(str) + "%"
abc_unique_stats["客户数占比"] = (abc_unique_stats["客户数"] / total_customers * 100).round(2).astype(str) + "%"

# **导出统计数据**
stats_output_file = r"C:\Users\32165\Desktop\retail\abc_statistics.csv"
abc_stats.to_csv(stats_output_file, encoding="utf-8-sig")
abc_unique_stats.to_csv(stats_output_file, mode="a", encoding="utf-8-sig")  # 追加写入

print(f"ABC 分类统计已导出至：{stats_output_file}")
```
