# 技术方案审计报告

**审计对象：** `TECHNICAL-DESIGN-v1.md`  
**对照基线：** `PRD-v6.md`  
**审计人：** Kimi Code  
**审计时间：** 2026-04-16  
**结论：** 方案整体可行，但存在 **5 项严重缺陷**、**7 项中等风险** 和 **6 项建议优化**，建议在进入开发前修正。

---

## 一、严重缺陷（必须修复）

### 1. 冲突检测算法使用了不存在的字段 `change.FromVersion`
**位置：** §5.1 `ConflictDetector.Detect`

**问题：** 代码中使用了 `change.FromVersion` 作为冲突判定依据：
```go
err := d.db.Where("guid = ? AND to_version > ?", change.GUID, change.FromVersion)...
if change.FromVersion < latestChange.ToVersion { ... }
```
但根据 PRD §6.3.2，`changes` 数组中的单条变更**没有 `from_version` 字段**，冲突检测应使用请求级别的 `base_version`（即 `Detect` 函数签名中的 `baseVersion int64`）。

**影响：** 该函数无法编译或逻辑错误，导致冲突检测完全失效。

**修复建议：** 将 `change.FromVersion` 全部替换为 `baseVersion`。

---

### 2. `VersionManager.NextVersion` 存在并发竞态条件
**位置：** §5.2 `VersionManager.NextVersion`

**问题：** 先执行 `UPDATE` 再执行 `SELECT` 读取新值，两步之间没有事务锁定：
```go
m.db.Exec(`UPDATE global_state SET current_version = current_version + 1 WHERE id = 1`)
m.db.First(&state, 1) // 竞态窗口
```

**影响：** 在高并发或 PostgreSQL 多连接场景下，可能返回重复的版本号，导致 `change_log.to_version` 唯一约束冲突或同步乱序。

**修复建议：**
- SQLite 3.35+ / PostgreSQL：使用 `UPDATE ... RETURNING current_version` 原子返回新值。
- 或包裹在 `BEGIN` + `SELECT FOR UPDATE` 事务中。

---

### 3. `conflicts` 表设计不符合 PRD，且存在变更日志截断后数据丢失风险
**位置：** §3.1 Schema 中的 `conflicts` 表

**问题：** 技术方案将冲突记录存储为 `local_change_id` / `remote_change_id`（外键指向 `change_log`）。但 PRD §7.4 明确要求冲突记录存储完整的 `local_change: Change` 和 `remote_change: Change` 对象。

**影响：**
- `change_log` 按 PRD §4.2 策略会在 7 天 / 1000 条后截断。一旦被引用记录被删除，`conflicts` 表将出现悬空外键，冲突解决 UI 和 API 将无法展示冲突详情。
- 违反 PRD 数据模型定义。

**修复建议：** 在 `conflicts` 表中增加 `local_change_data` 和 `remote_change_data` 字段（类型 `TEXT`，存储完整 Change JSON），并将外键改为可选（或删除级联约束）。

---

### 4. 归一化查询使用了错误的 JSON Path
**位置：** §5.3 `NormalizeWeights`

**问题：** 代码使用 `data->>'$.guid'` 查询最近一次的 reindex 记录：
```go
db.Where("type = 'reindex' AND data->>'$.guid' = ?", parentGUID)...
```
但 PRD §4.4 定义的 `reindex` 变更格式中，`guid` 是 **Change 对象的顶层字段**，而非 `data` 内部字段：
```json
{ "type": "reindex", "guid": "folder-guid", "data": { "children_weights": {...} } }
```

**影响：** 该查询永远返回空，导致 5 分钟节流逻辑失效，可能频繁触发归一化。

**修复建议：** 改为 `WHERE type = 'reindex' AND guid = ?`。

---

### 5. PostgreSQL 时间戳字段存在溢出的高危隐患
**位置：** §3.1 Schema + §3.2 PostgreSQL 兼容说明

**问题：** SQLite schema 中所有时间戳使用 `INTEGER`，虽然 SQLite 的 INTEGER 是 64 位，但 §3.2 的兼容说明仅轻描淡写地提到 "INTEGER → BIGINT"。如果开发者在迁移到 PostgreSQL 时漏改任一字段，将直接导致 32 位溢出（PostgreSQL INTEGER 上限约 21 亿，而 Unix 毫秒在 1970 年 1 月 25 日就会超过该值）。

**影响：** 数据库无法写入任何时间戳，服务完全不可用。

**修复建议：**
- 在 SQLite schema 中直接使用 `BIGINT`（SQLite 也兼容 `BIGINT` 关键字），从源头上统一。
- 或在 §3.2 中升级为 **强制项** 而非建议，并列出所有必须修改的字段清单。

---

## 二、中等风险（建议修复）

### 6. 缺失 `sync_records` 表，导致历史记录 API 实现困难
**位置：** §3.1 Schema

**问题：** PRD §7.2 定义了 `SyncRecord` 数据模型，且 §6.5 的 `GET /api/v1/bookmarks/history` 需要返回 `device_id` 和 `changes_count`。技术方案完全没有设计 `sync_records` 表。

**影响：** 实现历史记录接口时，要么临时补表导致 schema 变更，要么被迫在 `change_log` 上做低效的聚合查询（按 from_version 范围分组不精确）。

**修复建议：** 增加 `sync_records` 表：
```sql
CREATE TABLE sync_records (
    version         INTEGER PRIMARY KEY,
    device_id       TEXT NOT NULL,
    change_ids      TEXT NOT NULL,  -- JSON 数组
    snapshot_hash   TEXT,
    synced_at       INTEGER NOT NULL
);
```

---

### 7. `bookmarks` 表包含 PRD 未定义的字段
**位置：** §3.1 Schema

**问题：** `bookmarks` 表中存在 `created_by` 和 `modified_by` 字段，但 PRD §7.1 的 Bookmark 模型中没有这两个字段。

**影响：** 属于与 PRD 的隐性偏离，可能导致数据冗余、Web UI 展示不一致，或在后续版本迭代中产生歧义。

**修复建议：** 若团队确实需要审计字段，应在 PRD 中补充说明；否则应从技术方案中删除。

---

### 8. Dockerfile 缺失前端构建步骤
**位置：** §6.1 Dockerfile

**问题：** Dockerfile 仅构建了 Go 后端二进制文件，没有构建 React Web 管理界面（`web/` 目录）。

**影响：** 部署后 Web 管理界面无法访问或返回 404。

**修复建议：** 增加多阶段构建，例如：
```dockerfile
FROM node:20-alpine AS web-builder
WORKDIR /build/web
COPY web/package*.json ./
RUN npm ci
COPY web/ ./
RUN npm run build

FROM golang:1.21-alpine AS go-builder
# ... 构建 server

FROM alpine:3.19
# ...
COPY --from=go-builder /build/server .
COPY --from=web-builder /build/web/dist ./web/dist
# 确保 LPK/Gin 配置为从 ./web/dist 提供静态文件
```

---

### 9. 环境变量中出现了 PRD 未提及的 `JWT_SECRET`
**位置：** §6.3 `.env.example`

**问题：** PRD §6.1 和 §9.3 明确的鉴权方式是 `Authorization: Bearer <API_Key>`（API Key 直接作为 Bearer Token），没有使用 JWT。配置文件中引入 `JWT_SECRET` 会让开发者困惑，甚至错误地实现 JWT 鉴权。

**修复建议：** 删除 `JWT_SECRET`，或明确注释说明 "仅当 LPK 框架内部需要时才配置"。

---

### 10. 离线队列合并逻辑对 `create` 操作存在 bug
**位置：** §5.4 `Queue.add`

**问题：** 合并逻辑通过 `c.guid === change.guid` 查找同一书签的未提交变更。当 `type = "create"` 时，`guid` 为 `null`，两条不同的 create 记录会错误地互相覆盖。

**影响：** 用户在离线状态下创建两个新书签，可能只保留最后一个。

**修复建议：** 当 `guid` 为 null 时，使用 `local_id` 作为合并键：
```typescript
const key = c.guid ?? c.local_id;
const changeKey = change.guid ?? change.local_id;
if (key === changeKey && !c.submitted) { ... }
```

---

### 11. `host_permissions` 过于宽泛
**位置：** §4.3 `manifest.json`

**问题：** `host_permissions` 设置为 `["https://*/*", "http://*/*"]`，权限过大。虽然便于用户配置任意服务器地址，但 Chrome Web Store 审核可能质疑，且存在安全风险。

**修复建议：**
- 使用 `optional_host_permissions`（MV3 支持），在插件配置页面通过 `chrome.permissions.request()` 动态申请用户输入的具体域名。
- 或在文档中明确说明此为 MVP 简化方案，后续迭代改为动态权限。

---

### 12. 监控指标使用了高基数 `device_id` label
**位置：** §8.1 `metrics.go`

**问题：** Prometheus Counter/Histogram 使用了 `device_id` 作为 label：
```go
[]string{"device_id", "status"}
```

**影响：** `device_id` 是 UUID/高基数维度，大量设备会导致 Prometheus 时间序列爆炸（TSDB cardinality issue），可能导致监控服务端 OOM。

**修复建议：** 移除 `device_id` label，改用固定的低基数维度（如 `status` 即可）。如需按设备分析，应使用日志或 APM Trace，而非 Metrics。

---

## 三、建议优化

### 13. 技术方案缺少快照生成与变更日志截断的具体实现
**位置：** §5 关键算法

**问题：** PRD §4.2 详细描述了变更日志截断策略（计数 > 1000 或年龄 > 7 天）和快照保留策略。技术方案仅在环境变量中提及配置项，没有在算法章节给出截断和快照生成的伪代码或流程。

**建议：** 补充 `SnapshotManager` 和 `ChangeLogTruncator` 的核心逻辑，包括：
- 截断时如何计算新的 `baseline_version`
- 何时触发快照生成（建议在 `POST /sync` 成功应用后异步生成）
- `reindex` 和 `rollback` 不计入 1000 条上限的具体 SQL 过滤条件

---

### 14. 未说明 LPK 框架与 Gin 的集成细节
**位置：** §2.1 / §2.2

**问题：** 技术方案选用 Gin 作为 Web 框架，理由是 "LPK 默认支持"。但没有说明 LPK 是否已有自己的路由/中间件机制，以及 Gin 在 LPK 中的集成方式（如是否通过 LPK 的 HTTP 服务注册、静态文件挂载方式等）。

**建议：** 补充 1-2 段说明 LPK 如何暴露 HTTP 端口、如何挂载 React 构建产物、以及 LPK 提供的默认中间件（如 CORS、Recovery）是否需要重复配置。

---

### 15. `InsertWithWeight` 前插权重可能无限递减
**位置：** §5.3 `InsertWithWeight`

**问题：** 反复在列表头部插入时，权重每次减 1.0，可能趋向负无穷。虽然 float64 范围极大，但如果存在恶意/异常场景（如脚本批量导入），可能触发异常值。

**建议：** 这是一个边缘场景，但建议文档中说明 "如权重 < -1e9 则触发强制归一化"。

---

### 16. 回滚全局锁在 PostgreSQL/多实例场景下失效
**位置：** §5.2 `VersionManager`

**问题：** `sync.Mutex` 仅在单进程内有效。如果未来从 SQLite 切换到 PostgreSQL 并横向扩展，回滚锁将失效。

**建议：** 在文档中添加风险提示：
> "当前全局锁基于进程内 `sync.Mutex`，适用于 SQLite 单实例部署。若切换至 PostgreSQL 或多实例部署，需替换为数据库级咨询锁（PostgreSQL advisory lock）或分布式锁。"

---

### 17. 开发里程碑中 Web UI 与后端任务耦合过紧
**位置：** §9 Phase 1

**问题：** Phase 1 第 5 天要求 "Web 管理界面框架" 与后端核心在同一天完成，但实际需要前端环境搭建、LPK 静态资源挂载等额外工作。

**建议：** 将 Web UI 框架搭建（如 React 路由、LPK 集成、API 客户端封装）单独抽出半天到 1 天，避免低估前端基建成本。

---

### 18. 测试策略中缺少对 `is_baseline=true` 全量重拉的覆盖
**位置：** §7 测试策略

**问题：** PRD §4.2 和 §8.1 将 `is_baseline=true` 作为核心溢出处理机制，但测试章节没有明确列出 "变更日志截断后触发全量同步" 的集成测试用例。

**建议：** 在 §7.3 集成测试清单中增加：
- 模拟变更日志截断 → 验证 `GET /bookmarks` 返回 `is_baseline=true`
- 验证客户端清空映射表并重建书签树

---

## 四、符合 PRD 的良好实践（值得肯定）

| 项 | 说明 |
|---|---|
| **先推后拉架构** | 与 PRD §3.4 数据流向完全一致。 |
| **float64 权重代替 index** | 有效避免了多设备排序冲突，方案描述清晰。 |
| **Chrome 三根映射** | 固定 GUID（root_bar/root_other/root_mobile）与 PRD §4.5 一致。 |
| **部分冲突处理** | §5.1 和 §7 测试用例均体现了 "无冲突变更正常应用" 的设计。 |
| **幂等保护** | `client_request_id` 和离线队列的 `submitted` 标记设计合理。 |
| **存储配额监控** | §4.4 的 5MB 配额检查与 80% 预警与 PRD §5.2.3 一致。 |
| **错误码表** | 技术方案未单独列出 API 错误码，但测试用例中引用了 E003/E007 等，与 PRD 对齐。 |

---

## 五、修复优先级清单

| 优先级 | 问题编号 | 修复内容 |
|---|---|---|
| P0 | 1 | 冲突检测使用 `baseVersion` 而非 `change.FromVersion` |
| P0 | 2 | `NextVersion` 使用 `UPDATE ... RETURNING` 或事务锁 |
| P0 | 3 | `conflicts` 表增加 `local_change_data` / `remote_change_data` |
| P0 | 4 | 归一化查询改为 `guid = ?` |
| P0 | 5 | SQLite schema 时间戳统一为 `BIGINT` |
| P1 | 6 | 补充 `sync_records` 表 |
| P1 | 8 | Dockerfile 增加前端构建阶段 |
| P1 | 9 | 删除 `.env.example` 中的 `JWT_SECRET` |
| P1 | 10 | 修复离线队列 `create` 合并键逻辑 |
| P1 | 12 | Prometheus 指标移除 `device_id` label |
| P2 | 7, 11, 13-18 | 按建议优化项逐项补充说明或文档 |

---

**总结：** `TECHNICAL-DESIGN-v1.md` 作为技术方案整体骨架清晰，技术选型合理，但在**核心算法（冲突检测、版本号管理、归一化查询）**和**数据模型（conflicts 表、sync_records 表、时间戳类型）**上存在多项与 PRD 不一致或存在运行时 bug 的设计。建议在启动 Phase 1 开发前，由方案作者针对上述 P0/P1 项进行一轮修正，并重新评审。
