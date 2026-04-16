# 懒猫书签同步系统技术方案 v3.0 评审报告

**评审者:** Composer2  
**评审日期:** 2026-04-17  
**评审对象:** `TECHNICAL-DESIGN-v3.md`（v3.0，基于 PRD v6.0）  
**对比基线:** v2.0 及五份 v2 审计结论（Composer2 / GLM-5 / GLM-5.1 / Kimi Code / Kimi K2.5）

---

## 执行摘要

v3.0 在 v2 基础上做了**系统性收敛**：独立「同步协议」章节、PromQL 修正、`sync_records` 与 PRD 对齐、冲突检测与截断/快照伪代码补全、前后端工具链（pnpm / workspace）与部署细节（锁文件、WAL）等，**整体已达到可进入 Phase 1 开发的成熟度**。

评审结论：**无阻塞开发的 P0 级文档缺陷**；仍存在若干 **P1（实现前必须拍板或补文档）** 与 **P2（实现阶段校准）** 项，建议在编码前将 P1 写入「设计决议」或 PRD/附录，避免各模块各自理解。

| 级别 | 数量 | 说明 |
|------|------|------|
| P0 | 0 | — |
| P1 | 4 | 协议与数据模型一致性、伪代码可落地性、链接与路径 |
| P2 | 8 | 监控完备性、边界条件、SQL 可移植性、表述细化 |

---

## 已确认的有效改进（相对 v2）

以下项与 v3「修订摘要」一致，评审认为**方向正确、对前期审计有实质回应**：

1. **第 4 章同步协议**：错误码、410 GONE、幂等与 `client_request_id` 集中描述，降低实现分叉风险。  
2. **冲突检测**：GUID 去重、批量最新变更子查询、同批 `parent_local_id` create 冲突、`classifyConflict` 五类枚举，逻辑闭环较 v2 完整。  
3. **版本与并发**：`NextVersionSQLite` 的 `FOR UPDATE`、`AcquireLock` 指数退避与调试日志，表述清晰。  
4. **数据层**：`sync_id` / `request_id`、`audit_log`  BIGINT 说明、GORM `datatypes.JSON` 选型与 SQLite 3.35+ 前提。  
5. **运维与质量**：Dockerfile `pnpm-lock.yaml`、`NormalizeWeights` 事务内写入 `change_log`的意图、集成测试统一 pnpm。  

---

## P1 — 实现前建议闭合的问题

### P1.1 同步响应结构与「先推后拉」的语义仍略模糊

**位置:** §4.2.2 `SyncResponse`；§4.1 协议概述  

**现象:**  
- `data.new_version` /顶层 `version` / `changes` 增量之间的关系未严格定义（例如：部分应用、部分冲突时，`applied_changes` 与 `new_version` 是否始终对应「已成功提交的 max to_version」）。  
- `success: boolean` 与 HTTP 200 + `CONFLICT` 行（§4.3）的**响应体形状**未统一说明（是靠 `success` 还是靠业务错误码枚举字段）。  

**建议:** 在 §4.2 增加一小节「响应不变式」（invariants）：无冲突 / 部分冲突 / 全冲突 / 幂等命中四种情况下，`version`、`new_version`、`changes`、`conflicts` 各自如何填充；并与插件侧 `last_known_version` 更新规则交叉引用（可指向 PRD 对应小节）。

---

### P1.2 `NormalizeWeights` 伪代码与 §2.1 JSON 类型约定不一致且存在未定义变量

**位置:** §6.3  

**问题:**  
1. `Change.Data` 在 §2.1 与冲突检测 structs 中倾向 `datatypes.JSON` / 强类型，伪代码仍使用 `map[string]interface{}` 且 `FromVersion`/`ToVersion` 为 0，与「版本单调」「可重放 change_log」目标不一致。  
2. `values = append(values, guids...)` 中的 **`guids` 未在片段中定义**（应为子节点 GUID 列表，与 `CASE guid WHEN ...` 的 IN 子句一致）。  

**建议:** 伪代码中显式：`childGUIDs := ...`，`change.FromVersion`/`ToVersion` 由事务内 `NextVersion`（或文档声明的等价物）填充；`Data` 字段与 GORM 模型对齐，避免读者照抄后无法编译或与迁移不一致。

---

### P1.3 `TruncateChangeLog` 的「按条数截断」语义需数学化说明

**位置:** §6.5  

**现象:** 使用 `ORDER BY to_version ASC` + `OFFSET(maxCount)` 选取截断边界，在**版本号非连续**、或存在 `reindex`/`rollback` 等类型时，**保留条数与 `maxCount` 的精确关系**不直观；且与 §3.1中 `change_log` 类型集合的过滤条件需长期保持一致（否则截断与基线会漂移）。  

**建议:** 用 2～3 句说明设计意图（例如：在「仅参与同步重放」的类型子集上保留最近 N 条，或保留 `to_version >=某阈值` 的所有行）；必要时给出「最坏情况下多删/少删」的边界，避免实现与运维对 `baseline_version` 理解不一致。

---

### P1.4 §12 参考链接路径与仓库布局不一致

**位置:** §12 参考文档  

**事实:** `TECHNICAL-DESIGN-v3.md` 位于仓库 `tech/` 目录下，与 `PRD-v6.md`（在 `prd/`）、各 `tech-design-v2-reviewed-*.md`（与 v3 同目录）并列。  

**问题:**  
- `./PRD-v6.md` 在 `tech/` 内解析会失败，应为 `../prd/PRD-v6.md`（或仓库约定的绝对文档路径）。  
- `./tech/tech-design-v2-reviewed-by-*.md` 会解析为 `tech/tech/...`，**多一层 `tech/`**。  

**建议:** 统一改为同目录 `./tech-design-v2-reviewed-by-*.md` 与 `../prd/PRD-v6.md`，或在 monorepo 根增加 `docs/` 入口页做跳转。

---

## P2 — 建议优化（不阻塞启动，但应在迭代中消化）

### P2.1 `SyncErrorRate` 在「零流量」时的 PromQL 行为

**位置:** §9.2  

`sum(rate(...error...[5m])) / sum(rate(...[5m]))` 在分母为 0 时，Prometheus 通常不产生有效样本或行为依赖配置。若需静默「无数据即不告警」，可补充 `and on() sum(rate(...[5m])) > 0` 或使用 `ignoring` 与默认向量等惯用法（实现时由可观测性负责人定稿）。

---

### P2.2 `lzc_version_gaps_total` 仅定义指标未定义产生条件

**位置:** §9.1  

建议在 §6.2 或 §11「版本号空洞」风险处，用一句话说明**何时递增**该 Counter（例如：检测到 `to_version` 序列异常、或后台审计任务发现空洞），否则指标易沦为「永远为 0」的摆设。

---

### P2.3 `NextVersion` 与 `NextVersionSQLite` 的选用策略

**位置:** §6.2  

文档同时给出 `RETURNING` 路径与 `FOR UPDATE` 路径，但未明确 **modernc.org/sqlite** 下默认推荐哪条、何种配置降级。建议一句决策表：单库 SQLite 用哪 API、PostgreSQL 用哪 API、是否始终走事务封装。

---

### P2.4 冲突检测中对「仅带 `local_id` 的 create」与空 GUID的路径

**位置:** §6.1 `Detect` 主循环  

对 `change.GUID == ""` 的 create，当前片段主要依赖 `createByParent` 分支；与服务器 `latestMap` 的交叉（例如远程已为同一逻辑节点分配 GUID）依赖 PRD 的 ID 映射规则。建议在 §4 或 §6 加**指回 PRD** 的一句话，避免实现阶段遗漏。

---

### P2.5 `GenerateSnapshot` 中快照清理 SQL 的可移植性

**位置:** §6.6  

`DELETE ... WHERE version NOT IN (SELECT ... ORDER BY ... LIMIT 500)` 在不同 SQLite 版本与 PostgreSQL 上对派生表/子查询限制不完全一致。实现时建议改为「先查保留列表再删除」两步或使用窗口函数（若 PG 为主），文档可标注「伪代码，生产需按目标库改写」。

---

### P2.6 `conflicts` 表 DDL 末尾逗号

**位置:** §3.1`resolved_by TEXT,` 后接 `);` 属于**尾随逗号**。SQLite 新版本普遍接受，若需兼容旧环境可去掉逗号。属低优先级一致性细节。

---

### P2.7 `classifyConflict` 中 `move` 与 `update` 的组合边界

**位置:** §6.1  

当一方为 `move`、另一方为 `update`（或 `reindex` 与服务端内部变更）时，当前 `switch` 逻辑落入默认 `update_conflict`。是否在业务上应归为 `move_conflict` 或单独类型，建议依赖 PRD 的冲突产品定义，文档可列一条「未显式列出之组合 → 默认 update_conflict」。

---

### P2.8 里程碑与测试策略的「可验证标准」

**位置:** §8、§10  

§8 中测试多为占位；建议在 Phase 4 行项中增加**最小验收条目**（例如：「部分冲突 + 重试 + 幂等同一 `client_request_id`」必须通过），与 §4协议不变式呼应，减少后期补测试成本。

---

## 与 v3 文首「状态」的核对

文首写「**已评审，待开发**」，而 §12 仍写「评审报告：见 `./tech/` 目录下 5 份审计报告」——在修正 P1.4 链接后，建议改为「v2 审计报告见同目录」并**预留本 v3 评审报告链接**（`tech-design-v3-reviewed-by-composer2.md`），使文档闭环。

---

## 总结| 维度 | 评价 |
|------|------|
|完整性 | 优：协议、schema、算法、部署、监控、里程碑齐全 |
| 一致性 | 良：伪代码与类型/版本/链接有少量需对齐处（P1.2、P1.4） |
| 可实施性 | 良：补齐响应不变式与截断语义后，可直接驱动拆分任务与接口契约 |
| 风险 | 低～中：主要残留风险在边界条件与可观测性细节，而非架构方向 |

**综合建议:** **批准以 v3.0 为开发基线**，在 Sprint 0 关闭 P1.1～P1.4（或等价的设计评审纪要），P2 按模块负责人在实现 PR 中逐项消化。

---

**文档状态:** 评审完成（Composer 2）  
**下一步:** 修正 §12 相对路径；可选将本节链接回写至 `TECHNICAL-DESIGN-v3.md` §12/§13
