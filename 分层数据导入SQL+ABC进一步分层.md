# RFM分层以及K-Means聚类后的数据导入SQL存储
``` python
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

## ABC分类
``` python
import pandas as pd
``
### python数据读取，确保customerID识别为字符串而不是数值，并且做好数据转换
``` python
csv_file = r"C:\Users\32165\Desktop\retail\customer_segments.csv"
df = pd.read_csv(csv_file, sep=",", dtype={'CustomerID': str})  
df.columns = df.columns.str.strip()
df["Monetary"] = df["Monetary"].astype(float)
```
### 个人销售额降序排列，从大到小，并且计算每行的积累销售额，从而达到根据销售额占比对客户进行ABC等级分类的目的
``` python
df = df.sort_values(by="Monetary", ascending=False)
df["Cumulative_Sum"] = df["Monetary"].cumsum()
df["Cumulative_Percent"] = df["Cumulative_Sum"] / df["Monetary"].sum()
···

### ABC分类的定义前 70% 的销售额 → A 类客户（最重要） 70% - 90% → B 类客户（次重要） 最后 10% → C 类客户（普通客户）
``` python
def classify_abc(percent):
    if percent <= 0.7:
        return "A类客户"
    elif percent <= 0.9:
        return "B类客户"
    else:
        return "C类客户"
```

df["ABC_Category"] = df["Cumulative_Percent"].apply(classify_abc)


### 导出 ABC 分类结果
``` python
output_file = r"C:\Users\32165\Desktop\retail\abc_classification.csv"
df.to_csv(output_file, index=False, encoding="utf-8-sig")  
print(f"ABC 分类结果已导出至：{output_file}")
```

### 不去重统计
``` python
total_sales = df["Monetary"].sum()
total_orders = len(df)
abc_stats = df.groupby("ABC_Category")["Monetary"].agg(["sum", "count"]).rename(columns={"sum": "销售金额", "count": "订单数"})
abc_stats["销售金额占比"] = (abc_stats["销售金额"] / total_sales * 100).round(2).astype(str) + "%"
abc_stats["订单数占比"] = (abc_stats["订单数"] / total_orders * 100).round(2).astype(str) + "%"
```

### 去重后统计
``` python
df_unique = df.drop_duplicates(subset="CustomerID")  # 按CustomerID去重
total_sales_unique = df_unique["Monetary"].sum()
total_customers = len(df_unique)
abc_unique_stats = df_unique.groupby("ABC_Category")["Monetary"].agg(["sum", "count"]).rename(columns={"sum": "销售金额", "count": "客户数"})
abc_unique_stats["销售金额占比"] = (abc_unique_stats["销售金额"] / total_sales_unique * 100).round(2).astype(str) + "%"
abc_unique_stats["客户数占比"] = (abc_unique_stats["客户数"] / total_customers * 100).round(2).astype(str) + "%"
```

### 导出统计数据
``` python
stats_output_file = r"C:\Users\32165\Desktop\retail\abc_statistics.csv"
abc_stats.to_csv(stats_output_file, encoding="utf-8-sig")
abc_unique_stats.to_csv(stats_output_file, mode="a", encoding="utf-8-sig")  
print(f"ABC 分类统计已导出至：{stats_output_file}")
```
### 最后的统计结果截图 先不去重后去重数据
![统计结果](https://github.com/ilovescho-O-olsomuch/retail-transaction/blob/main/%E5%90%8C.png)
