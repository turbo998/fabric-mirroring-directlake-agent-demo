# Fabric Mirroring + DirectLake + Data Agent Demo

> **零 ETL + 实时 BI + 自然语言查数据** —— 用 Microsoft Fabric 三件套，把多源订单与客户数据
> 在 5 个小时内拼成一个业务可用的"实时智能零售看板 + 数据助手"。
>
> **本仓库当前只包含规划与操作文档**（Spec only），不含可执行代码 / IaC，
> 旨在作为后续动手实现或客户 PoC 的蓝本。

---

## 🎯 业务场景一句话

> "我希望我的销售经理在 Power BI 上看到 Snowflake 里的订单和 Azure SQL 里的客户合并后的实时表现，
> 并能用一句中文问'今年华东咖啡机退货率'就拿到答案。"

---

## 🏗️ 架构概览

```
Azure SQL DB      ──┐
(Customers)         │  Mirroring (零 ETL, < 1 min)
                    │
Snowflake           ├──> OneLake Delta ──> Lakehouse Gold ──> Semantic Model
(Orders)            │                       (Spark JOIN)       (DirectLake)
                    │                                                │
Cosmos DB           │                                                ├──> Power BI 报表
(OrderItems JSON)   ──┘                                              │
                                                                     └──> Data Agent
                                                                          (中文自然语言查询)
```

详细 Mermaid 图见 [docs/01-architecture.md](docs/01-architecture.md)。

---

## ✨ 三大亮点

| 亮点 | 解决什么传统痛点 |
|------|------------------|
| **Mirroring** | 不再为多源数据写 ETL；存储不收费 |
| **DirectLake** | 报表零导入零刷新；底层数据变化秒级反映 |
| **Data Agent** | 业务用户不再排队等 BI 工程师写 SQL |

---

## 📚 文档导航

| 文档 | 内容 |
|------|------|
| [00-overview.md](docs/00-overview.md) | 项目目标、目标读者、收益预期 |
| [01-architecture.md](docs/01-architecture.md) | 架构图、数据流、组件职责 |
| [02-setup-prereqs.md](docs/02-setup-prereqs.md) | Foundry 容量、Snowflake、Azure SQL、Cosmos DB 准备 |
| [03-step-by-step.md](docs/03-step-by-step.md) | 5 小时一步步操作指南（Lab A-E） |
| [04-cost.md](docs/04-cost.md) | 容量、Mirroring、存储、查询计费明细 |
| [05-cleanup.md](docs/05-cleanup.md) | 资源关停清单避免账单失控 |

---

## 🚦 状态

| 项 | 状态 |
|----|------|
| 文档 / Spec | ✅ Ready |
| 真实跑通验证 | ⬜ 待用户在 Fabric 容量上实操 |
| IaC（Bicep/Terraform）| ⬜ Out of scope |
| 自动化脚本 | ⬜ Out of scope |

---

## 🔗 关联仓库

- **学习路径**：[microsoft-fabric-learning-plan](https://github.com/turbo998/microsoft-fabric-learning-plan) — 8 周从 0 到精通 Fabric
- **AI 语音 Demo**：[contact-center-on-gpt-realtime](https://github.com/turbo998/contact-center-on-gpt-realtime) — GPT-realtime 三模型联动

---

## 📄 License

MIT — see [LICENSE](LICENSE).

## 🙏 Acknowledgements

Microsoft Fabric · Power BI · Azure · Snowflake · FabCon 2026 community
