# TECHNICAL-DESIGN-v2.md 审计报告

**审计对象:** `TECHNICAL-DESIGN-v2.md`（基于 PRD v6.0，v2.1 修订）  
**审计人:** GLM-5 (大网管)  
**审计日期:** 2026-04-16  
**对照文档:** PRD-v6.md、TECHNICAL-DESIGN-v1 审计报告（GLM-5/Composer 2）

---

## 总评：8.5 / 10

v2 是一次**高质量的修订**。v1 审计中提出的 5 个 P0 和 8 个 P1 问题全部得到了明确回应和修复。架构方向不变，核心算法代码已可基本信任。剩余问题主要是**边界条件、截断实现缺失、以及部分伪代码细节**——不再阻塞 Phase 1 启动，但需在开发过程中同步解决。

**v1→v2 改进追踪：**

| v1 审计问题 | v2 状态 | 备注 |
|-------------|---------|------|
| P0: NextVersion 原子性 | ✅ 已修复 | UPDATE RETURNING + 事务 fallback |
| P0: 冲突检测用 from_version | ✅ 已修复 | 改为 baseVersion |
| P0: NormalizeWeights PG 语法 | ✅ 已修复 | 改用 guid 列 |
| P0: 截断策略未实现 | ⚠️ 部分修复 | 环境变量补全，但仍无截断伪代码 |
| P0: ON DELETE 行为 | ✅ 已修复 | 明确注释"递归处理子树" |
| P1: Web 前端栈缺失 | ✅ 已修复 | React + Vite + shadcn/ui |
| P1: JWT 矛盾 | ✅ 已修复 | 已删除 |
| P1: 扩展权限过大 | ✅ 已修复 | optional_host_permissions |
| P1: 多实例锁 | ✅ 已修复 | 注释明确单实例 |
| P1: 冲突类型不全 | ⚠️ 部分修复 | 补了 rename_conflict，create_conflict 仍缺 |
| P1: Docker HEALTHCHECK | ✅ 已修复 | 添加 wget |
| P1: Metrics 基数 | ✅ 已修复 | 移除 device_id |
| P1: PromQL 表达式 | ✅ 已修复 | rate > 0.167、除零保护 |

**统计:** ✅ 完全修复 12/14 | ⚠️ 部分修复 2/14

---

## 1. 优点（相比 v1 的增量改进）

1. **CGO 问题彻底解决** — 改用 `modernc.org/sqlite` 纯 Go 实现，Dockerfile 的 `CGO_ENABLED=0` 终于合理了。还提供了 `pgx` 作为 PG 驱动，选型连贯。
2. **NextVersion 双路径设计** — 主路径用 `RETURNING`（SQLite 3.35+/PG），fallback 用事务包裹。注释清晰，开发者不会踩坑。
3. **AcquireLock 修复 goroutine 泄漏** — `TryLock + ticker` 方案干净，不再有泄漏风险。
4. **冲突检测批量查询** — 一次查出所有 GUID 的最新变更，构建 map 后 O(1) 判定。解决了 v1 的 N+1 查询问题。
5. **conflicts 表去外键** — 存储 JSON 快照而非外键引用，截断时不再有 FK 冲突。
6. **新增 audit_log 表** — 对齐 PRD §9.3 审计日志要求。
7. **新增 sync_records 表** — 对齐 PRD §7.2 SyncRecord 模型。
8. **环境变量大幅补全** — 新增 SYNC_INTERVAL_SECONDS、SOFT_DELETE_DAYS、MAX_BOOKMARKS、MAX_SYNC_CHANGES，覆盖了 v1 审计中 4.8 提到的所有缺失项。
9. **前端构建集成到 Dockerfile** — 多阶段构建，Web 产物直接打包进最终镜像。
10. **插件 UI 改为手写组件** — 明确了不使用 content script、不使用 shadcn/ui 的理由（轻量、AI 友好），决策合理。

---

## 2. 严重问题 (P0) — 建议编码前修正

### 2.1 变更日志截断实现仍然缺失

v2 补全了环境变量和计数规则注释（"仅统计 create/update/delete/move"、"reindex 和 rollback 不计入上限"），但**没有给出截断的伪代码或流程描述**。

PRD v6 要求的截断行为很精确：
- 截断条件：普通变更 > 1000 OR 最旧记录年龄 > 7 天
- 截断时更新 `baseline_version`
- 截断与快照更新需要原子性
- 截断后落后设备走全量同步

`global_state` 表有 `baseline_version` 和 `last_truncated_at` 字段，但没有代码展示何时/如何使用它们。

**建议:** 即使不写完整实现，至少加一段截断流程伪代码：
```
1. 统计 change_log 中 type IN ('create','update','delete','move') 且 timestamp > (now - 7d) 的条数
2. 如果 > 1000 或有 7d 前的记录：
   a. 开启事务
   b. 计算保留范围：min(to_version) WHERE 满足条件的最新 1000 条
   c. DELETE FROM change_log WHERE to_version < 保留范围最小值
   d. UPDATE global_state SET baseline_version = 保留范围最小值, last_truncated_at = now
   e. 生成/更新 baseline snapshot
   f. 提交事务
```

---

### 2.2 `create_conflict` 检测逻辑仍然缺失

`classifyConflict` 新增了 `rename_conflict`（title 不同），但 PRD v6 定义的**创建冲突**（同 `parent_guid` + 同名 + 同 URL 的书签合并）在冲突检测算法中完全没有体现。

`Detect` 函数的入口前提是 `changes` 已有 GUID，但创建冲突的场景是：**两个设备同时创建了相同 URL 的书签，此时还没有 GUID**。

**PRD 原文:** "创建冲突：同一 parent_guid + 同名 + 同 URL 的书签（无 GUID）→ 仅同文件夹内合并"

**建议:** 
1. 在 `Detect` 中增加一条规则：如果 change.type == "create" 且 change.guid == null，检查同 parent_guid 下是否已存在同名同 URL 的书签
2. 或在 sync handler 层面、GUID 分配之前做去重检测
3. 创建冲突应自动合并（不需用户干预），与普通冲突的交互流程不同

---

### 2.3 批量冲突检测查询可能返回错误的"最新变更"

**现状:**
```go
err := d.db.Where("guid IN ? AND to_version > ?", guids, baseVersion).
    Order("to_version DESC").
    Find(&latestChanges).Error
```

**问题:** 这个查询返回的是**所有**满足 `to_version > baseVersion` 的变更记录（按版本倒序），然后代码用 map 取第一个出现的作为"最新"。但如果一个 GUID 有多条变更，`Find` 返回所有记录后 map 只保留第一条——由于 `ORDER BY to_version DESC`，这确实是最新的一条，逻辑正确。

**但有一个边界情况:** 如果同一批次推送中，两条变更涉及同一个 GUID（如先 create 再 move），`guids` 会有重复。建议先对 guids 去重：
```go
guidSet := make(map[string]struct{})
for _, change := range changes {
    if change.GUID != "" {
        guidSet[change.GUID] = struct{}{}
    }
}
guids := make([]string, 0, len(guidSet))
for g := range guidSet {
    guids = append(guids, g)
}
```

这是个小问题，不影响正确性（数据库查询对重复值也有效），但体现了对边界场景的考虑。

---

## 3. 重要问题 (P1) — Phase 1 内解决

### 3.1 `NormalizeWeights` 事务中更新了 bookmarks 但未更新 change_log

**现状:** 事务内做了 `tx.Model(&child).Update("index_weight", newWeight)` 循环更新，然后构造了一个 `Change` 对象但注释掉了插入逻辑：`// err = tx.Create(change).Error`。

**问题:** 
1. bookmarks 已被更新，但 change_log 未写入 → 客户端无法通过增量拉取知道权重变了
2. 如果事务后续失败回滚，bookmarks 更新也会回滚（这是对的），但如果 `Change` 写入在事务外就危险了

**建议:** 取消注释 `tx.Create(change)`，确保 reindex 变更和权重更新在同一事务内。同时 `FromVersion/ToVersion` 应在事务内通过 `NextVersion` 填充。

---

### 3.2 `NextVersionSQLite` 的 SELECT + UPDATE 仍有极小的竞态窗口

**现状:** 事务内 `SELECT current_version` → `newVersion++` → `UPDATE`。SQLite 的写锁在事务级别串行化，所以在**默认的 serialized 模式**下是安全的。

但如果 GORM 配置了 `WAL journal mode` + `busy_timeout`，多个并发事务可能短暂重叠。现代 SQLite（3.35+）普遍支持 `RETURNING`，建议文档中明确**要求 SQLite 3.35+** 作为最低版本，这样 `NextVersion` 主路径就够了，`NextVersionSQLite` 可以作为 fallback 或删除。

---

### 3.3 `host_permissions` 中的默认值不合理

**现状:** `host_permissions: ["https://api.example.com/*"]` — 这是一个占位域名。

**问题:** 用户首次安装时还没配置 API 地址，这个硬编码域名没有意义。应该默认为空 `[]`，用户配置 API 地址后通过 `chrome.permissions.request()` 动态添加。

**建议:**
```json
"host_permissions": [],
"optional_host_permissions": [
    "<all_urls>"
]
```
安装时 host_permissions 为空，用户在 options 页面配置 API 地址后动态授权。

---

### 3.4 `sync_records` 表与 `change_log` 的关系未说明

**现状:** 新增了 `sync_records` 表（version, device_id, change_ids JSON, snapshot_hash, synced_at），但没有说明：
- 何时写入？（每次 sync 请求后？还是定期汇总？）
- `change_ids` 与 `change_log.change_id` 的对应关系
- 是否与 PRD §7.2 的 SyncRecord 模型完全对齐
- `version` 是 `to_version` 的最大值还是 sync 请求的 `new_version`

**PRD §7.2 定义:**
```typescript
SyncRecord {
  version: integer
  device_id: string
  change_ids: string[]    // 本次 sync 产生的变更 ID 列表
  snapshot_hash: string   // 快照 SHA256
  synced_at: integer
}
```

**建议:** 补充写入时机说明和查询用途（如历史接口 `GET /history` 展示 "版本 125，设备 A，5 条变更"）。

---

### 3.5 部分冲突后客户端不变量描述分散

PRD v6 详细定义了部分冲突后的客户端行为（更新 last_known_version、保留冲突变更、更新 base_version），但 v2 中这些逻辑分散在：
- `SyncEngine.test.ts` 的断言中
- `Queue.ts` 的 markSubmitted 中
- 但没有集中的**状态机或不变量文档**

**建议:** 在技术方案中增加一节"同步状态不变量"，明确：
```
1. last_known_version 始终等于客户端已成功应用的最高版本
2. 离线队列中 submitted=true 的变更已服务端确认
3. 未提交变更的 base_version 始终等于 last_known_version
4. 冲突变更不参与队列合并
```

---

### 3.6 `snapshots` 表的存储格式需明确

PRD v6 要求快照使用**嵌套树结构**（`BookmarkNode` 含 `children` 递归），v2 的注释改为"JSON（嵌套树结构，可选 gzip 压缩）"，但"可选"很模糊。

**建议:** 明确为 gzip 压缩后 base64 编码存储，或明确为原生 JSON（如果 SQLite TEXT 列够大）。PRD 估算 5000 书签的快照 gzip 后约 100-150KB，应该在实现时统一。

---

## 4. 一般问题 (P2) — 建议改善

### 4.1 测试示例仍有不可编译的方法调用

- `Queue` 测试中 `await queue.getAll()` — 类定义中没有 `getAll()` 方法
- `SyncEngine.test.ts` 中 `response` 对象用 `...,` 省略了数组元素，不是合法 TypeScript

### 4.2 集成测试脚本仍用 npm

注释写着 `cd extension && npm run build`，但已改为 pnpm。应统一为 `pnpm run build`。

### 4.3 `audit_log` 表没有清理策略

审计日志会持续增长（100MB/百万条），但未定义清理策略。建议加 `AUDIT_LOG_MAX_AGE_DAYS=365` 环境变量。

### 4.4 LPK 框架集成仍偏抽象

列出了"开箱即用能力"（健康检查、日志、指标），但没有说明 LPK 注入配置的方式、日志格式要求、Docker 构建差异。这可能是 LPK 文档本身的问题，但建议在技术方案中注明"参见 LPK 文档 §X"。

### 4.5 `InsertWithWeight` 未考虑并发插入

两个设备同时往同一位置插入书签时，可能计算出相同的 weight。归一化会修复，但会触发额外的 reindex 变更。

这实际上不是 bug（归一化机制就是为此设计的），但建议在注释中说明这是预期行为。

### 4.6 `conflicts` 表的 `resolution` 缺少 merge 的 merged_data

PRD v6 的冲突解决支持 `merge` 类型，需要额外传入 `merged_data`。但 `conflicts` 表只有 `resolution` 字段，没有 `merged_data` 列。

**建议:** 增加一列 `merged_data TEXT` 存储 merge 后的数据 JSON。

### 4.7 前端 Dockerfile 阶段用了 `package*.json`

```dockerfile
COPY web/package*.json ./
RUN pnpm install --frozen-lockfile
```

pnpm 使用 `pnpm-lock.yaml`，不是 `package-lock.json`。应改为：
```dockerfile
COPY web/package.json web/pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
```

### 4.8 Docker Compose version 字段已废弃

`version: '3.8'` 在现代 Docker Compose V2 中已忽略并产生警告。建议删除该行。

---

## 5. 与 PRD v6 的需求追踪矩阵 (RTM)

| PRD 需求 | v1 状态 | v2 状态 | 备注 |
|----------|---------|---------|------|
| §4.1 双层 ID 映射 + GUID 规范树 | ✅ | ✅ | 无变化 |
| §4.2 版本号 +1 语义 | ⚠️ 原子性不足 | ✅ | RETURNING + 事务 fallback |
| §4.2 变更日志截断策略 | ❌ 无实现 | ⚠️ | 环境变量补全，但无截断伪代码 (P0) |
| §4.3 冲突检测（base_version） | ❌ 用了 from_version | ✅ | 已改为 baseVersion |
| §4.3 五种冲突类型 | ❌ 缺两种 | ⚠️ | 补了 rename，create_conflict 仍缺 (P0) |
| §4.4 排序权重设计 | ⚠️ 语法错误 | ✅ | 修复语法 + 添加事务 |
| §4.5 Chrome 三根映射 | ✅ | ✅ | 无变化 |
| §4.6 回滚操作机制 | ⚠️ 多实例锁 | ✅ | 注释明确单实例 |
| §5.1 API 鉴权管理 | ⚠️ JWT 多余 | ✅ | 已删除 JWT_SECRET |
| §5.2.4 离线队列管理 | ⚠️ | ✅ | create 合并修复 |
| §6 API 设计 | ✅ | ✅ | 无变化 |
| §7 数据模型 | ⚠️ ON DELETE | ✅ | 注释明确递归处理 |
| §7.2 SyncRecord | ❌ 缺失 | ⚠️ | 新增表，但关系未说明 (P1) |
| §7.4 Conflict | ⚠️ FK 截断问题 | ✅ | 去外键存 JSON |
| §9 安全考虑 | ✅ | ✅ | 新增 audit_log |
| §12 验收标准 | ❌ 未引用 | ❌ | 仍未引用或给出可测试指标 |

**覆盖统计:** ✅ 完全覆盖 10/16 | ⚠️ 部分覆盖 5/16 | ❌ 缺失 1/16

**v1→v2 改善:** ✅ 从 6→10 | ⚠️ 从 7→5 | ❌ 从 3→1

---

## 6. 修订优先级

| 优先级 | 数量 | 关键项 |
|--------|------|--------|
| **P0** | 3 | 截断实现伪代码、create_conflict 检测、批量查询 guids 去重（小） |
| **P1** | 6 | 归一化事务内写 change_log、SQLite 3.35+ 要求、host_permissions 默认值、sync_records 说明、同步不变量文档、快照存储格式 |
| **P2** | 8 | 测试可编译性、pnpm 统一、audit_log 清理、LPK 文档引用、并发 weight 注释、merge merged_data、Dockerfile pnpm-lock、Docker Compose version |

---

## 7. 总结

`TECHNICAL-DESIGN-v2.md` 是一份**成熟度显著提升**的技术方案。v1 中的核心硬伤（版本号原子性、冲突检测语义、SQLite 语法、外键截断）全部得到了正确修复。新增的 audit_log、sync_records、Web 前端栈、Docker 多阶段构建都是实实在在的改进。

**剩余风险集中在两个方向：**
1. **截断实现** — 这是最复杂的运维逻辑，没有伪代码开发时容易出错
2. **创建冲突** — 这是 PRD 唯一定义了但技术方案完全没有覆盖的冲突类型

**建议路径:** 补充截断伪代码 + create_conflict 逻辑后 → 评审通过 → Phase 1 启动（同步处理 P1）→ Phase 4 前完成 P2。

可以开始干了。🥷

---

*审计基于 `TECHNICAL-DESIGN-v2.md`（v2.1 修订）与 `PRD-v6.md` 的对照分析，同时追踪了 v1 审计报告（GLM-5）中的 14 个问题。*
