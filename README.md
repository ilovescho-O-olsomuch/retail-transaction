# Retail-transaction

## 数据集文件格式
CSV格式

## 数据结构 Columns:
1. CustomerID: Unique identifier for each customer. ->顾客的唯一认证
2. ProductID: Unique identifier for each product. ->产品的唯一认证
3. Quantity: The number of units purchased for a particular product. ->产品购买数量
4. Price: The unit price of the product. ->产品的单价
5. TransactionDate: Date and time when the transaction occurred. ->购买时间
6. PaymentMethod: The method used by the customer to make the payment. ->购买方式
7. StoreLocation: The location where the transaction took place. ->产品购买地点
8. ProductCategory: Category to which the product belongs. ->产品所属类别
9. DiscountApplied(%): Percentage of the discount applied to the product. ->产品的折扣率
10. TotalAmount: Total amount paid for the transaction. ->总消费额

## 数据分析的基本流程
1. 数据清洗 使用工具：mysql（执行） 和 python(jupyter lab)（检查清洗效果）
2. 客户分层 依据：RFM 和 K-MEANS 使用工具：jupyter lab的scikit-learn, seaborn库
3. 进阶分层 ABC 使用工具：jupyter lab 的 numpy库
4. 数据可视化（客户画像） 使用工具：POWER BI + DAX 公式
