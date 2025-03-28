# **客户分析与营销优化项目 README**

## **1️⃣ 项目背景（Introduction）**

### **项目目标**

本项目旨在优化客户分类，并基于数据驱动的策略提升营销精准度。通过 RFM 分析、K-Means 聚类、ABC 分类等方法，识别不同类型的客户，并探索折扣策略对销售的影响。

### **数据来源**

数据来自 Kaggle，包含零售环境中的交易记录，涉及消费者行为、产品销售情况以及支付方式等信息。[数据集链接](https://www.kaggle.com/datasets/fahadrehman07/retail-transaction-dataset)

数据集包含 10 个关键字段：

1. **CustomerID**：唯一标识每位客户
2. **ProductID**：唯一标识每个产品
3. **Quantity**：购买的商品数量
4. **Price**：商品单价
5. **TransactionDate**：交易发生的日期时间
6. **PaymentMethod**：支付方式（如 PayPal、现金、信用卡等）
7. **StoreLocation**：交易发生的地点
8. **ProductCategory**：商品类别
9. **DiscountApplied(%)**：折扣百分比
10. **TotalAmount**：交易总金额

## **2️⃣ 数据处理（Data Processing）**

### **数据清洗（Data Cleaning）**

1. **CustomerID** 检查重复状况但不去重
2. **Price、DiscountApplied、TotalAmount** 保留两位小数
3. **TransactionDate** 仅保留日期部分
4. **StoreLocation** 仅保留美国州级单位

### **特征工程（Feature Engineering）**

#### **RFM 指标计算**

- **R（Recency）**：计算每位客户最近一次交易时间与基准时间（数据集最大日期的次月 1 号）之间的间隔
- **F（Frequency）**：统计每位客户的交易次数
- **M（Monetary）**：计算客户的总交易金额

#### **ABC 分类**

基于销售金额的累积比例划分：

- **A 类客户（Top 70%）**：贡献最多销售额的客户
- **B 类客户（中间 20%）**
- **C 类客户（底部 10%）**

## **3️⃣ 关键分析（Key Insights）**

### **RFM 分析结果**

#### **客户分类结果（去重后统计，每个 CustomerID 仅计算一次）**

| 客户分类  | 客户数    |
| ----- | ------ |
| 潜力客户  | 35,057 |
| 高价值客户 | 34,958 |
| 流失客户  | 20,579 |
| 一般客户  | 4,621  |

### **K-Means 聚类分析**

| Cluster  | Recency | Frequency | Monetary |
| -------- | ------- | --------- | -------- |
| 0（流失客户）  | 279.70  | 1.00      | 167.65   |
| 1（一般客户）  | 185.77  | 1.00      | 524.91   |
| 2（高价值客户） | 121.60  | 2.04      | 503.23   |
| 3（潜力客户）  | 89.98   | 1.00      | 166.73   |

### **ABC 分类结果**

| ABC 类别 | 销售金额          | 客户数    | 销售金额占比 | 客户数占比  |
| ------ | ------------- | ------ | ------ | ------ |
| A 类客户  | 17,383,236.48 | 37,540 | 70.00% | 39.43% |
| B 类客户  | 4,966,869.76  | 24,822 | 20.00% | 26.07% |
| C 类客户  | 2,483,388.41  | 32,853 | 10.00% | 34.50% |

### **客户流失与折扣策略分析**

- 目前 **A 类客户流失较多**，但 B 类和 C 类客户表现出更高的价值和潜力，人数基本均分。
- **ABC 三类客户的折扣集中于 0%~15%**，说明低折扣策略在所有客户群体中较为普遍。
- **四个客户群体的数量基本相当**，表明当前客户结构较为均衡。
- **三类客户的支付方式分布基本平均，其中 PayPal 使用率稍高**。
- **流失客户享受的折扣最少**，表明折扣可能对客户忠诚度有一定影响。

## **4️⃣ 可视化与 Power BI（Visualization & Power BI）**

### **Power BI 报表主要展示信息**

1. **总销售额按年份和月份**
2. **客户 ID 计数（按 Cluster\_Label 分类）**
3. **总销售额按支付方式（PaymentMethod）**
4. **个人总消费金额（按客户分类和 Recency）**
5. **Monetary 计数（按 ABC 分类）**
6. **客户 ID 计数（按 ABC 分类）**
7. **客户 ID 计数（按 DiscountApplied）**
8. **总销售额（按 DiscountApplied）**
9. **总销售额（按美国各州）**
    
![powerbi报表截图](https://github.com/ilovescho-O-olsomuch/retail-transaction/blob/main/%E6%95%B0%E6%8D%AE%E5%8F%AF%E8%A7%86%E5%8C%96powerbi.png)

### **选择的可视化图表**

- **折线图**：用于展示销售额的趋势（时间序列）
- **柱状图**：用于显示不同分类的销售额对比
- **散点图**：用于展示 Recency 与 Monetary 之间的关系
- **饼图**：用于展示 ABC 分类的客户分布
- **地图**：用于展示不同州的销售额分布

## **5️⃣ 业务价值（Business Value）**

### **如何优化营销策略？**

1. **高价值客户**：可推送 VIP 会员计划，提高客户忠诚度
2. **流失客户**：针对 Recency 高的客户发送再营销优惠券
3. **潜力客户**：提供个性化推荐，提高复购率

### **折扣策略优化**

- 低折扣（0%-10%）比高折扣更能带来长期收益
- A 类客户对折扣不敏感，但 B、C 类客户折扣响应度较高
- 需关注流失客户折扣较少的现象，并测试不同折扣策略的影响

## **6️⃣ 未来改进方向（Future Work）**

### **数据优化**

- 引入用户浏览行为、社交媒体互动数据，以更精准刻画用户画像
- 结合市场趋势数据，分析不同时间段的销售模式

### **模型优化**

- 进一步优化聚类模型，如引入 DBSCAN、层次聚类进行比较
  
### **业务实践优化**

- 设计 A/B 测试，评估不同折扣策略的实际效果
- 通过机器学习模型预测客户流失率，并采取预防措施




