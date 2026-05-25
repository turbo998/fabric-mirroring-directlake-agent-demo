# 03 · 一步步操作指南（5 小时）

> 全程在 Fabric Portal + Notebook 中完成；不写一行 IaC、不部署一行应用代码。

---

## Lab A · 建立三个 Mirror（45 分钟）

### A.1 Mirror Azure SQL Customers

1. Workspace → New → **Mirrored Azure SQL Database**
2. Connection: 输入 server / database / 凭据
3. 勾选 `Customers` 表 → **Mirror database**
4. 等 5-10 分钟，看到状态 = `Running` ✅

### A.2 Mirror Snowflake Orders

1. Workspace → New → **Mirrored Snowflake**
2. Account: `<id>.<region>.snowflakecomputing.com` / Warehouse / Database
3. 勾选 `RETAIL.PUBLIC.ORDERS`
4. 启动 → 等待初始化（10-20 分钟取决于数据量）

### A.3 Mirror Cosmos DB OrderItems

1. Workspace → New → **Mirrored Azure Cosmos DB**
2. 选 NoSQL 账号 / DB / Container
3. 启动

### A.4 验证三个 Mirror 都到位

```sql
-- 在 Fabric SQL Endpoint 直接查
SELECT COUNT(*) FROM [MirrorSQL].dbo.Customers;
SELECT COUNT(*) FROM [MirrorSnowflake].PUBLIC.ORDERS;
SELECT COUNT(*) FROM [MirrorCosmos].dbo.order_items_container;
```

数字与源端一致 = 成功。

---

## Lab B · 建 Gold Lakehouse 与 Notebook 拼数据（60 分钟）

### B.1 创建 Lakehouse

- Workspace → New → **Lakehouse** → 命名 `retail_gold`

### B.2 创建 Notebook

`Notebook nb_build_gold.ipynb` —— 复制粘贴执行：

```python
# Cell 1: 读三个 mirror
from pyspark.sql import functions as F

ws = "<workspace-id-or-name>"
sql_path  = f"abfss://{ws}@onelake.dfs.fabric.microsoft.com/MirrorSQL.Database/Tables/Customers"
sf_path   = f"abfss://{ws}@onelake.dfs.fabric.microsoft.com/MirrorSnowflake.Database/Tables/ORDERS"
cos_path  = f"abfss://{ws}@onelake.dfs.fabric.microsoft.com/MirrorCosmos.Database/Tables/order_items_container"

customers = spark.read.format("delta").load(sql_path)
orders    = spark.read.format("delta").load(sf_path)
items_raw = spark.read.format("delta").load(cos_path)

# Cell 2: 打平 Cosmos JSON
items = (items_raw
    .select("order_id", F.explode("items").alias("it"))
    .select("order_id",
            F.col("it.sku").alias("sku"),
            F.col("it.qty").alias("qty"),
            F.col("it.unit_price").alias("unit_price"),
            F.col("it.category").alias("category"),
            F.col("it.country_of_origin").alias("country_of_origin")))

# Cell 3: 维度 + 事实
dim_customer = (customers.select(
    "customer_id", "customer_name", "region", "tier"))

dim_product = (items.groupBy("sku", "category", "country_of_origin")
                    .agg(F.first("unit_price").alias("list_price")))

fact_orders = (orders.alias("o")
    .join(dim_customer.alias("c"), "customer_id", "left")
    .withColumn("order_date", F.to_date("placed_at"))
    .select("order_id", "customer_id", "region", "tier",
            "order_date", "amount", "order_status"))

fact_order_items = items.join(orders.select("order_id", "placed_at"), "order_id", "inner")

# Cell 4: 写回 Lakehouse Gold
for df, name in [(dim_customer, "dim_customer"),
                 (dim_product,  "dim_product"),
                 (fact_orders,  "fact_orders"),
                 (fact_order_items, "fact_order_items")]:
    (df.write.format("delta")
       .mode("overwrite").option("overwriteSchema", "true")
       .saveAsTable(f"retail_gold.{name}"))
```

### B.3 调度（可选）

Notebook → Schedule → Every 15 minutes，让 Gold 也跟随 Bronze 更新。
（生产用 Pipeline + Activator 触发更靠谱）

---

## Lab C · 建 DirectLake 语义模型 + Power BI 报表（45 分钟）

### C.1 自动建语义模型

- Lakehouse `retail_gold` → **New semantic model**
- 模式选择：**DirectLake**
- 勾选 4 张表：fact_orders / dim_customer / dim_product / fact_order_items
- 关系自动识别（按 customer_id / sku 主外键）

### C.2 加 DAX measures

```dax
Total Revenue =
    SUMX(
        FILTER(fact_orders, fact_orders[order_status] <> "returned"),
        fact_orders[amount]
    )

Return Rate =
    DIVIDE(
        CALCULATE(COUNTROWS(fact_orders), fact_orders[order_status] = "returned"),
        COUNTROWS(fact_orders)
    )

Revenue YoY =
    VAR _curr = [Total Revenue]
    VAR _prev = CALCULATE([Total Revenue], SAMEPERIODLASTYEAR(dim_date[Date]))
    RETURN DIVIDE(_curr - _prev, _prev)
```

### C.3 报表设计

- 标题：**全球零售实时总览**
- 卡片：今日订单数 / 今日营收 / 本月退货率
- 柱图：按区域营收
- 折线：日营收趋势 + YoY 对比
- 表：商品 Top 20 + Return Rate

### C.4 验证实时性

- 去 Azure SQL 改一条 Customer 的 `region`
- 等 30 秒
- Power BI 报表点刷新 → 该客户已分到新区域

---

## Lab D · 建 Data Agent + Ontology（60 分钟）

### D.1 创建 Ontology

Workspace → New → **Ontology** → `retail-business-ontology`

按 [01-architecture.md §4.3](01-architecture.md#43-ontology-关键定义节选) 节录扩展定义；
重点是绑定到 Gold 表的物理字段。

### D.2 创建 Agent

- Workspace → New → **Data agent** → `retail-insights-agent`
- Grounding: `retail_gold` Lakehouse + `retail_directlake_model` Semantic Model
- Linked ontology: `retail-business-ontology`
- Instructions 见 [learning-plan W7 §2.3](https://github.com/turbo998/microsoft-fabric-learning-plan/blob/main/deep-dives/w7-data-agents-deep-dive.md)

### D.3 测试

输入 5 个典型问题：
1. "今年华东咖啡机退货率多少？对比去年同期"
2. "本月营收 Top 5 客户"
3. "VIP 客户占总营收的多少？"
4. "意大利产品的退货率是否高于平均？"
5. "把华东咖啡机的报表打开给我"（验证 skill 调用）

---

## Lab E · MCP 接入 + 综合演示（30 分钟）

### E.1 启用 MCP endpoint

Agent → Settings → API → 启用 MCP endpoint
拿到 URL 并记录。

### E.2 用 M365 Copilot Chat 调用（可选）

如果租户开通 M365 Copilot：在 Copilot Studio 里 import MCP endpoint 注册为 plugin，在 Teams 里 @bot 提问。

### E.3 走通演示路径

完成 [00-overview.md §3.1](00-overview.md#31-5-分钟版本) 5 分钟版的全部 5 步。

---

## 排错速查

| 现象 | 可能原因 | 解决 |
|------|---------|------|
| Mirror 状态 = `Error` | 源端凭据 / CDC 未开 | 重新检查 §02 |
| Spark Notebook 慢 | Lakehouse 文件太小太碎 | `OPTIMIZE table_name` 合并 |
| DirectLake 模型显示 `Direct Query fallback` | 表未 V-Order 或行数过大 | 在 Lakehouse 跑 `VACUUM` + `OPTIMIZE` |
| Agent 答错 | Ontology 字段映射错 | 在 Ontology 测试器里手工验证概念绑定 |
| MCP 连接 401 | Bearer token 过期 | 重新 `az login --scope https://api.fabric.microsoft.com/.default` |

---

→ 演示完毕，记得看 [04-cost.md](04-cost.md) 和 [05-cleanup.md](05-cleanup.md) 别忘了关停！
