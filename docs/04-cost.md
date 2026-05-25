# 04 · 成本测算

> ⚠️ 价格随 region / 优惠 / 时间变化，本表仅作量级参考，**以 [Microsoft Fabric pricing](https://azure.microsoft.com/pricing/details/microsoft-fabric/) 为准**。

---

## 1. Fabric 容量

| SKU | 月价（按需）| 月价（1 年 RI）| 适用场景 |
|-----|------------|----------------|----------|
| F2  | ~$262      | ~$155          | **本 demo 够用** |
| F4  | ~$525      | ~$310          | 团队 PoC |
| F32 | ~$4,205    | ~$2,475        | 中型生产 |
| F64 | ~$8,410    | ~$4,945        | 较大生产 |

**Demo 推荐**：60 天免费试用 → 跑完后立刻 Pause 容量（按秒计费，Pause = 0 元）

```bash
az fabric capacity suspend --capacity-name <name> --resource-group <rg>
```

---

## 2. Mirroring 费用

| 项 | 收费 |
|----|------|
| 镜像数据**存储** | **0**（微软承担） |
| 镜像数据**同步计算** | **0**（微软承担） |
| 查询镜像数据 | 按目标 Workspace 的 CU 计费 |

**关键**：Mirror 本身**完全免费**，只在你用 Notebook/SQL/Power BI 查询时才消耗 F-SKU 的 CU。

---

## 3. 源端费用

| 源 | 镜像引发的源端额外费用 |
|----|----------------------|
| Azure SQL | CDC 占少量 IO；忽略不计 |
| Snowflake | 每分钟 1 次轻查询，约 $0.001/次 → **月约 $40** |
| Cosmos DB | ChangeFeed 读 RU，启用 Continuous Backup 增加存储费约 20% |

---

## 4. OneLake 存储

| 项 | 单价 | demo 月用量 | 月费 |
|----|------|------------|------|
| OneLake 存储 | $0.026/GB/月 | 10 GB | ~$0.26 |

Mirror 的存储不计在 OneLake，所以基本可忽略。

---

## 5. Data Agent 查询费

每次自然语言提问会引发：
- LLM 生成 SQL/DAX：消耗少量 CU（约 1-2 CU 秒）
- 执行 SQL/DAX：按数据量消耗 CU
- 典型一次提问：**0.5-2 CU 秒** → F2 容量 24×60×60×2 CU 秒/天 = 172800 CU 秒/天 → 一天可以问几万次

---

## 6. demo 总成本估算

### 6.1 跑 4 小时 demo 一次

| 项 | 金额 |
|----|------|
| F2 容量 4 小时 | $0.36 |
| Snowflake 测试数据 + Mirror 拉取 | $1 |
| Azure SQL Basic 4 小时 | $0.01 |
| Cosmos DB Serverless 测试 | $0.50 |
| **合计** | **~$2** |

### 6.2 持续运行 1 个月（demo 环境）

| 项 | 金额 |
|----|------|
| F2 容量 24×7 月度 | $262 |
| Snowflake trial credit | $0（在 trial 内） |
| Azure SQL Basic | $5 |
| Cosmos DB Serverless 低用量 | $20 |
| OneLake 存储 10GB | $0.26 |
| **合计** | **~$290 / 月** |

### 6.3 中型生产模拟（F32 + 1TB 数据 + 1000 业务用户）

| 项 | 月度 |
|----|------|
| F32 容量（1 年 RI） | $2,475 |
| OneLake 存储 1TB | $26 |
| Azure SQL S3 | $150 |
| Snowflake 中等用量 | $500 |
| Cosmos DB Provisioned 4000 RU | $230 |
| **合计** | **~$3,400 / 月** |

对比传统方案（Databricks + ADF + Tableau + ADLS 4 份存储）类似规模 ~$8,000-12,000/月。

---

## 7. 节流建议

| 策略 | 节省 |
|------|------|
| **Pause 容量**（夜间 / 周末） | 50-70% |
| 用 **1 年 RI** 而非按需 | 40% |
| Notebook 调度避开高峰 | 平滑 CU 峰值，可降一档 SKU |
| Mirror 不需要的表去掉 | 减少查询 CU |
| Data Agent 设 daily quota | 防止意外暴涨 |

---

## 8. 告警

在 Azure Portal → Cost Management → 设置 budget alert：

```text
预算: $500/月
告警阈值: 50% / 80% / 100%
通知: 邮件 + Teams webhook
```

Fabric Portal → Capacity Metrics App → 监控 CU 使用率 > 80% 自动告警。

---

→ 演示完毕请按 [05-cleanup.md](05-cleanup.md) 清理资源
