# TECHNICAL-DESIGN-v2.md 审计报告

**审计对象:** `TECHNICAL-DESIGN-v2.md`（基于 PRD v6.0，v2.0 / v2.1 修订）  
**审计人:** Composer 2  
**审计日期:** 2026-04-16  
**结论摘要:** v2 相对 v1 在驱动选型、CGO/SQLite、版本递增、归一化查询、冲突表外键、Web 栈、扩展权限、健康检查与指标标签等方向**明显改进**，变更摘要与正文大体一致。仍有几处**算法/协议/工具链**层面的不一致与 PromQL 细节问题，建议在开发契约（OpenAPI 或「同步协议」专节）中收口后再标为「冻结」。

---

## 1. v1 评审项的跟进情况（摘要）

| v1 审计主题 | v2 状态 |
|-------------|---------|
| CGO_ENABLED=0 与 SQLite | 已改为 `modernc.org/sqlite`，与 `CGO_ENABLED=0` 一致 |
| NextVersion 竞态 | 已给出 `UPDATE … RETURNING` 及事务版 `NextVersionSQLite` |
| AcquireLock goroutine | 已改为 `TryLock` + 轮询 |
| NormalizeWeights JSON 方言 | 已改为 `guid` 列 + 事务包裹 |
| conflicts 外键与截断 | 已改为存完整 Change JSON，并说明避免截断断引用 |
| Web 栈缺失 / React vs Preact | 已拆分：Web 用 React + shadcn；插件用手写组件 + Tailwind |
| JWT 孤儿配置 | 已删除 `JWT_SECRET` |
| host_permissions 过宽 | 已改为示例域名 + `optional_host_permissions` |
| Docker HEALTHCHECK | 已 `apk add wget` |
| Prometheus device_id 基数 | 已移除该 label |
| 告警 rate 单位 | 高冲突率已改为约 0.167/s |
| 多实例 mutex | 正文已注明单进程限制及 PG advisory lock 方向 |

---

## 2. 严重 / 高优先级问题

### 2.1 同批次多条变更指向同一 GUID 时，冲突可能重复

`Detect` 对 `changes` **逐条** `append` 冲突；若客户端在一次 sync 里对同一 `guid` 打包了多条未合并的变更（异常或边界实现），会生成**多条 Conflict 结构**，`guid` 重复，后续 UI/落库可能重复或语义混乱。

**建议:** 在检测循环内对 `guid` 去重（保留「与 remote 对比」所需的一条本地代表变更），或在协议中**禁止**同批重复 `guid` 并写明服务端返回 400。

### 2.2 批量「最新变更」查询依赖遍历顺序，文档未证明不变量

当前实现：先取 `guid IN (…) AND to_version > ?`，按 **`to_version` 全局降序** 扫描，用 `latestMap` 每 `guid` **首次出现**作为最新。这在数学上**等价于**每组 `guid` 的 `max(to_version)`，结论成立，但对读者不显然，且未来若有人改成 `Find` 无序或加 `LIMIT` 会静默出错。

**建议:** 在文中加一句不变量说明，或改为显式 SQL：`SELECT guid, MAX(to_version) … GROUP BY guid` 再 JOIN 回表取行，可读性更好。

### 2.3 `classifyConflict` 与类型枚举不一致

`Conflict.Type` 注释含 `create_conflict`，但 `classifyConflict` 分支**从未返回** `create_conflict`；`rename_conflict` 与「URL 变更」等也未区分。

**建议:** 与 PRD 冲突类型表对齐，删掉未使用枚举或补全分支与测试。

### 2.4 同步协议仍散落在算法与测试之间§7.1 写明「单条变更没有 `from_version`、用 `BaseVersion`」，但 `change_log` 表仍含 **`from_version NOT NULL`**。这是合理的（服务端写入时填充），但全文**没有独立「API / 同步协议」**小节：`SyncRequest`/`SyncResponse` 字段、`NewVersion` 与全局 `Version` 语义、部分应用与重试规则仍仅靠测试片段表达。

**风险:** 前后端与第三方客户端实现分叉。

**建议:** 增加 **§5.0 或附录：同步 API 与字段语义**（可与 PRD 交叉引用）。

### 2.5 `SyncErrorRate` PromQL 仍可能不合法或难告警

表达式：

```text
rate(lzc_sync_requests_total{status="error"}[5m]) 
/ (rate(lzc_sync_requests_total[5m]) > 0)
```

在 Prometheus 中，与布尔向量相除在**总流量为 0** 时仍可能出现 **除零 / Inf**，且「`> 0` 作为分母」非常规。业界更常见：`sum(rate(...error...[5m])) / sum(rate(...[5m]))`，并配合 `and on() (sum(rate(total[5m])) > 0.01)` 或 `clamp_min(denom, epsilon)`。

**建议:** 改写为文档中可复制的、经 `promtool check rules` 验证的写法。

### 2.6 `HighConflictRate` 与多维 label

`lzc_conflicts_total` 带 `type` label时，`rate(lzc_conflicts_total[5m])` 会按 **每条时间序列** 计算；阈值「10 次/分钟」通常指**全站合计**。

**建议:** 使用 `sum(rate(lzc_conflicts_total[5m])) > 0.167`（若语义为总和），并在注释中写清。

---

## 3. 中优先级问题

### 3.1 Dockerfile 前端与 pnpm 锁文件

Web 构建使用 `pnpm install --frozen-lockfile`，但仅 `COPY web/package*.json ./` **未必**复制 `pnpm-lock.yaml`，冻结安装会失败或不可复现。

**建议:** 显式 `COPY web/package.json web/pnpm-lock.yaml ./`（若锁文件在仓库中）。

### 3.2 集成测试脚本与包管理器不一致

§7.3 仍写 `cd extension && npm run build`，与全文 **pnpm 统一**矛盾。

**建议:** 改为 `pnpm install && pnpm run build`，或注明 monorepo 根目录命令。

### 3.3 `sharedConfig` 表述偏虚

§4.2「通过 Vite 的 `sharedConfig` 配置共享代码」——Vite 并无名为 `sharedConfig` 的一等 API，易误导。

**建议:** 改为 **pnpm workspace / path alias /独立 `packages/shared` 包** 等可执行方案。

### 3.4 `optional_host_permissions` 与 `<all_urls>`

在用户授予后权限面仍极大；与「最小权限」目标需平衡。文档已说明动态申请，**可接受**，但建议在安全小节强调：**默认仅允许用户配置的 origin**，`<all_urls>` 作为高级选项并提示审核与风险。

### 3.5 `audit_log` 与 PostgreSQL 迁移

SQLite 使用 `AUTOINCREMENT`，PostgreSQL 需 `GENERATED`/`SERIAL` 等；§3.2 未点名 `audit_log`，迁移时易漏。

### 3.6 `sync_records` 与 `change_log` 的对应关系

已新增 `sync_records`，但未描述：**一次 HTTP sync** 对应一行还是多行、与 `client_request_id` 幂等如何协同、失败回滚是否写入。

**建议:** 两三句数据流说明即可，避免实现各写各的。

### 3.7 术语「Go 1.23+ LTS」

Go 的版本策略与「LTS」常见含义不完全一致，对外文档可写「1.23+（团队约定）」以免争议。

---

## 4. 低优先级 / 笔误

- §7.2 插件测试仍调用私有方法 `mergeExcess`、`getAll`，以及占位符 `'ch_001', ..., 'ch_008'`——作为**意图示例**可以，但标注「伪代码 / 需导出或包内测试」可减少误读。
- `NextVersion` 的 `Scan` 目标为 `state.CurrentVersion`，需保证 GORM `Raw` 扫描列名与驱动行为一致（小问题，实现时注意）。
- 文档状态写「已评审，待开发」，与顶栏一致；若内部有正式评审记录，可链到会议纪要或 checklist（可选）。

---

## 5. 亮点（值得保留）

- **v1→v2 变更摘要**便于 diff 与责任追溯；修订历史表清晰。
- **插件不用 content script**、书签仅 `chrome.bookmarks` 的路径明确，降低复杂度。
- **审计日志表**与 PRD §9.3 对齐，利于合规与排障。
- **书签物理删除不依赖 FK 级联**的注释，符合树形业务。
- **归一化节流**与「reindex 是否计入条数上限」在环境变量说明中有补充，比 v1 清晰。

---

## 6. 修订优先级建议

| 优先级 | 项 |
|--------|-----|
| P0 | 同步协议专节（请求/响应/版本语义）；同批 `guid` 去重或禁止；PromQL 分母与 `sum(rate(...))` 修正 |
| P1 | `classifyConflict` 与冲突类型枚举一致；Dockerfile 复制 `pnpm-lock.yaml`；集成测试改为 pnpm；`sync_records` 写入规则 |
| P2 | 批量最新变更 SQL 可读性；shared 目录工程化表述；插件测试示例标注为伪代码 |

---

## 7. 总结

`TECHNICAL-DESIGN-v2.md` 已实质性吸收 v1 及多份评审中的工程向问题，**可作为 Phase 1 启动基线**；剩余问题主要集中在**同步协议形式化**、**告警查询的精确 PromQL**、**少量工具链与测试片段一致性**，以及 **冲突检测在边界输入下的去重**。处理完 P0/P1 后，文档对实现与联调的约束力会显著增强。

---

*本报告针对 `TECHNICAL-DESIGN-v2.md` 自身一致性与可实施性；若需与 `PRD-v6.md` 做逐条需求追踪（RTM），建议在 v2 中增加独立章节或表格引用 PRD 小节编号。*
