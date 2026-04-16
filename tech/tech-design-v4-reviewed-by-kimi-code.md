# 技术方案审计报告

**审计对象：** `tech/TECHNICAL-DESIGN-v4.md`  
**对照基线：** `PRD-v6.md`  
**审计人：** Kimi Code  
**审计时间：** 2026-04-17  
**结论：** v4 修复了 v3 的绝大部分问题，但仍有 **6 项严重缺陷** 与 PRD v6 直接矛盾或存在运行时错误，**8 项中等风险** 需在编码前修正。

---

## 一、严重缺陷（必须修复）

### 1. `create_conflict` 检测完全缺失（PRD 要求未实现）
**位置：** §6.1 `ConflictDetector.Detect()`

**问题：** PRD §4.3 和 §6.1 明确要求检测 **"同一 `parent_guid` + 同名 + 同 URL"** 的创建冲突。但 v4 在 `Detect()` 中删除了所有 `create_conflict` 检测逻辑，仅在 `classifyConflict()` 中保留了针对**同一 GUID 的 create vs create** 的判断。而 PRD 定义的 `create_conflict` 发生在 `guid == null` 时（即客户端 create 尚未分配 GUID），需要查询 `bookmarks` 表或 `change_log` 中是否存在同 `parent_guid` + 同名 + 同 URL 的记录。

**影响：** 多设备同时离线创建同名同 URL 的书签时，服务端会生成两条独立的 GUID，导致后续出现重复书签，无法按 PRD "仅同文件夹内合并" 策略处理。

**修复建议：** 在 `Detect()` 中，对 `type == "create" && guid == ""` 的变更，主动查询服务端已存在书签和近期 `change_log`，若发现 `parent_guid + title + url` 完全匹配且未删除，则生成 `create_conflict`。

---

### 2. `classifyConflict` 中 create 逻辑完全颠倒
**位置：** §6.1 `classifyConflict()`

**问题：** 当前代码逻辑：
```go
case local.Type == "create" && remote.Type == "create":
    if local.Data.Title != remote.Data.Title || local.Data.URL != remote.Data.URL {
        return "create_conflict"
    }
    return "update_conflict"
```

这意味着：当两本书签**标题或 URL 不同**时，返回 `create_conflict`；当**完全一致**时，返回 `update_conflict`。

这与 PRD 语义完全相反：PRD §4.3 中 `create_conflict` 是指**同一文件夹内同名同 URL 的重复创建**（应合并），而标题/URL 不同意味着它们根本就是不同的书签，不应判定为冲突。

**影响：** 该逻辑几乎不可能在正常运行中触发（因为相同 GUID 的 create 本身就是异常），但一旦触发就会给出完全错误的冲突类型，导致 UI 提示和解决策略混乱。

**修复建议：** 将逻辑颠倒过来：当 title 和 URL 都相同时返回 `create_conflict`（表示重复创建），否则不应出现在 `classifyConflict` 中（应在 `Detect()` 阶段处理）。

---

### 3. 截断算法将 `reindex`/`rollback` 计入 1000 条上限
**位置：** §6.4 `TruncateChangeLog()`

**问题：** PRD §4.2 "变更日志截断策略" 明确规定：
> "计数规则：仅统计 type 为 create/update/delete/move 的变更。reindex 和 rollback 不计入 1000 条上限，但仍受 7 天年龄限制。"

但 v4 代码中：
```go
tx.Where("type IN ?", []string{"create", "update", "delete", "move", "reindex", "rollback"})
```
将**全部 6 种类型**都计入了 `maxCount` 的统计，导致当存在大量 `reindex` 时，会过早触发截断。

**影响：** 例如场景 D（PRD §4.2）：800 条普通变更 + 300 条 reindex，最旧 5 天 → PRD 要求**不截断**，但 v4 会错误截断。

**修复建议：** 按数量截断的 `WHERE` 条件应仅包含 `{"create", "update", "delete", "move"}`；按年龄截断的 `WHERE` 条件才应包含全部类型（或不限制类型）。

---

### 4. 错误码定义与 PRD v6 §6.1 完全不一致
**位置：** §4.4 错误码表

**问题：** v4 彻底重定义了 `E001-E006` 的含义和 HTTP 状态码，与 PRD 的 API 契约严重背离：

| 错误码 | PRD v6 定义 | v4 定义 | 是否一致 |
|---|---|---|---|
| E001 | 401 API Key 无效 | 400 base_version 无效 | ❌ |
| E002 | 403 权限不足 | 400 变更数超过 500 | ❌ |
| E003 | 409 版本冲突 | 401 API Key 无效 | ❌ |
| E004 | 400 设备未注册 | 400 变更日志已截断 | ❌ |
| E005 | 400 数据格式错误 | 429 限流 | ❌ |
| E006 | 429 请求频率超限 | 400 参数格式错误 | ❌ |

**影响：** 如果前端、插件或其他集成方已按 PRD 错误码表实现错误处理逻辑，v4 的实现会导致所有错误处理失效。同时 PRD §6.1 中的 `E007`（500 服务端错误）和 `E008`（410 维护模式）在 v4 中完全缺失。

**修复建议：** 严格恢复 PRD §6.1 的错误码映射表，不得擅自重定义。

---

### 5. `POST /sync` 返回 `full_sync: true` 与 PRD 矛盾
**位置：** §4.3 响应不变式、§4.5 全量同步触发条件

**问题：** PRD §6.1 和 §6.3.2 明确规定：当 `base_version` 严重落后或早于 `baseline_version` 时，`POST /sync` 应返回 **409 E003**（版本冲突），客户端收到后触发全量同步。PRD §4.2 中 "返回 200 + 完整规范树（`is_baseline=true`）" 的规定仅适用于 **GET /bookmarks**（增量拉取端点），而非 `POST /sync`。

v4 将溢出处理改为 `POST /sync` 返回 HTTP 200 + `full_sync: true`，这与 PRD 的"先推后拉"协议不一致：
- 推送端点返回 200 会让客户端误以为变更已被接受，可能错误更新 `last_known_version`；
- PRD 明确使用 409 强制客户端中断当前推送流程、先执行全量拉取。

**影响：** 客户端状态机与服务端响应不匹配，可能导致增量同步窗口丢失或数据不一致。

**修复建议：**
- `POST /sync` 在 `base_version < baseline_version` 时返回 **409 E003**（与 PRD §6.1 一致）；
- 删除 `SyncResponse` 中的 `full_sync` 字段；
- 全量树 + `is_baseline=true` 仅保留在 `GET /api/v1/bookmarks` 响应中。

---

### 6. `ChangeData` 结构体无法表示 `reindex` 变更的数据
**位置：** §6.1 `ChangeData`、§6.3 `NormalizeWeights`

**问题：** `ChangeData` 结构体定义为：
```go
type ChangeData struct {
    Title       string  `json:"title,omitempty"`
    URL         string  `json:"url,omitempty"`
    ParentGUID  string  `json:"parent_guid,omitempty"`
    IndexWeight float64 `json:"index_weight,omitempty"`
    DateCreated int64   `json:"date_created,omitempty"`
    DateModified int64  `json:"date_modified,omitempty"`
}
```
但 `reindex` 变更需要存储的数据格式是：
```json
{ "children_weights": { "guid-1": 1.0, "guid-2": 2.0 } }
```

`ChangeData` 中没有 `ChildrenWeights` 字段，导致 `NormalizeWeights` 中的伪代码只能写 `change.Data = ChangeData{}` 并注释 "实际实现中填充"，实际上无法编译或存储。

**影响：** `reindex` 变更的 `data` 字段无法被 GORM 正确序列化，其他设备拉取后无法解析权重更新。

**修复建议：** 在 `ChangeData` 中增加字段：
```go
ChildrenWeights map[string]float64 `json:"children_weights,omitempty"`
```
或使用 `json.RawMessage` / `map[string]interface{}` 作为底层存储，再通过类型断言转换。

---

## 二、中等风险（建议修复）

### 7. Prometheus `VersionGaps` 告警误用 `rate` 函数
**位置：** §9.2 `VersionGaps` 告警规则

**问题：**
```yaml
expr: rate(lzc_version_gaps_total[1h]) > 10
```
`rate` 返回的是**每秒变化率**。1 小时内出现 10 个空洞，其 `rate` 约为 `10/3600 ≈ 0.0028`，而不是 `10`。当前规则意味着每秒 10 个空洞（1 小时 3.6 万个），与注释 "过去 1 小时空洞超过 10 个" 相差 3600 倍。

**修复建议：** 改为 `increase(lzc_version_gaps_total[1h]) > 10`。

---

### 8. `SyncResponse` 缺失 PRD 要求的字段
**位置：** §4.2.2 `SyncResponse`

**问题：** 与 PRD §6.3.2 相比，`SyncResponse` 缺少以下字段：
- `synced_at`：PRD 要求所有响应顶层包含服务端时间戳；
- `generated_mappings`：当请求包含 `create` 操作时，必须返回 `local_id ↔ guid` 映射表；
- `Conflict` 结构缺少 `resolution_hint`：PRD 部分冲突响应示例中包含该字段（`"resolution_hint": "use_local"`）。

**修复建议：** 补充接口定义：
```typescript
interface SyncResponse {
  success: boolean;
  data: {
    new_version: number;
    applied_changes: string[];
    conflicts?: Conflict[];
    generated_mappings?: { local_id: string; guid: string }[];
  };
  version: number;
  synced_at: string;  // ISO 8601
}

interface Conflict {
  conflict_id: string;
  guid: string;
  type: string;
  local_change: Change;
  remote_change: Change;
  resolution_hint?: string;  // 如 "use_local"
}
```

---

### 9. `TruncateChangeLog` baseline 计算仍不精确
**位置：** §6.4 `TruncateChangeLog`

**问题：** v4 计算 `cutoffVersion = oldestChange.FromVersion - 1`（第一个被删除记录的 from_version - 1）。PRD §4.2 要求：
> "更新基准快照：is_baseline=true 的 snapshot.version 更新为截断点的最小 from_version"

即 `baseline_version` 应等于**保留记录中最小的 `from_version`**，而不是 `oldestChange.FromVersion - 1`。由于 `to_version = from_version + 1`，第一个保留记录的 `from_version` 通常等于 `oldestChange.ToVersion`（≥ `oldestChange.FromVersion + 1`）。因此 v4 的 baseline 比 PRD 要求偏小 1~N，会导致更多设备被不必要地降级为全量同步。

**修复建议：** 截断后查询剩余记录中最小的 `from_version`，直接设为 `baseline_version`：
```go
var baselineVersion int64
err = tx.Model(&Change{}).Where("to_version > ?", cutoffVersion).Select("MIN(from_version)").Scan(&baselineVersion).Error
```

---

### 10. `AcquireLock` 使用全局 `zap.L()` 存在耦合风险
**位置：** §6.2 `AcquireLock`

**问题：** `zap.L()` 是 Zap 的全局 logger。如果项目采用依赖注入 logger（而非全局替换），此处可能写入 no-op 或默认配置，导致锁等待日志不可见；若全局 logger 未初始化，某些 Zap 版本可能 panic。

**修复建议：** 在 `VersionManager` 结构体中注入 `*zap.Logger`：
```go
type VersionManager struct {
    db *gorm.DB
    mu sync.Mutex
    log *zap.Logger
}
```

---

### 11. `GenerateSnapshot` 清理条件可能误删 `is_baseline` 快照
**位置：** §6.5 `GenerateSnapshot`

**问题：** 当前清理 SQL 未排除 `is_baseline=true` 的快照。PRD §4.2 要求 `is_baseline=true` 的快照是变更日志截断后的全量同步基准点，一旦被删除，落后设备将无法重建书签树。

**修复建议：** 在 `DELETE` 条件中增加 `AND is_baseline = 0`：
```sql
DELETE FROM snapshots 
WHERE is_baseline = 0 
  AND version NOT IN (...) 
  AND created_at < ?
```

---

### 12. `NextVersion` 的 `Raw + Scan` 在 GORM 中存在兼容性隐患
**位置：** §6.2 `NextVersion`

**问题：** `db.Raw("UPDATE ... RETURNING ...").Scan(&state.CurrentVersion)` 对 pgx 和 modernc.org/sqlite 3.35+ 理论上有效，但 GORM 的 `Raw()` 在某些版本/驱动组合下对 `UPDATE ... RETURNING` 支持并不完善（例如可能不执行写操作、或 Scan 不到结果）。

**修复建议：** 这不是代码缺陷，但应在文档中标注为**高风险实现点**，并补充单元测试覆盖：
> "`NextVersion` 依赖 `UPDATE ... RETURNING`，必须在 CI 中对 SQLite 和 PostgreSQL 分别测试，验证其原子性和正确性。"

---

### 13. `Change` 结构体 `ParentLocalID` 存在 JSON 序列化泄漏风险
**位置：** §6.1 `Change` 结构体

**问题：** `ParentLocalID string gorm:"-"` 仅阻止 GORM 持久化，但 Gin JSON 序列化默认会包含该字段。如果服务端在返回 `change_log` 条目时带出 `parent_local_id`，会暴露客户端内部临时 ID。

**修复建议：** 增加 `json:"-"` 标签：
```go
ParentLocalID string `gorm:"-" json:"-"`
```

---

### 14. 最小验收条目未覆盖 PRD 的 `is_baseline` 全量同步
**位置：** §8.4 最小验收条目

**问题：** v4 的验收条目写的是 "返回 `full_sync: true`"，但按 PRD 修正后，应验证 "变更日志截断后客户端正确触发全量同步（`GET /bookmarks` 返回 `is_baseline=true`）"。

**修复建议：** 更新验收条目以匹配 PRD 协议。

---

## 三、值得肯定的改进（v3 → v4）

| 改进项 | 评价 |
|---|---|
| **删除 FOR UPDATE，统一使用 RETURNING** | 正确解决了 SQLite 不支持 FOR UPDATE 的问题。 |
| **删除同批 create 误杀逻辑** | 同一 `parent_local_id` 下多个 create 不再被错误判定为冲突。 |
| **响应不变式表格** | 四种场景下字段填充规则清晰，极大减少了实现歧义。 |
| **Dockerfile 仅复制必要目录 + USER 指令** | 构建优化和安全提升到位。 |
| **健康检查增加 DB 连接验证** | 比单纯的 HTTP 200 更有实际意义。 |
| **连接池配置与 busy_timeout** | SQLite 并发稳定性得到重视。 |
| **集成测试统一 pnpm + `--frozen-lockfile`** | 工具链一致性修复彻底。 |

---

## 四、修复优先级清单

| 优先级 | 问题编号 | 修复内容 |
|---|---|---|
| P0 | 1 | 在 `Detect()` 中补充 `create_conflict` 检测（同 `parent_guid` + `title` + `url`） |
| P0 | 2 | 修正 `classifyConflict` 中 create 逻辑（相同返回 create_conflict） |
| P0 | 3 | `TruncateChangeLog` 按数量截断时仅统计 create/update/delete/move |
| P0 | 4 | 恢复 PRD §6.1 的 `E001-E008` 错误码映射表 |
| P0 | 5 | `POST /sync` 溢出时返回 409 E003，删除 `full_sync` 字段 |
| P0 | 6 | `ChangeData` 增加 `ChildrenWeights` 字段以支持 reindex |
| P1 | 7 | `VersionGaps` 告警改为 `increase(...) > 10` |
| P1 | 8 | `SyncResponse` 补充 `synced_at`、`generated_mappings`、`resolution_hint` |
| P1 | 9 | `TruncateChangeLog` baseline 取保留记录的 `MIN(from_version)` |
| P1 | 10 | `VersionManager` 注入 logger 替代全局 `zap.L()` |
| P1 | 11 | `GenerateSnapshot` 删除条件增加 `AND is_baseline = 0` |
| P1 | 12 | 为 `NextVersion` 的 `RETURNING` 添加跨数据库单元测试覆盖说明 |
| P1 | 13 | `Change.ParentLocalID` 增加 `json:"-"` 标签 |
| P1 | 14 | 最小验收条目对齐 PRD 的 `is_baseline` 验证 |

---

**总结：** `TECHNICAL-DESIGN-v4.md` 在前几轮的基础上做了大量有价值的工程改进，文档清晰度和可执行性显著提升。但当前版本存在**与 PRD v6 的核心协议背离**（错误码重定义、`full_sync` 机制、`create_conflict` 缺失）以及**数据结构缺陷**（`ChangeData` 无法存储 reindex 数据、截断算法类型统计错误）。这些问题必须在进入 Phase 1 开发前修正，否则将导致服务端与客户端协议不兼容、增量同步逻辑失效。建议在修正 P0 项后重新进行快速评审。
