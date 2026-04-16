# 技术方案审计报告

**审计对象：** `tech/TECHNICAL-DESIGN-v3.md`  
**对照基线：** `PRD-v6.md`  
**审计人：** Kimi Code  
**审计时间：** 2026-04-16  
**结论：** v3 修复了前两版的大量问题，但仍有 **6 项严重缺陷** 与 PRD v6 直接矛盾或存在运行时错误，**6 项中等风险** 需在编码前修正。

---

## 一、严重缺陷（必须修复）

### 1. 410 GONE 处理与 PRD v6 直接矛盾
**位置：** §4.3 错误码表、§4.4 410 GONE 处理

**问题：** PRD §4.2 "变更日志溢出策略（v6 修正）" 明确写明：
> "服务端直接返回 HTTP 200 + 完整规范树（is_baseline=true）"
> "不再返回 410 GONE（v6 变更，简化流程）"
> "注意：410 GONE 仅用于服务端主动拒绝服务（如维护模式），正常溢出场景用 200 全量响应。"

但技术方案 v3 在 §4.3 中将 `GONE | 410` 定义为"变更日志已截断"，并在 §4.4 中要求客户端处理 410 后拉取全量快照。

**影响：** 客户端和服务端实现将与 PRD 规定的协议不一致，可能导致两端对同一溢出场景产生不同的降级行为（一端期待 200，另一端返回 410）。

**修复建议：**
- 删除 §4.3 中的 `GONE | 410 | 变更日志已截断` 错误码；
- 将 §4.4 改为：当 `last_known_version < baseline_version` 时，服务端返回 HTTP 200 + 完整规范树 + `is_baseline=true`；
- 410 仅保留用于维护模式等主动拒绝场景。

---

### 2. 同批 create 冲突检测逻辑错误（误杀正常批量创建）
**位置：** §6.1 `Detect()` 函数中的 `createByParent` 检测

**问题：** 代码将"同一 `parent_local_id` 下有多个 create"标记为 `create_conflict`：
```go
for parentID, creates := range createByParent {
    if len(creates) > 1 {
        // 同 parent_local_id 有多个 create，标记为冲突
```
但 PRD §6.3.2 "同批 create 的 GUID 分配协议" 明确支持并预期这种情况：
> "场景：客户端一次性上传 500 条 create 变更，其中包含父子关系"
> "协议：客户端按拓扑顺序排序变更（父节点在前，子节点在后）"
> "子节点的 parent_guid 字段填写服务端规范树中的 GUID（如果父节点已存在）"
> "如果父节点也是本次上传的新节点：使用 parent_local_id 字段指向父节点的 local_id"

同 `parent_local_id` 下创建多个子节点/书签是**完全正常且高频**的操作（例如在某文件夹里同时创建 3 个新书签）。

**影响：** 正常批量 create 会被错误判定为冲突，导致所有创建操作挂起，用户无法同步新书签。

**修复建议：** 删除 `createByParent` 这段同批 create 冲突检测逻辑。真正的 `create_conflict` 是指：
- **服务端已存在** 同 `parent_guid` + 同名 + 同 URL 且未删除的书签（PRD §4.3）；
- 或**不同设备**已提交的同条件 create（可通过查询 `change_log` 或 `bookmarks` 表检测）。

---

### 3. `NormalizeWeights` 插入 `change_log` 时未分配版本号
**位置：** §6.3 `NormalizeWeights`

**问题：** 事务内插入了 `change_log`：
```go
change = &Change{
    // ...
    FromVersion: 0,
    ToVersion:   0,  // 版本号为 0！
    // ...
}
err = tx.Create(change).Error
```
但 `change_log.to_version` 有 `UNIQUE` 约束且是全局版本号体系的核心。如果两次归一化操作的 `ToVersion` 都是 0，将触发唯一约束冲突；即使有唯一 `change_id` 主键避免重复行，版本号 0 也会破坏客户端的增量拉取逻辑（客户端可能已经有 `last_known_version > 0`，因此永远拉不到这条 `reindex` 变更）。

**影响：**
- 第二次归一化时可能因 `to_version = 0` 与第一次冲突（取决于数据库约束优先级）；
- 客户端版本号体系与 `change_log` 脱节，权重变更无法被其他设备感知。

**修复建议：** 在事务内调用 `NextVersion()`（或传入版本号分配器）获取当前版本号 +1，正确填充 `FromVersion` 和 `ToVersion`。

---

### 4. 截断算法 `baseline_version` 更新逻辑错误
**位置：** §6.5 `TruncateChangeLog`

**问题：**
1. `baseline_version` 被设置为 `cutoffVersion`（即被删除记录的最大 `to_version`），但 PRD §4.2 要求：
   > "更新基准快照：is_baseline=true 的 snapshot.version 更新为截断点的最小 from_version"
   > "新设备或落后设备（last_known_version < baseline_version）走全量同步"
   
   正确的 `baseline_version` 应该是**截断后保留的记录中最小的 `from_version`**，而不是被删除的 `to_version`。如果设为 `cutoffVersion`，那么 `last_known_version == cutoffVersion - 1` 的设备虽然只差一个版本，却已经拉不到变更，会被错误地判定为需要全量同步。

2. 代码中没有更新 `snapshots` 表的 `is_baseline` 标记。PRD 明确要求将某个 snapshot 标记为 `is_baseline=true`。

**影响：** 截断后大量本可以增量同步的设备被错误地降级为全量同步，增加服务端负载和客户端重建开销。

**修复建议：**
- 截断后查询剩余记录中最小的 `from_version`，设为 `baseline_version`；
- 找到 `version <= baseline_version` 的最大 snapshot，将其 `is_baseline` 设为 1（其余设为 0）；
- 如果没有现成 snapshot，应在截断前或截断后异步生成一个。

---

### 5. `SyncResponse` 结构错误地包含 `changes` 字段
**位置：** §4.2.2 `SyncResponse`

**问题：** 接口定义为：
```typescript
interface SyncResponse {
  // ...
  changes?: Change[];  // 增量变更（如果有）
}
```
但 PRD §3.4、§6.3.2 和 §8.1 均明确采用**"先推后拉"**模式：
> "4. POST /api/v1/bookmarks/sync（推送本地变更）"
> "5. GET /api/v1/bookmarks?since_version=last_known_version（拉取远程变更）"

推送响应中不应该包含远程增量变更，增量变更是通过单独的 GET 拉取。这是 PRD 的核心架构决策（"避免拉取后到提交前产生新的竞争窗口"）。

**影响：** 如果实现该接口，会破坏"先推后拉"的顺序保证，可能导致客户端在应用远程变更后又推送本地变更，产生复杂的时序问题。

**修复建议：** 从 `SyncResponse` 中删除 `changes` 字段。

---

### 6. `Change` 接口中 `parent_guid` 位置与 PRD 不一致
**位置：** §4.2.1 `Change` 接口

**问题：** 技术方案将 `parent_guid` 放在 `Change` 顶层：
```typescript
interface Change {
  // ...
  parent_guid: string | null;
  parent_local_id: string | null;
  data: {
    title?: string;
    url?: string;
    index_weight?: number;
  };
}
```
但 PRD §6.3.2 的请求体示例中，`parent_guid` 在 `data` 对象内部：
```json
{
  "data": { "title": "New Bookmark", "url": "https://...", "parent_guid": "root_bar" }
}
```
且 PRD §4.1 `PendingChange` 中只有 `parent_local_id` 在顶层。

**影响：** 服务端解析逻辑与 PRD 不一致，客户端如果按照 PRD 发送 `data.parent_guid`，服务端可能读取不到。

**修复建议：** 将 `parent_guid` 移入 `data` 对象内部，保持与 PRD 一致：
```typescript
data: {
  title?: string;
  url?: string;
  parent_guid?: string;
  index_weight?: number;
};
```

---

## 二、中等风险（建议修复）

### 7. `Detect()` 在 `guids` 为空时可能生成非法 SQL
**位置：** §6.1 `Detect()`

**问题：** 当 `changes` 数组全为 `create` 操作（`guid` 为空）时，`guids` 切片为空。此时 GORM 生成的 `WHERE guid IN ?` 可能变成 `WHERE guid IN ()`，这在 SQLite/PostgreSQL 中都是语法错误。

**修复建议：** 在查询前判断 `if len(guids) == 0 { return result, nil }`。

---

### 8. `GenerateSnapshot` 旧快照清理逻辑不完整且存在 SQL 风险
**位置：** §6.6 `GenerateSnapshot`

**问题：**
1. `NOT IN` 子查询在内部返回 NULL 时的行为在不同数据库中可能有歧义；
2. PRD §4.2 和 §5.1.2 要求快照保留策略为"最近 500 个版本 **或** 最近 30 天"，但代码只按数量清理，没有按年龄清理；
3. 子查询 `SELECT version FROM snapshots ORDER BY version DESC LIMIT 500` 在 SQLite 中支持，但在 PostgreSQL 的 `NOT IN` 子查询中如果没有别名可能报错。

**修复建议：**
- 改为 `DELETE FROM snapshots WHERE version < (SELECT version FROM snapshots ORDER BY version DESC LIMIT 1 OFFSET 500)` 的方式，或分两步查询再删除；
- 同时补充 30 天年龄清理：`AND created_at < now - 30days`。

---

### 9. `PendingChange.local_id` 被错误标记为 nullable
**位置：** §6.4 `PendingChange` 接口

**问题：** `local_id: string | null`，但 PRD §4.1 中 `PendingChange.local_id` 的类型是 `string`（非 null）。即使是 `create` 操作，也需要 `local_id` 来在离线队列中标识该书签。

**修复建议：** 改为 `local_id: string`。

---

### 10. 集成测试脚本仍使用旧版 `docker-compose` 命令
**位置：** §8.3 `tests/integration/run.sh`

**问题：** 脚本中使用 `docker-compose up -d` 和 `docker-compose down`。新版 Docker（v2+）已统一为 `docker compose`（无短横线）作为默认命令，旧版 `docker-compose` Python 插件在部分 CI 环境中可能未安装。

**修复建议：** 统一改为 `docker compose up -d` 和 `docker compose down`。

---

### 11. 集成测试构建插件时未使用 `--frozen-lockfile`
**位置：** §8.3 `run.sh`

**问题：** `cd extension && pnpm install && pnpm run build` 没有加 `--frozen-lockfile`，CI 环境中可能因依赖版本漂移导致构建不一致。

**修复建议：** 改为 `pnpm install --frozen-lockfile`。

---

### 12. `ChangeData` 结构缺少 `date_created` / `date_modified`
**位置：** §6.1 `ChangeData`

**问题：** 虽然 PRD 中 `data` 字段主要关注 `title`/`url`/`parent_guid`/`index_weight`，但 §4.2.1 的 `Change` 接口顶层包含了 `date_created` 和 `date_modified`。这部分数据在首次同步全量上传时是否需要携带？服务端是否只负责生成时间戳？文档未明确说明。

**修复建议：** 在 §4.2.1 或 §6.1 中添加注释说明：
> "`date_created`/`date_modified` 在 create/update 时由服务端统一生成并写入 `bookmarks` 表，客户端在 `data` 中不需要携带。"

---

## 三、值得肯定的改进

| 改进项 | 评价 |
|---|---|
| **新增同步协议章节** | 将 API 定义集中整理，提升了文档可读性。 |
| **UPDATE ... RETURNING / FOR UPDATE** | 版本号竞态问题已彻底解决。 |
| **pnpm workspace 共享代码** | 比 v2 的 `resolve.alias` 更符合工程实践。 |
| **截断算法和快照生成算法** | 补充了 v1/v2 缺失的核心伪代码，算法骨架清晰。 |
| **PromQL 修正** | 使用 `sum(rate(...))` 正确聚合，避免了除以零。 |
| **conflicts 表、sync_records 表、audit_log 表** | 数据模型已对齐 PRD 并补充了审计能力。 |
| **host_permissions 默认为空** | 配合动态权限申请，安全性和用户体验兼顾。 |

---

## 四、修复优先级清单

| 优先级 | 问题编号 | 修复内容 |
|---|---|---|
| P0 | 1 | 删除 410 GONE 作为正常溢出场景，统一为 200 + is_baseline=true |
| P0 | 2 | 删除误杀正常批量 create 的 `createByParent` 冲突检测逻辑 |
| P0 | 3 | `NormalizeWeights` 在事务内调用 `NextVersion()` 分配正确版本号 |
| P0 | 4 | 修正 `TruncateChangeLog` 的 `baseline_version` 计算逻辑，并更新 snapshot 的 `is_baseline` |
| P0 | 5 | 从 `SyncResponse` 中删除 `changes` 字段 |
| P0 | 6 | 将 `Change` 接口中的 `parent_guid` 移入 `data` 内部 |
| P1 | 7 | `Detect()` 在 `guids` 为空时提前返回 |
| P1 | 8 | `GenerateSnapshot` 改用更安全的清理 SQL，并补充 30 天年龄清理 |
| P1 | 9 | `PendingChange.local_id` 改为非 null |
| P1 | 10 | 集成测试脚本改用 `docker compose` |
| P1 | 11 | 集成测试构建使用 `pnpm install --frozen-lockfile` |
| P1 | 12 | 明确 `date_created`/`date_modified` 的生成责任方 |

---

**总结：** `TECHNICAL-DESIGN-v3.md` 已经是一份**相当成熟的技术方案**，前两轮的核心算法缺陷、并发问题和工程细节大都得到了妥善解决。当前最紧要的问题是**与 PRD v6 的协议一致性**（尤其是 410 GONE 的移除和"先推后拉"的响应格式），以及**同批 create 冲突检测的误杀**。建议在启动 Phase 1 前用半天时间集中修正上述 P0 项，即可放心进入开发。
