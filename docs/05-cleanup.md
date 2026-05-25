# 05 · 清理资源

> Demo 跑完后**务必清理**，否则 F-SKU 容量按秒持续计费。

---

## 1. 立即"暂停"（如还想保留环境）

```bash
# Fabric 容量暂停 = 0 元
az fabric capacity suspend \
    --capacity-name <fabric-cap-name> \
    --resource-group <rg-name>
```

Snowflake：

```sql
ALTER WAREHOUSE COMPUTE_WH SUSPEND;
```

Azure SQL Basic 本就便宜，可不动；如果想关，把 DTU 改成 Basic 5：

```bash
az sql db update --resource-group <rg> --server <srv> \
    --name retaildb --service-objective Basic
```

---

## 2. 彻底清理（保险起见）

### 2.1 Fabric 端

1. Workspace 设置 → Delete workspace（会删除所有 Lakehouse、Mirror、Agent、Report）
2. Fabric capacity 设置 → Delete capacity

### 2.2 源端清理

```bash
# Azure SQL
az sql db delete -g <rg> -s <srv> -n retaildb --yes
az sql server delete -g <rg> -n <srv> --yes

# Cosmos DB
az cosmosdb delete -g <rg> -n <acct> --yes

# 整个 RG（最保险）
az group delete -n <rg> --yes --no-wait
```

### 2.3 Snowflake

```sql
DROP DATABASE RETAIL;
DROP WAREHOUSE COMPUTE_WH;
DROP USER fabric_mirror_user;
DROP ROLE fabric_mirror_role;
-- 如果是 trial 账号到期会自动失效
```

---

## 3. 验证清理

| 检查项 | 命令 / 路径 |
|--------|------------|
| Fabric 容量已删 | Azure Portal → All resources, 过滤 type = Microsoft.Fabric/capacities |
| 资源组已空 | `az group show -n <rg>` 返回 NotFound |
| Cost Management 已无新增 | 第二天看 Cost analysis，应无新 charge |
| Snowflake 已无 warehouse 运行 | Snowsight → Warehouses 全 Suspended |

---

## 4. 故障恢复（万一删错）

| 误删 | 恢复 |
|------|------|
| Fabric 容量 | 重新创建（命名变化不影响 trial） |
| Workspace（含 Lakehouse） | **不可恢复** —— 重新建并跑 Lab B 即可（demo 数据可重灌）|
| Azure SQL DB | 启用了 PITR 可恢复 7 天内任意点 |
| Snowflake DB | UNDROP 7 天内有效：`UNDROP DATABASE RETAIL;` |

---

## 5. 离场清单

- [ ] Fabric capacity = Suspended 或 Deleted
- [ ] Azure SQL DB 已删或缩 Basic
- [ ] Cosmos DB 已删
- [ ] Snowflake warehouse Suspended，DB 已删（如 trial）
- [ ] Resource Group 已删（如使用了独立 RG）
- [ ] Cost Management 看次日 0 新增 charge
- [ ] 凭据从本地 / Key Vault 移除

---

🎉 **Demo 完成！**

下一步推荐：
- 把这个架构介绍给团队 / 客户
- 继续刷 [microsoft-fabric-learning-plan](https://github.com/turbo998/microsoft-fabric-learning-plan) W8（RTI 综合项目）
- 把语音前端接入：[contact-center-on-gpt-realtime](https://github.com/turbo998/contact-center-on-gpt-realtime)
