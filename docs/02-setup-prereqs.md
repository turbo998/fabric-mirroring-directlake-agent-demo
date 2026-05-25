# 02 · 前置准备

> 整个 demo 预计**首次跑通 4-5 小时**，其中 1-2 小时是账号 / 容量 / 测试数据准备。

---

## 1. 必备账号清单

| 项 | 推荐选项 | 备注 |
|----|---------|------|
| **Microsoft Fabric Capacity** | F2 / F4（试用 60 天免费）| 演示场景 F2 够，生产至少 F32 |
| **Power BI Service** | 含在 Fabric 容量中 | 业务用户需要 Pro license 才能看（也能用 Fabric F SKU 共享）|
| **Azure 订阅** | 任意支持 Fabric 的 region | 用 East US / West Europe / Southeast Asia |
| **Azure SQL Database** | Basic / S0 即可 | 存放客户主数据 |
| **Snowflake 账号** | Trial（30 天 $400 credit） | 存放订单事实 |
| **Cosmos DB 账号** | Serverless / 1000 RU | NoSQL API |
| **Entra ID 账号** | 全局管理员或 Fabric 管理员 | 用于授权 service principal |

---

## 2. 配额检查

### 2.1 Fabric 容量

- 在 Azure Portal → Microsoft Fabric → 创建 F2 容量（指定 region）
- 把当前用户加为 **Capacity Admin**
- 在 [app.fabric.microsoft.com](https://app.fabric.microsoft.com) 创建 Workspace，绑定该容量

### 2.2 Snowflake

- 启用 **CHANGE_TRACKING**（必须，否则 Mirror 失败）
  ```sql
  ALTER TABLE RETAIL.PUBLIC.ORDERS SET CHANGE_TRACKING = TRUE;
  ```
- 创建只读服务账号（避免用 ACCOUNTADMIN）
- 确保 Snowflake 账号能被 Azure 公网/PrivateLink 访问（演示常用公网 + IP 白名单）

### 2.3 Azure SQL

- 启用 SI 和 Change Tracking：
  ```sql
  ALTER DATABASE [retaildb] SET ALLOW_SNAPSHOT_ISOLATION ON;
  ALTER DATABASE [retaildb] SET CHANGE_TRACKING = ON
      (CHANGE_RETENTION = 3 DAYS, AUTO_CLEANUP = ON);
  ALTER TABLE Customers ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = ON);
  ```
- 配置防火墙：允许 Azure 服务访问 + 当前 IP

### 2.4 Cosmos DB

- 启用 **Continuous Backup**（Mirror 前提条件）
  ```bash
  az cosmosdb update --name <acct> --resource-group <rg> \
      --backup-policy-type Continuous
  ```

---

## 3. 测试数据准备

### 3.1 Azure SQL Customers（500 行）

```sql
CREATE TABLE Customers (
    customer_id INT PRIMARY KEY,
    customer_name NVARCHAR(100),
    region NVARCHAR(20),       -- 华东/华南/华北/华西
    tier NVARCHAR(20),         -- VIP/Gold/Silver/Regular
    email NVARCHAR(100),
    created_at DATETIME2 DEFAULT SYSUTCDATETIME()
);

-- 用任意脚本插入 500 行；推荐用 Python Faker 库
```

### 3.2 Snowflake Orders（10 万行）

```sql
CREATE TABLE RETAIL.PUBLIC.ORDERS (
    order_id BIGINT,
    customer_id INT,
    placed_at TIMESTAMP_NTZ,
    amount NUMBER(12, 2),
    order_status VARCHAR(20)   -- placed/shipped/delivered/returned
);
-- 用 Faker / SQL generate_series 灌 10 万行
```

### 3.3 Cosmos DB OrderItems（30 万条）

```json
{
  "id": "guid",
  "order_id": 123456,
  "items": [
    {"sku": "CM-001", "qty": 1, "unit_price": 1299.00,
     "category": "咖啡机", "country_of_origin": "意大利"},
    {"sku": "CP-022", "qty": 2, "unit_price": 49.00,
     "category": "水杯", "country_of_origin": "中国"}
  ]
}
```

> ⚠️ 真实跑通时建议先用 100 / 1000 / 10000 三档数据量梯度试，避免一次性灌爆。

---

## 4. 凭据收纳

强烈建议用 Azure Key Vault 或 Fabric 内置 Workspace Identity，不要明文存：

| 凭据 | 存放位置 |
|------|---------|
| Snowflake user/password | Fabric Workspace Identity → Snowflake role |
| Azure SQL 连接串 | Fabric Connection（Managed Identity 最佳） |
| Cosmos DB key | Fabric Connection（System Assigned MI 最佳） |

---

## 5. 网络

| 路径 | 推荐 |
|------|------|
| Fabric → Snowflake | 公网 + IP allowlist（Snowflake 端） |
| Fabric → Azure SQL | "Allow Azure services" + 私有终结点（可选） |
| Fabric → Cosmos DB | "Allow Azure services" |
| 用户 → Fabric | 公网 + Entra ID SSO |

生产环境建议全部走 Private Endpoint + VNet Data Gateway，但演示阶段可以先公网过 PoC。

---

## 6. 自检清单

- [ ] Fabric F2 容量已创建并绑定 Workspace
- [ ] Snowflake 测试库 + 服务账号 + CHANGE_TRACKING 已就绪
- [ ] Azure SQL 库 + 测试表 + Change Tracking 已就绪
- [ ] Cosmos DB 容器 + Continuous Backup 已开启
- [ ] 上述三源数据量符合预期
- [ ] 凭据收纳到 Key Vault / Fabric Connection
- [ ] 用户拥有 Workspace Admin + Capacity Admin 双权限

---

→ 准备好后进入 [03-step-by-step.md](03-step-by-step.md)
