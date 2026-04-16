# 技术方案审计报告

**审计对象：** `TECHNICAL-DESIGN-v2.md`（含 v2.1 修订）  
**对照基线：** `PRD-v6.md`  
**审计人：** Kimi Code  
**审计时间：** 2026-04-16  
**结论：** v2 已修复 v1 的绝大部分严重缺陷，整体质量大幅提升，可进入开发。但存在 **2 项严重缺陷**、**5 项中等风险** 和 **4 项建议优化**，建议在编码前修正。

---

## 一、严重缺陷（必须修复）

### 1. 冲突检测算法遗漏了 `create` 冲突
**位置：** §5.1 `ConflictDetector.Detect`

**问题：** PRD §4.3 明确要求检测 **"同一 `parent_guid` + 同名 + 同 URL"** 的创建冲突，且冲突条件第一条写明 "同一 GUID（**或创建冲突的同 parent_guid + 同名 + 同 URL**）"。但 `Detect` 函数仅通过 `guid IN ?` 查询已有变更，对于客户端 `guid = null` 的 `create` 操作完全未作检测。

**影响：** 多设备同时离线创建同名同 URL 的书签时，服务端会生成两条独立的 GUID，导致后续出现重复书签且无法按 PRD 策略 "仅同文件夹内合并"。

**修复建议：** 在 `Detect` 中增加对 `type = "create"` 且 `guid = null`（或空字符串）的变更的专门检测逻辑：
```go
for _, change := range changes {
    if change.Type == "create" && change.GUID == "" {
        // 查询同 parent_guid + 同名 + 同 URL 且未软删除的书签
        // 或查询 change_log 中同条件的近期 create 记录
        // 若存在则判定为 create_conflict
    }
}
```

---

### 2. `NormalizeWeights` 的事务不完整（`change_log` 未在同一事务中写入）
**位置：** §5.3 `NormalizeWeights`

**问题：** 事务内完成了 `bookmarks` 表的权重更新，但 `change_log` 的插入被注释掉：
```go
// 插入 change_log（版本号由调用者填充）
// err = tx.Create(change).Error
// return err
```
且 `change` 对象是在事务内部生成、但由函数返回后由调用者填充版本号再写入。

**影响：** 如果调用者在更新 `bookmarks` 后、写入 `change_log` 前崩溃或遇到网络异常，数据库中书签权重已改变但 `change_log` 无记录。其他设备增量拉取时将看不到 `reindex` 变更，导致排序权重与服务端不一致。

**修复建议：** 将版本号分配和 `change_log` 写入封装在同一事务中。可以：
- 在 `NormalizeWeights` 中传入 `VersionManager` 或 `nextVersion func() (int64, error)` 回调；
- 在事务内调用 `NextVersion()` 获取版本号，填充 `change`，并执行 `tx.Create(change)`。

---

## 二、中等风险（建议修复）

### 3. `PendingChange` 接口缺少 `base_version` 字段
**位置：** §5.4 `Queue.ts` 中的 `PendingChange` 接口

**问题：** PRD §6.3.2 和 §8.1 均要求客户端在部分冲突后 "更新冲突变更的 `base_version = new_version`"，且 §7.2 的测试用例中引用了 `pendingChanges[0].base_version`。但 `PendingChange` 接口定义中没有该字段。

**影响：** 插件侧无法正确实现部分冲突重试逻辑，导致冲突变更反复以旧 `base_version` 推送。

**修复建议：**
```typescript
interface PendingChange {
  // ... 现有字段
  base_version: number;  // 用于冲突检测的基准版本号
}
```

---

### 4. Dockerfile 前端构建未复制 pnpm lock 文件
**位置：** §6.1 Dockerfile（web-builder 阶段）

**问题：**
```dockerfile
COPY web/package*.json ./
RUN pnpm install --frozen-lockfile
```
`pnpm install --frozen-lockfile` 需要 `pnpm-lock.yaml` 文件才能正常工作。如果该文件未复制到构建上下文中，构建将失败；如果只复制了 `package*.json` 而没有 lock 文件，安装行为不可复现。

**影响：** CI/CD 构建不稳定或失败。

**修复建议：**
```dockerfile
COPY web/package.json web/pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
```

---

### 5. 集成测试脚本仍使用 `npm` 而非 `pnpm`
**位置：** §7.3 `tests/integration/run.sh`

**问题：** v2.1 已将包管理器统一为 pnpm，但集成测试脚本中仍写：
```bash
cd extension && npm run build
```

**影响：** 集成测试与开发工具链不一致，可能导致依赖安装方式不同、构建结果差异。

**修复建议：** 改为 `cd extension && pnpm run build`。

---

### 6. `host_permissions` 的默认示例域名在实际分发时存在体验问题
**位置：** §4.4 `manifest.json`

**问题：** `host_permissions` 固定为 `["https://api.example.com/*"]`，虽然 `optional_host_permissions` 提供了 `<all_urls>`，但插件首次安装后如果没有动态申请权限，将无法直接访问用户配置的私有服务器地址。

**影响：** 首次配置体验可能不顺畅，用户可能在未触发权限申请的情况下看到网络错误。

**修复建议：** 在文档中明确说明：
- 插件首次保存 API 配置时，必须调用 `chrome.permissions.request({origins: [userServerUrl + "/*"]})`；
- 若用户拒绝权限申请，应给出清晰的中文提示。

---

### 7. `classifyConflict` 对 "重命名冲突" 的分类逻辑存在边界模糊
**位置：** §5.1 `classifyConflict`

**问题：** 代码中仅当 `local.Type == "update" && remote.Type == "update" && local.Data.Title != remote.Data.Title` 时返回 `rename_conflict`。但如果 `local` 是 `update`（改 title），`remote` 是 `move`（改 parent_guid），按当前逻辑会返回 `update_conflict`，而非 PRD §4.3 表格中更具体的 `move_conflict`。

**影响：** 冲突类型分类可能不够精确，影响 UI 提示文案。

**修复建议：** 调整分类优先级：先检查 `delete_conflict`，再检查 `move_conflict`，再检查 `rename_conflict`，最后 fallback 到 `update_conflict`。

---

## 三、建议优化

### 8. 归一化操作存在 N+1 更新风险
**位置：** §5.3 `NormalizeWeights` 事务内

**问题：** 对每个子节点执行单独的 `tx.Model(&child).Update(...)`，当单文件夹下有数千个子节点时，会产生大量 UPDATE 语句。

**建议：** 虽然 PRD 规定单文件夹子节点上限 10,000 且归一化低频，但建议在技术方案中补充批量更新策略（如 `UPDATE bookmarks SET index_weight = CASE guid WHEN ? THEN ? ... END WHERE parent_guid = ?`），或至少作为后续优化项记录。

---

### 9. 仍缺少快照生成与变更日志截断的核心算法
**位置：** §5 关键算法

**问题：** PRD §4.2 详细定义了变更日志截断策略（计数 > 1000 或年龄 > 7 天）和快照 `is_baseline` 更新逻辑，但技术方案在 §5 算法章节中仍未给出 `SnapshotManager` 或截断器的伪代码。

**建议：** 补充一个 `ChangeLogTruncator` 的流程说明，包括：
- 截断触发条件的具体 SQL 计算方式；
- 截断后如何选取新的 `baseline_version`（即剩余记录中最小的 `from_version`）；
- 快照生成时机（建议在 `POST /sync` 成功应用后异步生成）。

---

### 10. 插件与 Web 管理端的共享代码目录结构不够明确
**位置：** §4.2 / §4.3

**问题：** 文档提到 "创建 `shared/` 目录存放插件与 Web 端共享的代码"，但 LPK 标准项目结构（§2.2）中 `web/` 和插件目录是平级关系，没有明确说明 `shared/` 的具体位置以及构建工具链如何引用（如 Vite alias、monorepo workspace、或符号链接）。

**建议：** 在 LPK 项目结构图中补充说明：
```
lzc-bookmark/
├── web/                 # React 管理端
├── extension/           # Chrome 插件
└── shared/              # 插件与 Web 共享的类型定义、常量、API 端点
```
并说明 Vite 中通过 `resolve.alias` 映射 `@shared` 到 `../shared`。

---

### 11. `AcquireLock` 的超时轮询存在微小延迟
**位置：** §5.2 `AcquireLock`

**问题：** `ticker := time.NewTicker(100 * time.Millisecond)` 配合 `select` 意味着每次锁获取失败后至少要等待 100ms 才能再次尝试。在回滚操作需要快速获取锁时，这可能引入不必要的延迟。

**建议：** 这不是严重问题，但可将轮询间隔缩短至 10ms，或直接使用 `m.mu.TryLock()` 的忙等（带 yield）配合 `runtime.Gosched()`，因为回滚操作本身是低频且短暂的。

---

## 四、值得肯定的改进（v1 → v2）

| 改进项 | 评价 |
|---|---|
| **CGO_ENABLED=0 + modernc.org/sqlite** | 正确解决了 v1 中 SQLite 与 CGO 的矛盾。 |
| **UPDATE ... RETURNING** | 彻底修复了 `NextVersion` 的并发竞态。 |
| **conflicts 表存储完整 Change JSON** | 避免了 change_log 截断后的悬空外键问题，符合 PRD。 |
| **sync_records 表** | 补齐了 PRD §7.2 的数据模型，支持历史记录接口。 |
| ** BIGINT 时间戳** | 消除了 PostgreSQL 迁移时的 32 位溢出隐患。 |
| **移除 JWT_SECRET / device_id label / 修复 create 合并** | 对 v1 评审意见的响应非常到位。 |
| **audit_log 表** | 主动增强了 PRD §9.3 的审计能力。 |
| **pnpm + shadcn/ui（Web 端）+ 手写组件（插件）** | 工具链统一且兼顾了体积与开发效率。 |

---

## 五、修复优先级清单

| 优先级 | 问题编号 | 修复内容 |
|---|---|---|
| P0 | 1 | 冲突检测补充 `create` 冲突逻辑（同 parent_guid + 同名 + 同 URL） |
| P0 | 2 | `NormalizeWeights` 在同一事务内完成版本号分配 + `change_log` 写入 |
| P1 | 3 | `PendingChange` 接口增加 `base_version` 字段 |
| P1 | 4 | Dockerfile 复制 `pnpm-lock.yaml` 以支持 `--frozen-lockfile` |
| P1 | 5 | 集成测试脚本 `run.sh` 统一改为 `pnpm run build` |
| P1 | 6 | 文档补充插件首次配置时的动态权限申请流程 |
| P1 | 7 | 调整 `classifyConflict` 优先级顺序 |
| P2 | 8-11 | 按建议优化项补充说明或预留接口 |

---

**总结：** `TECHNICAL-DESIGN-v2.md` 是一份**高质量的技术方案**，v1 中的核心算法缺陷、数据模型错误和部署问题均已得到妥善修复。剩余的问题主要集中在**边界场景（create 冲突、base_version 字段）**和**工程细节（pnpm lock 文件、事务完整性）**。建议在 Phase 1 启动前用 1-2 小时修正 P0/P1 项，即可进入开发。
