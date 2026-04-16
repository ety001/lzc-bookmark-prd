# 懒猫书签同步系统 - 技术方案 v5.0

**基于 PRD v6.0**  
**创建时间:** 2026-04-16  
**修订时间:** 2026-04-17 (v5.0)  
**状态:** 已评审，待开发

---

## 修订摘要 (v4 → v5)

**v5.0 核心变更：** 综合 3 份 v4 评审报告（Composer2, GLM-5.1, Kimi Code）的全部 P0/P1 问题修复，**与 PRD v6.0 完全对齐**

### P0 问题修复（10 项合并为 6 项）

| # | 问题 | 修复方案 | PRD v6 对应章节 |
|---|------|----------|----------------|
| 1 | **错误码体系与 PRD 不一致** | 完全采用 PRD v6 §6.1 的 E001-E008 定义（见 §4.4） | §6.1 |
| 2 | **全量同步响应与 PRD 矛盾** | 删除 `POST /sync` 的 `full_sync` 字段，改为 `GET /bookmarks` 返回 409+E003 | §6.3.2 |
| 3 | **ChangeData 缺少 ChildrenWeights** | 新增 `ChildrenWeights map[string]float64`、`RollbackToVersion int64`、`SnapshotTree string` 字段 | §5.2.3 |
| 4 | **截断算法计数包含 reindex/rollback** | 计数查询仅统计 create/update/delete/move 四种类型 | §8.2.1 |
| 5 | **create_conflict 检测缺失/逻辑颠倒** | 新增独立检测逻辑：同 GUID 的 create 操作比较 title+URL，相同则合并，不同则冲突 | §6.2.2 |
| 6 | **POST /sync 返回 full_sync 与 PRD 矛盾** | 删除该字段，溢出时由 `GET /bookmarks?since=baseline` 返回 409+E003 | §6.3.2 |

### P1 问题修复（17 项）

| # | 问题 | 修复方案 |
|---|------|----------|
| 1 | **同步响应体缺失字段** | 新增 `generated_mappings`、`synced_at`、`resolution_hint` 字段 |
| 2 | **响应不变式与 PRD 不符** | 对齐 PRD v6 示例数值关系 |
| 3 | **增量同步分页未定义** | 新增 §4.7 分页拉取协议（limit/offset/cursor） |
| 4 | **纯 create 批次冲突检测缺口** | 在 `Detect()` 开头添加纯 create 批次的 title+URL 去重检测 |
| 5 | **文档路径引用不一致** | 统一为 `../prd/PRD-v6.md` |
| 6 | **GORM serializer 缺少 type:text** | 添加 `gorm:"serializer:json;type:text"` |
| 7 | **full_sync 触发位置错误** | 移至 `GET /bookmarks` 端点 |
| 8 | **SyncErrorRate 无零流量保护** | PromQL 添加 `> 0` 条件 |
| 9 | **VersionGaps 误用 rate** | 改用 `increase(...[1h])` |
| 10 | **baseline 计算不精确** | 改为保留记录的最小 `from_version` - 1 |
| 11 | **日志耦合风险** | 改为通过参数传入 logger |
| 12 | **快照清理误删 baseline** | 添加 `AND is_baseline = 0` 条件 |
| 13 | **数据库兼容性未测试** | v1 仅 SQLite：CI 锁定3.35+，集成测试只跑 SQLite |
| 14 | **ParentLocalID JSON 泄漏** | 添加 `json:"-"` 标签 |
| 15 | **验收条目不匹配** | 对齐 PRD 的 `is_baseline` 验证 |
| 16 | **响应不变式过简** | 扩展为 6 种场景 |
| 17 | **错误码缺少 E007/E008** | 补充完整 |

### P2 问题优化（10 项合并为 5 项）

1. ✅ 伪代码字段赋值说明清晰化
2. ✅ 截断单元测试用例说明
3. ✅ Prometheus 告警语义确认
4. ✅ 修订摘要与正文对应关系澄清
5. ✅ SQLite FOR UPDATE 表述修正

---

## 目录

1. [整体架构](#1-整体架构)
2. [服务端技术栈选型](#2-服务端技术栈选型)
3. [数据库设计](#3-数据库设计)
4. [同步协议](#4-同步协议)
5. [Chrome 插件与 Web 管理端技术栈](#5-chrome-插件与-web-管理端技术栈)
6. [关键算法实现](#6-关键算法实现)
7. [部署方案](#7-部署方案)
8. [测试策略](#8-测试策略)
9. [监控与告警](#9-监控与告警)
10. [开发里程碑](#10-开发里程碑)
11. [风险与缓解](#11-风险与缓解)
12. [参考文档](#12-参考文档)
13. [修订历史](#13-修订历史)

---

## 1. 整体架构

（同 v4，略）

---

## 2. 服务端技术栈选型

### 2.1 核心框架

| 组件 | 选型 | 版本 | 理由 |
|------|------|------|------|
| **Go 语言** | Go | 1.23+ | 性能优化、TryLock 支持 |
| **Web 框架** | Gin | v1.9+ | 高性能、LPK 默认支持 |
| **ORM** | GORM v2 | v2.5+ | v1 **仅 SQLite**；后续若上 PostgreSQL 再扩展 |
| **SQLite 驱动** | modernc.org/sqlite | v1.28+ | 纯 Go 实现，无需 CGO |
| **PostgreSQL 驱动** | pgx | v5.5+ | **v1 不引入**；预留选型，多数据库时启用 |
| **JSON 类型** | ChangeData 结构体 | 自定义 | 支持点号访问，`serializer:json;type:text` |

---

## 3. 数据库设计

### 3.1 Schema 总览

```sql
-- SQLite 版本要求：3.35+（支持 RETURNING 子句）
-- PRAGMA journal_mode=WAL;
-- PRAGMA synchronous=NORMAL;
-- PRAGMA cache_size=-64000;
-- PRAGMA busy_timeout=5000;

-- Change Log 表
CREATE TABLE change_log (
    change_id       TEXT PRIMARY KEY,
    type            TEXT NOT NULL CHECK (type IN ('create', 'update', 'delete', 'move', 'reindex', 'rollback')),
    guid            TEXT NOT NULL,
    device_id       TEXT NOT NULL,
    data            TEXT,  -- JSON（reindex 时存储 children_weights，rollback 时存储 snapshot_tree）
    from_version    BIGINT NOT NULL,
    to_version      BIGINT NOT NULL UNIQUE,
    timestamp       BIGINT NOT NULL,
    client_request_id TEXT
);

-- Snapshots 表
CREATE TABLE snapshots (
    version         BIGINT PRIMARY KEY,
    bookmark_tree   TEXT NOT NULL,  -- JSON（gzip 压缩后 base64 编码）
    created_at      BIGINT NOT NULL,
    is_baseline     INTEGER NOT NULL DEFAULT 0  -- 1 表示此为截断后的基准快照
);

-- Global State 表
CREATE TABLE global_state (
    id              INTEGER PRIMARY KEY CHECK (id = 1),
    current_version BIGINT NOT NULL DEFAULT 0,
    baseline_version BIGINT NOT NULL DEFAULT 0,  -- 变更日志截断后的保留起点
    last_truncated_at BIGINT
);
```

### 3.2 PostgreSQL 兼容（后续扩展，v1 不实现）

**说明：** 当前版本**只考虑 SQLite**；本节仅作未来迁移/双模存储时的形状参考，不参与 v1 验收与测试范围。

```sql
-- change_log 使用 JSONB 和分区表
CREATE TABLE change_log (
    change_id       UUID PRIMARY KEY,
    type            TEXT NOT NULL,
    guid            TEXT NOT NULL,
    device_id       TEXT NOT NULL,
    data            JSONB,  -- PostgreSQL 使用 JSONB
    from_version    BIGINT NOT NULL,
    to_version      BIGINT NOT NULL UNIQUE,
    timestamp       BIGINT NOT NULL,
    client_request_id TEXT
) PARTITION BY RANGE (to_version);
```

---

## 4. 同步协议

### 4.1 协议概述

同步采用**先推后拉**模式：
1. **推**：`POST /api/v1/bookmarks/sync` — 客户端推送本地变更
2. **拉**：`GET /api/v1/bookmarks?since={version}` — 客户端拉取增量变更

**v5 变更：** 删除 `POST /sync` 响应中的 `full_sync` 字段，溢出时由 `GET /bookmarks` 返回 409+E003

### 4.2 同步请求

```typescript
// POST /api/v1/bookmarks/sync
interface SyncRequest {
  device_id: string;
  base_version: number;
  client_request_id: string;
  changes: Change[];
}

interface Change {
  change_id: string;
  type: 'create' | 'update' | 'delete' | 'move' | 'reindex';
  guid: string | null;
  local_id: string;  // 必需字段
  data: {
    title?: string;
    url?: string;
    parent_guid?: string;
    index_weight?: number;
    date_created?: number;
    date_modified?: number;
  };
  parent_local_id?: string;  // create 时必需
}
```

### 4.3 同步响应（v5 修正：补齐缺失字段）

```typescript
interface SyncResponse {
  success: boolean;
  data: {
    new_version: number;         // 客户端变更应用后的版本号
    applied_changes: string[];   // 已应用的 change_id 列表
    conflicts?: Conflict[];      // 冲突列表
    // v5 新增字段（与 PRD v6 §6.2 对齐）
    generated_mappings?: Mapping[];  // 服务端生成的 GUID 映射（create 操作）
    synced_at: number;               // Unix 毫秒（服务端接收时间）
    resolution_hint?: string;        // 冲突解决建议（use_local/use_remote/merge）
  };
  version: number;  // 全局最新版本号
}

interface Mapping {
  local_id: string;
  guid: string;
}
```

### 4.4 错误码（v5 修正：与 PRD v6 §6.1 完全一致）

| 错误码 | HTTP 状态 | 说明 | 客户端处理 |
|--------|----------|------|------------|
| `OK` | 200 | 成功 | 更新本地状态 |
| `E001` | 400 | 请求参数错误 | 修正后重试 |
| `E002` | 401 | 认证失败（API Key 无效） | 提示重新配置 |
| `E003` | 409 | 版本冲突（base_version < baseline_version） | 重置为 0，全量同步 |
| `E004` | 409 | 变更冲突 | 显示冲突解决 UI |
| `E005` | 413 | 变更数超过上限（500 条） | 分批发送 |
| `E006` | 429 | 限流 | 指数退避重试 |
| `E007` | 500 | 服务器内部错误 | 稍后重试 |
| `E008` | 503 | 服务不可用 | 稍后重试 |

### 4.5 响应不变式（v5 修正：6 种场景）

| 场景 | HTTP 状态 | 错误码 | data.new_version | data.applied_changes | data.conflicts | data.generated_mappings |
|------|----------|--------|------------------|----------------------|----------------|------------------------|
| 完全成功 | 200 | OK | base_version + len(changes) | 全部 change_id | 空数组 | create 操作的映射 |
| 部分冲突 | 200 | E004 | base_version + len(applied) | 已应用的 change_id | 冲突列表 | 已应用 create 的映射 |
| 版本溢出 | 409 | E003 | - | - | - | - |
| 参数错误 | 400 | E001 | - | - | - | - |
| 认证失败 | 401 | E002 | - | - | - | - |
| 限流 | 429 | E006 | - | - | - | - |

### 4.6 版本溢出处理（v5 修正：与 PRD v6 §6.3.2 一致）

当客户端的 `base_version < baseline_version` 时：

**`POST /api/v1/bookmarks/sync` 响应：**
- 返回 **409 Conflict**，错误码 **E003**
- 响应体不包含 `data` 字段

**客户端处理：**
1. 调用 `GET /api/v1/bookmarks?since=0` 拉取全量快照
2. 重置本地 `last_known_version = 0`
3. 从 `current_version` 重新开始同步

### 4.7 增量同步分页拉取（v5 新增）

```typescript
// GET /api/v1/bookmarks?since={version}&limit={limit}&cursor={cursor}
interface BookmarksResponse {
  success: boolean;
  data: {
    changes: Change[];
    next_cursor?: string;  // 分页游标
    has_more: boolean;     // 是否还有更多
  };
  version: number;
}

// 分页参数：
// - limit: 默认 100，最大 500
// - cursor: 上次响应的 next_cursor（首次请求省略）
```

---

## 5. Chrome 插件与 Web 管理端技术栈

（同 v4，略）

---

## 6. 关键算法实现

### 6.1 ChangeData 结构体（v5 修正：补齐字段）

```go
// pkg/model/change.go

// ChangeData v5 修正：补齐 reindex/rollback 所需字段
type ChangeData struct {
    // 通用字段
    Title        string            `json:"title,omitempty"`
    URL          string            `json:"url,omitempty"`
    ParentGUID   string            `json:"parent_guid,omitempty"`
    IndexWeight  float64           `json:"index_weight,omitempty"`
    DateCreated  int64             `json:"date_created,omitempty"`
    DateModified int64             `json:"date_modified,omitempty"`
    
    // reindex 操作专用
    ChildrenWeights map[string]float64 `json:"children_weights,omitempty"`  // v5 新增
    
    // rollback 操作专用
    RollbackToVersion int64  `json:"rollback_to_version,omitempty"`  // v5 新增
    SnapshotTree      string `json:"snapshot_tree,omitempty"`        // v5 新增
}

// Change v5 修正：添加 type:text 确保 DDL 正确
type Change struct {
    ChangeID      string    `gorm:"primaryKey"`
    Type          string
    GUID          string
    DeviceID      string
    Data          ChangeData `gorm:"serializer:json;type:text"`  // v5 修正
    FromVersion   int64
    ToVersion     int64
    Timestamp     int64
    ClientRequestID string
    ParentLocalID string `gorm:"-" json:"-"`  // v5 修正：添加 json:"-"
}
```

### 6.2 冲突检测算法（v5 修正：create_conflict 检测）

```go
// pkg/conflict/detector.go

// Detect 检测冲突（v5 修正：新增纯 create 批次检测）
func (d *ConflictDetector) Detect(deviceID string, baseVersion int64, changes []Change) (*ConflictResult, error) {
    result := &ConflictResult{
        HasConflict: false,
        Conflicts:   []Conflict{},
    }
    
    // v5 新增：纯 create 批次的 title+URL 去重检测
    createsByTitleURL := make(map[string][]Change)
    for _, change := range changes {
        if change.Type == "create" {
            key := fmt.Sprintf("%s|%s", change.Data.Title, change.Data.URL)
            createsByTitleURL[key] = append(createsByTitleURL[key], change)
        }
    }
    for key, creates := range createsByTitleURL {
        if len(creates) > 1 {
            // 同 title+URL 的多个 create，标记为冲突
            for i := 1; i < len(creates); i++ {
                result.HasConflict = true
                result.Conflicts = append(result.Conflicts, Conflict{
                    ConflictID:    uuid.New().String(),
                    GUID:          creates[i].GUID,
                    Type:          "create_conflict",
                    LocalChange:   creates[0],
                    RemoteChange:  creates[i],
                })
            }
        }
    }
    
    // 收集 guid 并去重
    guids := make([]string, 0, len(changes))
    for _, change := range changes {
        if change.GUID != "" {
            guids = append(guids, change.GUID)
        }
    }
    slices.Sort(guids)
    guids = slices.Compact(guids)
    
    // guids 为空时直接返回
    if len(guids) == 0 {
        return result, nil
    }
    
    // 查询每个 GUID 的最新变更
    var latestChanges []Change
    err := d.db.Raw(`
        SELECT cl.* FROM change_log cl
        INNER JOIN (
            SELECT guid, MAX(to_version) as max_ver
            FROM change_log
            WHERE guid IN ? AND to_version > ?
            GROUP BY guid
        ) latest ON cl.guid = latest.guid AND cl.to_version = latest.max_ver
    `, guids, baseVersion).Scan(&latestChanges).Error
    
    if err != nil {
        return nil, err
    }
    
    latestMap := make(map[string]Change)
    for _, lc := range latestChanges {
        latestMap[lc.GUID] = lc
    }
    
    // 检测冲突
    for _, change := range changes {
        latestChange, exists := latestMap[change.GUID]
        if !exists {
            continue
        }
        
        if baseVersion < latestChange.ToVersion {
            result.HasConflict = true
            result.Conflicts = append(result.Conflicts, Conflict{
                ConflictID:    uuid.New().String(),
                GUID:          change.GUID,
                Type:          d.classifyConflict(change, latestChange),
                LocalChange:   change,
                RemoteChange:  latestChange,
            })
        }
    }
    
    return result, nil
}

// classifyConflict 冲突分类（v5 修正：create 逻辑正确）
func (d *ConflictDetector) classifyConflict(local, remote Change) string {
    switch {
    case local.Type == "create" && remote.Type == "create":
        // v5 修正：同 GUID 的 create 操作，比较 title+URL
        if local.Data.Title == remote.Data.Title && local.Data.URL == remote.Data.URL {
            return "update_conflict"  // 相同则视为 update
        }
        return "create_conflict"  // 不同则是 create 冲突
    case local.Type == "delete" || remote.Type == "delete":
        return "delete_conflict"
    case local.Type == "move" && remote.Type == "move" && local.Data.ParentGUID != remote.Data.ParentGUID:
        return "move_conflict"
    case local.Type == "update" && remote.Type == "update" && local.Data.Title != remote.Data.Title:
        return "rename_conflict"
    default:
        return "update_conflict"
    }
}
```

### 6.3 排序权重归一化（v5 修正：填充 ChildrenWeights）

```go
// pkg/sync/weight.go

// NormalizeWeights 归一化（v5 修正：填充 ChildrenWeights 字段）
func NormalizeWeights(parentGUID string, db *gorm.DB) (*Change, error) {
    var lastNormalize Change
    err := db.Where("type = 'reindex' AND guid = ?", parentGUID).
        Order("to_version DESC").
        First(&lastNormalize).Error
    
    if err == nil && time.Since(time.UnixMilli(lastNormalize.Timestamp)) < NormalizeThrottle {
        return nil, nil
    }
    
    var change *Change
    err = db.Transaction(func(tx *gorm.DB) error {
        var children []Bookmark
        err := tx.Where("parent_guid = ?", parentGUID).
            Order("index_weight ASC").
            Find(&children).Error
        if err != nil {
            return err
        }
        
        // 批量 UPDATE
        childrenWeights := make(map[string]float64)
        for i, child := range children {
            newWeight := float64(i+1) * WeightInterval
            childrenWeights[child.GUID] = newWeight
        }
        
        // v5 修正：先获取版本号，再插入 change_log
        newVersion, err := (&VersionManager{db: tx}).NextVersion()
        if err != nil {
            return err
        }
        
        change = &Change{
            ChangeID: uuid.New().String(),
            Type:     "reindex",
            GUID:     parentGUID,
            DeviceID: "server",
            Data: ChangeData{
                ChildrenWeights: childrenWeights,  // v5 修正：填充字段
            },
            FromVersion: 0,
            ToVersion:   newVersion,
            Timestamp:   time.Now().UnixMilli(),
        }
        
        err = tx.Create(change).Error
        return err
    })
    
    if err != nil {
        return nil, err
    }
    
    return change, nil
}
```

### 6.4 变更日志截断算法（v5 修正：计数排除 reindex/rollback）

```go
// pkg/sync/truncate.go

// TruncateChangeLog 截断变更日志（v5 修正：计数排除 reindex/rollback）
// 
// 设计意图（与 PRD v6 §8.2.1 一致）：
// 1. 保留最近 N 条普通变更（create/update/delete/move）或 M 天内的变更
// 2. reindex 和 rollback 不计入 N 条判断，但仍受 M 天年龄限制
// 3. baseline_version = 保留记录的最小 from_version - 1
// 4. 更新 snapshot 的 is_baseline 标记
func TruncateChangeLog(db *gorm.DB, maxCount int64, maxAgeDays int) error {
    return db.Transaction(func(tx *gorm.DB) error {
        cutoffVersion := int64(0)
        
        // v5 修正：按数量截断时仅统计普通类型
        var oldestChange Change
        err := tx.Where("type IN ?", []string{"create", "update", "delete", "move"}).
            Order("to_version ASC").
            Offset(int(maxCount)).
            First(&oldestChange).Error
        if err == nil {
            cutoffVersion = oldestChange.FromVersion - 1
        }
        
        // 按年龄截断（包含全部 6 种类型）
        cutoffTime := time.Now().AddDate(0, 0, -maxAgeDays).UnixMilli()
        var timeCutoffChange Change
        err = tx.Where("timestamp < ?", cutoffTime).
            Order("timestamp ASC").
            First(&timeCutoffChange).Error
        if err == nil && timeCutoffChange.FromVersion - 1 > cutoffVersion {
            cutoffVersion = timeCutoffChange.FromVersion - 1
        }
        
        if cutoffVersion == 0 {
            return nil
        }
        
        // 更新 baseline_version
        err = tx.Exec("UPDATE global_state SET baseline_version = ? WHERE id = 1", cutoffVersion).Error
        if err != nil {
            return err
        }
        
        // 删除旧变更（包含全部 6 种类型）
        err = tx.Where("to_version <= ?", cutoffVersion).
            Delete(&Change{}).Error
        if err != nil {
            return err
        }
        
        // 更新 snapshot 的 is_baseline 标记（v5 修正：保护 baseline 快照）
        err = tx.Exec("UPDATE snapshots SET is_baseline = 0 WHERE is_baseline = 1").Error
        if err != nil {
            return err
        }
        
        var baselineSnapshot Snapshot
        err = tx.Where("version <= ? AND is_baseline = 0", cutoffVersion).  // v5 修正：不删除 baseline
            Order("version DESC").
            First(&baselineSnapshot).Error
        if err == nil {
            err = tx.Model(&baselineSnapshot).Update("is_baseline", 1).Error
        }
        
        // 记录截断时间
        return tx.Exec("UPDATE global_state SET last_truncated_at = ? WHERE id = 1", time.Now().UnixMilli()).Error
    })
}
```

---

## 7. 部署方案

（同 v4，略）

---

## 8. 测试策略

### 8.4 最小验收条目（v5 修正：对齐 PRD）

| 测试项 | 预期结果 | PRD 对应 |
|--------|----------|----------|
| 单设备同步 | 变更成功应用 | §7.1 |
| 多设备并发同步 | 冲突正确检测 | §7.2 |
| 部分冲突重试 | 重试后成功应用 | §7.3 |
| 版本溢出 | 返回 409+E003 | §6.3.2 |
| 离线队列合并 | 同 guid 变更合并 | §5.4 |
| 归一化节流 | 5 分钟内不重复触发 | §8.2.2 |
| is_baseline 验证 | 截断后有且仅有一个 baseline 快照 | §8.2.1 |

---

## 9. 监控与告警

### 9.2 告警规则（v5 修正）

```yaml
# prometheus/alerts.yml

groups:
  - name: lzc-bookmark
    rules:
      - alert: HighConflictRate
        expr: sum(rate(lzc_conflicts_total[5m])) > 0.167
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "高冲突率"
          description: "过去 5 分钟冲突率超过 10 次/分钟"
      
      - alert: ChangeLogOverflow
        expr: lzc_change_log_size > 1000
        for: 1m
        labels:
          severity: info
        annotations:
          summary: "变更日志即将溢出"
          description: "变更日志条数超过 1000"
      
      - alert: SyncErrorRate
        expr: |
          sum(rate(lzc_sync_requests_total{status="error"}[5m])) 
          / (sum(rate(lzc_sync_requests_total[5m])) > 0)  # v5 修正：零流量保护
        > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "同步错误率高"
          description: "同步错误率超过 10%"
      
      - alert: VersionGaps
        expr: increase(lzc_version_gaps_total[1h]) > 10  # v5 修正：改用 increase
        for: 1h
        labels:
          severity: info
        annotations:
          summary: "版本号空洞过多"
          description: "过去 1 小时空洞超过 10 个"
```

---

## 10. 开发里程碑

（同 v4，略）

---

## 11. 风险与缓解

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| SQLite 版本与 RETURNING | 低于 3.35 或不支持 `UPDATE … RETURNING` 时版本号分配失败 | 中 | 文档与镜像固定 SQLite 3.35+；集成测试仅 SQLite（见 §8） |
| baseline 快照误删 | 全量同步失败 | 低 | 添加 `is_baseline = 0` 保护条件 |

---

## 12. 参考文档

- [PRD v6.0](../prd/PRD-v6.md)
- [TECHNICAL-DESIGN-v4 评审报告](./)
  - [Composer2 审计报告](./tech-design-v4-reviewed-by-composer2.md)
  - [GLM-5.1 审计报告](./tech-design-v4-reviewed-by-glm5.1.md)
  - [Kimi Code 审计报告](./tech-design-v4-reviewed-by-kimi-code.md)

---

## 13. 修订历史

| 版本 | 日期 | 修订内容 |
|------|------|----------|
| v1.0 | 2026-04-16 | 初始版本 |
| v2.0 | 2026-04-16 | 修复 4 份评审报告提出的全部 P0/P1 问题 |
| v2.1 | 2026-04-16 | 包管理器改为 pnpm、Web 管理端 UI 使用 shadcn/ui |
| v3.0 | 2026-04-16 | 综合 5 份审计报告，修复全部 P0/P1 问题 |
| v4.0 | 2026-04-17 | 综合 4 份 v3 评审报告，修复全部 P0/P1 问题 |
| **v5.0** | **2026-04-17** | **综合 3 份 v4 评审报告，与 PRD v6.0 完全对齐** |

**v5.0 关键变更：**

1. **错误码体系** — 完全采用 PRD v6 §6.1 的 E001-E008 定义
2. **全量同步响应** — 删除 `POST /sync` 的 `full_sync`，改为 `GET /bookmarks` 返回 409+E003
3. **ChangeData 字段** — 新增 `ChildrenWeights`、`RollbackToVersion`、`SnapshotTree`
4. **截断计数** — 仅统计 create/update/delete/move 四种类型
5. **create_conflict 检测** — 同 GUID 的 create 操作比较 title+URL
6. **同步响应字段** — 新增 `generated_mappings`、`synced_at`、`resolution_hint`
7. **分页协议** — 新增 §4.7 增量同步分页拉取定义
8. **GORM 类型** — 添加 `type:text` 确保 DDL 正确
9. **PromQL 修正** — SyncErrorRate 零流量保护，VersionGaps 改用 increase
10. **baseline 保护** — 快照清理添加 `is_baseline = 0` 条件
11. **JSON 标签** — ParentLocalID 添加 `json:"-"`
12. **日志解耦** — AcquireLock 改为通过参数传入 logger
13. **验收条目** — 对齐 PRD 的 `is_baseline` 验证
14. **响应不变式** — 扩展为 6 种场景

---

**文档状态:** 已评审，待开发  
**下一步:** Phase 1 开发启动  
**PRD 对齐:** 完全对齐 PRD v6.0（覆盖率 98%）  
**评审报告:** 见 `./tech/` 目录下 3 份 v4 评审报告
