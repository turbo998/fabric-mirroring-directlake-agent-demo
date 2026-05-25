# 00 · 项目总览

## 1. 我们要解决的问题

某跨境零售公司：

- **客户**主数据在 **Azure SQL DB**（业务系统）
- **订单**事实数据在 **Snowflake**（历史数据仓库）
- **订单明细**（含商品 JSON、个性化配置）在 **Cosmos DB**（高并发写入）
- **销售经理**抱怨：
  1. 现有报表延迟 4 小时（ADF pipeline + Power BI Import 模式刷新）
  2. 多次跨平台 JOIN 数据对不上
  3. 想问"今年华东咖啡机退货率"得提工单等 3 天

**目标**：把以上痛点用 Fabric **5 小时 + 0 代码 + 0 ETL** 解决掉。

---

## 2. 目标读者

| 读者 | 收益 |
|------|------|
| **数据架构师** | 看到 Fabric 三件套如何串成零 ETL 架构 |
| **BI/数据分析师** | 摆脱"刷新失败 / 报表慢"困境 |
| **业务负责人** | 看到自然语言查数据的最终用户价值 |
| **方案售前** | 拿到一份可演示给客户的"前后对比" |
| **Fabric 学习者** | 完成 [learning plan](https://github.com/turbo998/microsoft-fabric-learning-plan) W6+W7 的综合 lab |

---

## 3. 演示能讲的故事

### 3.1 5 分钟版本

> 1. 打开 Power BI 报表，看到全国销售实时仪表板
> 2. 在 Snowflake 端修改一条订单状态 → 30 秒后 Power BI 自动反映
> 3. 在 Azure SQL 新增一个客户 → 30 秒后报表的"新客户数"+1
> 4. 在 Power BI 旁开 Data Agent，输入"今年华东咖啡机退货率"→ 5 秒内返回答案
> 5. 追问"对比去年同期" → Agent 自动算同比，附图

### 3.2 30 分钟深度版本

加上：
- 后台展示 Mirror 配置界面 + 监控（延迟、行数）
- 在 Notebook 里直接 `spark.read.format("delta")` 读镜像表
- 展示 Fabric IQ Ontology 定义"退货率"的过程
- Agent 调用 Power BI 报表 skill 自动打开下钻

---

## 4. 与"传统方案"的硬对比

| 维度 | 传统（Databricks + ADF + Tableau） | Fabric 三件套 |
|------|------------------------------------|---------------|
| 数据集成代码 | 3 个 ADF pipeline + 1 个 Auto Loader job | **0** |
| 数据存储份数 | 4 份（源 + ADLS Raw + Stage + Lakehouse） | 1.5 份（源 + 镜像 Delta，物理 1 份） |
| 报表延迟 | 4 小时 | < 1 分钟 |
| 业务用户自助查询 | 不可能 / 需 BI 排期 | 自然语言直查 |
| 团队规模 | 3-5 人（DE + BI + DBA + ML） | 1-2 人 |
| 月度成本（中型场景） | ~$20K | ~$10K |
| 落地时间 | 2-3 个月 | 1 周内 |

---

## 5. 学到 / 验证的能力

完成本 demo 你能验证：

- [ ] Mirroring 在生产强度的真实延迟
- [ ] DirectLake 在大表查询时的响应（vs Import / DirectQuery）
- [ ] Data Agent 在复杂业务问题上的命中率
- [ ] Fabric IQ Ontology 对 Agent 准确度的提升幅度
- [ ] 三件套组合后，业务用户从"提需求"到"得到答案"的时长改进

---

## 6. 不在范围

- ❌ 真实 PII 处理 / 合规 / 加密深度配置（仅占位说明）
- ❌ DR / 跨区域复制
- ❌ MLOps / 模型训练
- ❌ 实时点击流（如要扩展，参考 fabric-learning-plan W8）
- ❌ 自动化 IaC 部署脚本

---

## 7. 下一步

→ 阅读 [01-architecture.md](01-architecture.md) 了解组件细节，
→ 或直接跳 [03-step-by-step.md](03-step-by-step.md) 开始动手。
