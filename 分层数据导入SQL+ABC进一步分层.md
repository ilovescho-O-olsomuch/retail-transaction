# RFM 分层及 K-Means 聚类数据导入 SQL 存储

## 1. RFM 分层与 K-Means 聚类数据导入 MySQL

```python
import pandas as pd
import pymysql
from sqlalchemy import create_engine

# 1. 读取 CSV 文件（解决编码问题）
file_path = r"C:\Users\32165\Desktop\result.csv"
try:
    # 尝试常见编码格式，如果失败请手动确认文件编码（如 GBK、ANSI、ISO-8859-1）
    df = pd.read_csv(file_path, encoding='gbk', engine='python', on_bad_lines='skip')
except UnicodeDecodeError:
    df = pd.read_csv(file_path, encoding='ISO-8859-1', engine='python', on_bad_lines='skip')

# 2. 处理缺失值和数据类型
df = df.fillna('')  # 将 NaN 替换为空字符串或默认值
df = df.astype({
    'CustomerID': str,  # 根据数据库表结构调整类型
    'Recency': int,
    'Frequency': int,
    'Monetary': float,
    'Cluster': int,
    'Cluster_Label': str
})

# 3. 使用 SQLAlchemy 高效批量插入数据（替代逐行插入）
# 安装依赖：pip install sqlalchemy
db_connection_str = 'mysql+pymysql://root:123456@localhost/retailDB?charset=utf8mb4'
engine = create_engine(db_connection_str)

try:
    # 直接映射 DataFrame 到数据库表
    df.to_sql(
        name='customers',
        con=engine,
        if_exists='append',  # 追加模式（避免覆盖）
        index=False,          # 不插入索引列
        chunksize=1000       # 分批提交提升性能
    )
    print("数据导入完成！")
except Exception as e:
    print(f"导入失败: {e}")
finally:
    engine.dispose()
   
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

csv_file = r"C:\Users\32165\Desktop\result.csv"
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

### 2.6 统计分析（已去重）
```python
total_sales_unique = df["Monetary"].sum()
total_customers = len(df)
abc_unique_stats = df.groupby("ABC_Category")["Monetary"].agg(["sum", "count"]).rename(columns={"sum": "销售金额", "count": "客户数"})
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

### 2.8 统计结果
| ABC_Category | 销售金额      | 客户数  | 销售金额占比 | 客户数占比 |
|-------------|-------------|--------|------------|---------|
| A类客户    | 17383236.48 | 37540  | 70.00%     | 39.43%  |
| B类客户    | 4966869.76  | 24822  | 20.00%     | 26.07%  |
| C类客户    | 2483388.41  | 32853  | 10.00%     | 34.50%  |

