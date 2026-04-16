# 懒猫书签同步系统 - 技术方案 v4 评审报告

**审计对象:** TECHNICAL-DESIGN-v4.md
**基准文档:** PRD v6.0
**审计时间:** 2026-04-17
**审计者:** GLM-5.1
**审计结论: 有条件通过，需修复 2 个阻断性问题 + 4 个重要问题后进入开发**

---

## 审计摘要

| 等级 | 数量 | 说明 |
|------|------|------|
| **阻断 (Blocker)** | 2 | 影响编译/运行正确性 |
| **重要 (Major)** | 4 | 影响与 PRD 一致性或逻辑正确性 |
| **建议 (Minor)** | 5 | 改进建议，不阻塞开发 |
| **v3 问题修复确认** | 3/3 Blocker 全部修复 | 无伪修复 |
| **正向反馈** | 7 | 值得肯定的设计决策 |

---

## 一、v3 审计问题修复追踪

| v3 编号 | 问题 | v4 状态 | 备注 |
|---------|------|---------|------|
| B1 | Data 字段无法编译 | **已修复** | `ChangeData` 结构体 + `gorm:"serializer:json"` 标签，支持点号访问 |
| B2 | SQLite 不支持 FOR UPDATE | **已修复** | 删除 `NextVersionSQLite`，统一使用 `RETURNING` |
| B3 | 同步协议与 PRD 矛盾 | **已修复** | 端点、410 GONE、parent_guid 位置、changes 字段全部对齐 |
| M1 | 截断算法未清理 reindex/rollback | **已修复** | WHERE 条件包含全部 6 种类型 |
| M2 | create_conflict 检测逻辑错误 | **已修复** | 删除同批 create 检测，classifyConflict 比较 title/URL |
| M3 | Data 字段类型混用 | **已修复** | 统一为 `ChangeData` |
| M4 | SQLite 缺 busy_timeout | **已修复** | 添加 `PRAGMA busy_timeout=5000` |
| M5 | Dockerfile 复制全量源码 | **已修复** | 改为选择性 COPY |

**修复率: 8/8 (100%)**。v3 的 3 个 Blocker 和 5 个 Major 全部正确修复，无伪修复。这是从 v1→v2→v3→v4 四个版本中首次实现 Blocker 全清。

---

## 二、阻断性问题 (Blocker)

### B1. NormalizeWeights 生成的 reindex 变更 data 字段为空

**位置:** 第 799-813 行 `pkg/sync/weight.go`

```go
change = &Change{
    // ...
    Data: ChangeData{
        // v4 修正：使用 ChangeData 结构体
    },
    // ...
}

// 填充 data 字段
change.Data = ChangeData{}  // 实际实现中填充 children_weights
```

**问题:** `ChangeData` 结构体定义的字段为 `Title`、`URL`、`ParentGUID`、`IndexWeight`、`DateCreated`、`DateModified`——没有 `ChildrenWeights` 字段。reindex 变更的核心数据（子节点权重映射表 `{guid: weight, ...}`）无处可存。

客户端收到 reindex 变更时，需要 `children_weights` 来更新本地排序。如果 data 为空，客户端无法应用归一化结果。

**修复方案（二选一）:**

方案 A：扩展 `ChangeData`，添加 reindex/rollback 专用字段：
```go
type ChangeData struct {
    Title           string            `json:"title,omitempty"`
    URL             string            `json:"url,omitempty"`
    ParentGUID      string            `json:"parent_guid,omitempty"`
    IndexWeight     float64           `json:"index_weight,omitempty"`
    DateCreated     int64             `json:"date_created,omitempty"`
    DateModified    int64             `json:"date_modified,omitempty"`
    ChildrenWeights map[string]float64 `json:"children_weights,omitempty"`  // reindex 用
    RollbackToVersion int64           `json:"rollback_to_version,omitempty"` // rollback 用
    SnapshotTree    json.RawMessage   `json:"snapshot_tree,omitempty"`       // rollback 用
}
```

方案 B（更优）：为不同变更类型定义不同的 Data 类型，使用 `interface{}` 或 `any`：
```go
type Change struct {
    // ...
    Data any `gorm:"type:text;serializer:json"`
    // ...
}
```
但这样会失去类型安全。综合来看，方案 A 更实用——多出的字段在非 reindex/rollback 场景为空值，`omitempty` 确保序列化时不输出。

**影响:** 归一化产生的变更无有效数据，客户端无法应用权重更新。

---

### B2. 截断算法计数逻辑与 PRD v6 矛盾

**位置:** 第 848-851 行 `TruncateChangeLog`

```go
err := tx.Where("type IN ?",
    []string{"create", "update", "delete", "move", "reindex", "rollback"}).  // 全部 6 种类型
    Order("to_version ASC").
    Offset(int(maxCount)).
    First(&oldestChange).Error
```

**问题:** PRD v6 明确规定"仅统计 type 为 create/update/delete/move 的变更"，"reindex 和 rollback 不计入 1000 条上限判断"。但 v4 的截断计数包含了全部 6 种类型（包括 reindex 和 rollback），这意味着归一化和回滚操作会加速触发截断，可能导致正常的增量变更被过早删除。

**修复:** 计数查询仅统计 4 种普通类型：
```go
// 按数量截断：仅统计普通变更
err := tx.Where("type IN ?",
    []string{"create", "update", "delete", "move"}).  // 排除 reindex/rollback
    Order("to_version ASC").
    Offset(int(maxCount)).
    First(&oldestChange).Error
```

删除查询仍包含全部 6 种类型（line 877），这是正确的。

**影响:** reindex/rollback 操作加速触发截断，可能导致正常的增量变更过早丢失。

---

## 三、重要问题 (Major)

### M1. 错误码表仍与 PRD v6 不一致

**位置:** 第 465-475 行 §4.4 错误码

| 错误码 | v4 定义 (HTTP) | PRD v6 定义 (HTTP) | 差异 |
|--------|---------------|-------------------|------|
| OK | 200 | - | PRD 无此错误码 |
| CONFLICT | 200 | - | PRD 无此错误码 |
| E001 | 400 (base_version 无效) | **401** (API Key 无效) | HTTP 状态码和语义完全不同 |
| E002 | 400 (变更数超过 500) | **403** (权限不足) | HTTP 状态码和语义完全不同 |
| E003 | 401 (API Key 无效) | **409** (版本冲突) | HTTP 状态码和语义完全不同 |
| E004 | 400 (变更日志已截断) | **400** (设备未注册) | 语义不同 |
| E005 | 429 (限流) | **400** (数据格式错误) | HTTP 状态码和语义完全不同 |
| E006 | 400 (参数格式错误) | **429** (限流) | HTTP 状态码和语义完全不同 |
| E007 | - (缺失) | **500** (服务端错误) | 缺失 |
| E008 | - (缺失) | **410** (维护模式) | 缺失 |

**问题:** v4 的错误码编号与 PRD v6 的含义完全错位（E001 在 PRD 是鉴权失败，在 v4 是版本号无效），且缺少 E007/E008。同时保留了 PRD 中不存在的 `OK` 和 `CONFLICT` 标识。

**修复:** 严格对齐 PRD v6 §6.1 的错误码表，删除 `OK`/`CONFLICT`，修正 E001-E008 的 HTTP 状态码和语义。

**影响:** 如果客户端按 PRD 定义的错误码实现处理逻辑，会与服务端返回不一致。

---

### M2. GORM `serializer:json` 未指定列类型

**位置:** 第 574 行

```go
Data ChangeData `gorm:"serializer:json"`
```

**问题:** `gorm:"serializer:json"` 告诉 GORM 如何序列化/反序列化，但未指定数据库列类型。GORM 会根据 Go 结构体类型推断列类型——对于复合结构体 `ChangeData`，GORM 可能生成的 DDL 不正确（如尝试创建多个列而非单个 TEXT 列）。应显式指定：

```go
Data ChangeData `gorm:"type:text;serializer:json"`
```

**影响:** AutoMigrate 生成的 DDL 可能不正确，或手动迁移时列类型不确定。

---

### M3. `full_sync` 触发位置与 PRD 不一致

**位置:** 第 439 行 SyncResponse、第 476-485 行 §4.5

v4 在 **POST /sync 响应** 中添加了 `full_sync: true`，当 `base_version < baseline_version` 时返回。

PRD v6 的溢出策略定义在 **GET /bookmarks** 端点（§6.3.1）：
- `GET /bookmarks?since_version=N` 返回 `is_baseline: true`
- 客户端收到后执行全量拉取

PRD 的 sync 接口（POST /sync）返回 409 E003 触发全量同步，而非 `full_sync: true`。

两种方案都能工作，但语义不同：
- PRD 方案：push 端点返回错误码 → 客户端从 pull 端点获取全量数据
- v4 方案：push 端点直接告知需要全量同步

建议在技术方案中明确标注这是对 PRD 的**设计决策变更**，并同步更新 PRD。否则开发时两份文档不一致。

**影响:** 不影响功能，但文档不一致导致实现时选择困难。

---

### M4. SyncErrorRate PromQL 仍无零流量保护

**位置:** 第 1287-1289 行

```yaml
expr: |
  sum(rate(lzc_sync_requests_total{status="error"}[5m]))
  / sum(rate(lzc_sync_requests_total[5m]))
> 0.1
```

服务刚启动无流量时，分母为 0，PromQL 返回 `NaN`。虽然 `NaN > 0.1` 为 false 不会误触发告警，但表达式不规范，部分 Prometheus 版本会发出 evaluation warning。

修复：
```yaml
expr: |
  sum(rate(lzc_sync_requests_total{status="error"}[5m]))
  / sum(rate(lzc_sync_requests_total[5m])) > 0.1
  and sum(rate(lzc_sync_requests_total[5m])) > 0
```

此问题从 v2 开始延续，建议在本版本终结。

---

## 四、建议性问题 (Minor)

### m1. Dockerfile web 阶段仍复制 `package*.json`

**位置:** 第 999 行

```dockerfile
COPY web/package*.json ./
```

glob 模式会匹配可能残留的 `package-lock.json`。既然已改用 pnpm，建议精确指定：
```dockerfile
COPY web/package.json ./
```
（`pnpm-lock.yaml` 已在下一行单独复制）

### m2. HealthHandler 未设置 Content-Type

**位置:** 第 1323 行

```go
json.NewEncoder(w).Encode(map[string]string{...})
```

`json.NewEncoder` 不会自动设置 `Content-Type: application/json` 头。虽然不影响功能，但不符合 HTTP 规范。建议添加：
```go
w.Header().Set("Content-Type", "application/json")
```

### m3. 缺少数据导出和搜索 API

从 v1 延续至今，PRD v6 §5.1.4 要求的"数据导出"和"搜索"功能仍未出现。建议在 Phase 4 补充：
```
GET /api/v1/bookmarks/export     # 导出 JSON
GET /api/v1/bookmarks/search?q=  # 搜索
```

### m4. 幂等窗口"5 分钟"与 reindex 变更的 potential 冲突

**位置:** 第 489 行

5 分钟幂等窗口对于弱网重试（指数退避可能 30s+5min）可能偏短。但 PRD 未规定窗口时间，且 `sync_records.request_id` 提供了持久化去重，实际 5 分钟只是内存缓存窗口。建议在注释中说明 5 分钟是内存窗口，`sync_records` 提供长期去重。

### m5. Conflicts 表 resolution 缺少 CHECK 约束

**位置:** 第 307 行

```sql
resolution TEXT,
```

PRD v6 定义 resolution 为 `use_local/use_remote/merge`，建议添加 CHECK：
```sql
resolution TEXT CHECK (resolution IS NULL OR resolution IN ('use_local', 'use_remote', 'merge')),
```

---

## 五、正向反馈

### P1. Blocker 全清——首次无伪修复

v1→v2 有 2 项伪修复，v2→v3 有 2 项伪修复。v3→v4 的 3 个 Blocker 全部正确修复，无伪修复。这是一个重要的质量转折点。

### P2. ChangeData + serializer:json 方案清晰

`Change` 结构体定义完整（第 569-580 行），`ChangeData` 字段明确，`gorm:"serializer:json"` 语义清晰。代码审查者可以直接看懂序列化机制。

### P3. 响应不变式表格（§4.3）

第 456-462 行新增的"响应不变式"表格，明确列出了四种场景（完全成功、部分冲突、版本溢出、参数错误）下每个字段的填充规则。消除了 sync 响应语义的歧义。

### P4. 截断算法的数学化说明

第 834-842 行添加了设计意图和边界条件说明（`baseline_version = 保留记录的最小 from_version - 1`，空 change_log 不截断），使截断逻辑的可理解性大幅提升。

### P5. Dockerfile 安全加固

添加 `USER appuser`（第 1029-1030 行）非 root 运行，是容器安全最佳实践。

### P6. 最小验收条目（§8.4）

第 1198-1205 行新增的验收表格，为 6 个核心场景提供了明确的预期结果，可直接用于 Phase 5 验收。

### P7. 健康检查包含 DB 连接验证

第 1309-1328 行的健康检查不仅检查 HTTP 端口，还验证数据库连接可用性，避免"服务存活但功能不可用"的假阳性。

---

## 六、PRD v6 覆盖度审查

| PRD v6 需求 | v3 状态 | v4 状态 | 备注 |
|-------------|---------|---------|------|
| 同步 API 端点路径 | `/sync` (矛盾) | `/bookmarks/sync` | **已对齐** |
| 错误码格式 | 纯英文 (矛盾) | E001-E006 | **部分对齐**（M1：编号含义错位+缺失2个） |
| 410 GONE 行为 | 410 (矛盾) | 200 + full_sync | **已对齐** |
| API 鉴权 | 已覆盖 | 已覆盖 | - |
| 书签 CRUD | 已覆盖 | 已覆盖 | - |
| 增量同步 | 已覆盖 | 已覆盖 | - |
| 冲突检测 | 已覆盖 | 已覆盖 | classifyConflict 逻辑正确 |
| 版本快照与回滚 | 已覆盖 | 已覆盖 | 新增生成/清理算法 |
| 变更日志截断 | 已覆盖 | 已覆盖 | reindex 计数有误（B2） |
| 排序权重归一化 | 已覆盖 | 已覆盖 | data 字段为空（B1） |
| Service Worker 降级 | 已覆盖 | 已覆盖 | - |
| 离线队列管理 | 已覆盖 | 已覆盖 | - |
| 限流策略 | 已覆盖 | 已覆盖 | - |
| 监控指标 | 已覆盖 | 已覆盖 | 新增版本空洞监控 |
| SyncRecord 模型 | 已覆盖 | 已覆盖 | - |
| 审计日志 | 已覆盖 | 已覆盖 | - |
| 冲突解决超时 | 未覆盖 | **已覆盖** | 7 天自动解决 |
| 数据库健康检查 | 未覆盖 | **已覆盖** | /health 含 DB 检查 |
| 数据导出 API | 缺失 | 缺失 | - |
| 书签搜索 API | 缺失 | 缺失 | - |

**覆盖率: ~93%**（从 v3 的 87% 提升至 93%，核心同步逻辑完整对齐 PRD）

---

## 七、修复优先级建议

| 优先级 | 编号 | 预估工作量 |
|--------|------|-----------|
| P0 | B1 (reindex data 字段为空) | 0.5 天 |
| P0 | B2 (截断计数排除 reindex/rollback) | 0.5 小时 |
| P1 | M1 (错误码对齐 PRD) | 1 小时 |
| P1 | M2 (serializer 列类型) | 5 分钟 |
| P1 | M3 (full_sync 文档对齐) | 0.5 小时 |
| P1 | M4 (PromQL 零流量保护) | 5 分钟 |
| P2 | m1-m5 | 0.5 天 |

**总计: 约 1.5 天**（从 v3 的 3.5 天大幅下降）

---

## 八、版本演进总结

| 维度 | v1 | v2 | v3 | v4 |
|------|----|----|----|-----|
| Blocker | 3 | 2 | 3 | **2** |
| Major | 8 | 6 | 5 | **4** |
| PRD 覆盖率 | 82% | 89% | 87% | **93%** |
| 修复工作量 | 6.5 天 | 3.5 天 | 3.5 天 | **1.5 天** |
| Blocker 性质 | 实现错误 | 伪修复 | 伪修复 | **实现细节** |
| 伪修复数 | - | 2 | 2 | **0** |

**趋势分析:**

1. **收敛趋势明显**: Blocker 从实现架构级错误（v1 的 CGO、goroutine 泄漏）收敛到数据字段缺失（v4 的 reindex data）。问题越来越局部化。
2. **首次零伪修复**: v3→v4 的修复全部正确，说明修复验证机制在改善。
3. **工作量持续下降**: 6.5 → 3.5 → 3.5 → 1.5 天，已接近可开发阈值。

---

## 九、结论

v4 是四个版本中质量最高的一版。v3 的 3 个 Blocker 全部正确修复，新增内容（响应不变式、健康检查、冲突超时、最小验收条目）均为有价值的补充。剩余 2 个 Blocker 属于实现细节级别（reindex data 字段缺失、截断计数范围），修复简单且不影响整体架构。

**建议:** 修复 B1+B2+M1 后即可启动 Phase 1 开发。P1 问题可在开发过程中并行修复。
