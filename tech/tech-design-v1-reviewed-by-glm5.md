# TECHNICAL-DESIGN-v1.md 审计报告

**审计对象:** `TECHNICAL-DESIGN-v1.md`（基于 PRD v6.0）  
**审计人:** GLM-5 (大网管)  
**审计日期:** 2026-04-16  
**对照文档:** PRD-v6.md (60KB, 完整阅读)

---

## 总评：7.0 / 10

技术方案整体架构清晰、选型务实，但**示例代码有多处与 PRD 语义不一致甚至直接矛盾的地方**，若干关键设计细节缺失。作为 v1 技术方案可以进入评审讨论，但**不能直接当作开发蓝图使用**——P0 项必须在编码前修正。

---

## 1. 优点

1. **架构分层清晰** — Chrome 插件、Web 管理界面、Go API、SQLite 四层职责分明，和 PRD 完全对齐。
2. **双数据库策略务实** — SQLite 默认 + PostgreSQL 可切换，附带了类型映射和分区建议。
3. **Schema 覆盖完整** — 7 张表覆盖了 PRD 中所有核心实体（bookmarks、change_log、snapshots、conflicts、devices、api_keys、global_state）。
4. **风险表落地** — 三根映射、SW 休眠、storage 配额、部分冲突重试等风险识别到位。
5. **监控与告警有雏形** — Prometheus 指标 + 告警规则 + Docker 部署方案，运维友好。
6. **测试示例可读性好** — 服务端/插件/集成测试三级测试策略结构合理。

---

## 2. 严重问题 (P0) — 编码前必须修正

### 2.1 `NextVersion` 并发原子性缺陷

**现状:** 伪代码分三步执行 — `SELECT` 读当前值 → `UPDATE +1` → 再 `SELECT` 读新值。注释说"SQLite: UPDATE ... RETURNING"但实际代码没有用 `RETURNING`。

**问题:** 
- 三步操作不在同一事务中，并发请求下可能读到不一致状态
- SQLite 虽然 WAL 模式下支持并发读，但写是串行的 — 如果 GORM 用了连接池，两次 SELECT 可能走不同连接
- 未来切 PostgreSQL 时问题更严重

**PRD 要求:** "每条变更对应全局版本号 +1"，`to_version` 必须全局唯一。

**建议:** 
```go
// 单事务原子递增
func (m *VersionManager) NextVersion(tx *gorm.DB) (int64, error) {
    var state GlobalState
    // SQLite: 利用写锁串行化
    // PG: SELECT ... FOR UPDATE
    err := tx.Model(&state).Where("id = ?", 1).
        Clauses(clause.Returning{Columns: []clause.Column{{Name: "current_version"}}}).
        Update("current_version", gorm.Expr("current_version + 1")).Error
    return state.CurrentVersion, err
}
```
同时文档中必须明确：**所有变更写入必须在同一事务中完成版本号递增 + change_log 插入**。

---

### 2.2 冲突检测算法与 PRD 语义不一致

**现状:** `Detect(deviceID string, baseVersion int64, changes []Change)` — `baseVersion` 参数未在查询中使用，实际用每条 `change.FromVersion`。

**PRD v6 明确规定:**
- 客户端发送 `base_version`（单个整数）
- 冲突条件：`base_version < 该 GUID 最近一次被修改的 to_version`
- 不是每条变更自带 `from_version` 做判定

**PRD 原文:** "设备 A 推送变更：guid='X', base_version=120 → 服务端检查：guid='X' 最近被修改的 to_version=125 → 判定：120 < 125 → 冲突"

**建议:** 
1. 入参改为 `baseVersion`（去掉 `changes[].FromVersion` 的判定逻辑）
2. 冲突判定改为：对每条变更查 `guid` 最新 `to_version`，与整批 `base_version` 比较
3. `from_version` 和 `to_version` 是**写入 change_log 时服务端填充**的，不是客户端传入的

---

### 2.3 `NormalizeWeights` 使用 PostgreSQL 语法，SQLite 下直接报错

**现状:** `db.Where("type = 'reindex' AND data->>'$.guid' = ?", parentGUID)`

**问题:**
- `data->>'$.guid'` 是 PostgreSQL 的 `jsonb` 操作符语法
- SQLite 应使用 `json_extract(data, '$.guid')`
- 更严重的是：`change_log` 表已有独立的 `guid` 列，根本不需要从 `data` JSON 中提取

**建议:** 直接改为 `db.Where("type = 'reindex' AND guid = ?", parentGUID)`，简单高效且双数据库兼容。

---

### 2.4 变更日志截断策略与 PRD 不一致

**现状:** 环境变量中定义了 `CHANGE_LOG_MAX_COUNT=1000` 和 `CHANGE_LOG_MAX_AGE_DAYS=7`，但未描述截断的具体逻辑和计数规则。

**PRD v6 明确规定:**
- 仅统计 `create/update/delete/move` 类型的变更（不含 `reindex` 和 `rollback`）
- 截断时更新 `baseline_version` 到截断点的最小 `from_version`
- 截断条件是**满足任一**（条数 > 1000 OR 最旧记录年龄 > 7 天）
- 截断后保留：最近 1000 条 **且** 年龄 ≤ 7 天

**缺失:** 
1. 没有截断实现代码或伪代码
2. `global_state` 表有 `baseline_version` 和 `last_truncated_at` 字段，但无使用说明
3. 未说明截断与快照更新的原子性要求

---

### 2.5 `bookmarks` 表缺少 `ON DELETE` 行为定义

**现状:** `FOREIGN KEY (parent_guid) REFERENCES bookmarks(guid)` — 无 `ON DELETE` 子句。

**问题:** Chrome 删除父文件夹时，子节点应级联删除或标记为软删除。当前 schema 下，如果物理删除父节点，外键约束会导致子节点无法存在（SQLite 默认 `RESTRICT`）。

**PRD v6 要求:** 软删除（`is_deleted=true`），90 天后物理删除。

**建议:** 
1. 软删除场景下外键不受影响（记录仍在）
2. 物理删除时需要先递归删除所有子节点
3. 文档需明确："物理删除前必须递归处理子树，不能依赖数据库级联"
4. 或改为 `ON DELETE SET NULL`（但 PRD 不允许 parent_guid 为 NULL）

---

## 3. 重要问题 (P1) — Phase 1 内必须解决

### 3.1 Web 管理端技术栈完全缺失

**现状:** 架构图标注 "Web 管理界面 (React SPA)"，但全文无任何 React 相关选型、依赖版本、构建配置。Chrome 插件选了 Preact，两套框架混用增加维护成本。

**建议:** 补充 Web 前端章节，明确：
- React vs Preact 的选型理由和边界
- 与插件共享的 TypeScript 类型/协议定义方式
- 构建工具链（是否也用 Vite）
- LPK 框架对 React 的集成方式

---

### 3.2 鉴权模型矛盾 — API Key vs JWT

**现状:** 环境变量出现 `JWT_SECRET=your-secret-key`，但全文和 PRD 都只描述了 API Key 鉴权。

**PRD v6 明确:** 只使用 API Key + bcrypt 哈希，没有 JWT。

**建议:** 删除 `JWT_SECRET` 环境变量，或明确说明 JWT 的使用场景（如 Web 管理端的 session token）。

---

### 3.3 Chrome 扩展权限过大

**现状:** `host_permissions: ["https://*/*", "http://*/*"]`

**问题:** 
- 攻击面大，Chrome Web Store 审核可能被拒
- PRD 明确："HTTPS（默认）/ HTTP（内网可选）"，用户只连接自己的懒猫微服
- `downloads` 权限用途未说明

**建议:** 
- 使用 `optional_host_permissions`，用户配置 API 地址后动态授权
- 或仅匹配用户配置的 API 基址域名
- 说明 `downloads` 权限用于本地备份（JSON 导出到 chrome.downloads）

---

### 3.4 全局 `sync.Mutex` 在多实例部署下无效

**现状:** PRD 描述的回滚操作使用全局写锁（`sync.Mutex`），但 `sync.Mutex` 只在单进程内有效。

**建议:** 文档中明确标注 "v1 假设单实例部署"，或改用数据库级锁（如 `SELECT ... FOR UPDATE`）。

---

### 3.5 `classifyConflict` 逻辑不完整

**现状:** 只区分了 `delete_conflict`、`move_conflict`、`update_conflict` 三种。

**PRD v6 定义了五种冲突类型:**
- 更新冲突
- 删除冲突
- 移动冲突
- 重命名冲突 — **缺失**
- 创建冲突 — **完全缺失**（同 parent_guid + 同名 + 同 URL 的合并逻辑）

**建议:** 
1. 补充 `rename_conflict`（title 不同但其他字段相同）
2. 补充 `create_conflict` 的检测逻辑（无 GUID，按 parent_guid + name + URL 匹配）
3. 创建冲突不应进入通用冲突流程，应自动合并

---

### 3.6 Docker HEALTHCHECK 使用 wget 但未安装

**现状:** `CMD wget -qO- http://localhost:8080/health || exit 1`，但运行时镜像 `alpine:3.19` 仅安装了 `ca-certificates`。

**Alpine 默认不带 `wget`**，健康检查永远失败。

**建议:** 改为 `CMD ["wget", "--spider", "-q", "http://localhost:8080/health"]` 并在 Dockerfile 中 `apk add --no-cache wget`，或改用 `/health` 端点返回特殊响应后用 shell 内置命令检测。

---

### 3.7 Prometheus 指标标签高基数风险

**现状:** `syncRequestsTotal` 和 `syncDuration` 使用 `device_id` 作为标签。

**问题:** 每个设备一个时间序列，设备数量增长时 Prometheus 存储和查询性能急剧下降。

**建议:** 
- `device_id` 移到日志中，metrics 只保留 `status` 标签
- 或对 device_id 哈希分桶（如 `device_bucket="0-99"`）

---

### 3.8 告警 PromQL 表达式有误

**现状:** 
- `rate(lzc_conflicts_total[5m]) > 10` — 注释说"10 次/分钟"，但 `rate()` 的单位是**每秒**，阈值应为 `10/60 ≈ 0.167`
- `SyncErrorRate` 分母可能为 0，产生 NaN

**建议:** 
- `HighConflictRate`: 改为 `rate(lzc_conflicts_total[5m]) > 0.167` 或用 `increase()` 替代
- `SyncErrorRate`: 加上 `> 0` 保护或用 `ignoring` 规则

---

## 4. 一般问题 (P2) — 建议改善

### 4.1 Go 版本偏旧

`golang:1.21-alpine` 已不在主动支持周期内。建议改为 **1.22+** 或 **1.23+**。

### 4.2 测试示例引用未定义方法

- `Queue` 测试调用 `mergeExcess()`（私有方法）和 `getAll()`（类中未定义）
- `TestSyncHandler_PartialConflict` 同时断言 `resp.NewVersion` 和 `resp.Version`，但 API 章节未定义这两个字段的区别

**PRD v6 区分了 `new_version`（客户端新基线）和 `version`（全局最新版本），技术方案应明确对应关系。**

### 4.3 集成测试脚本工具名不一致

注释写 "Puppeteer"，命令用 `npx playwright test`，应统一。

### 4.4 `snapshots` 表存储格式描述不足

PRD v6 要求快照使用**嵌套树结构**（`BookmarkNode` 含 `children` 递归），技术方案仅说 "JSON（gzip 压缩后 base64）"，未明确嵌套 vs 扁平格式。

### 4.5 LPK 框架集成细节不足

文档说 "LPK 框架提供开箱即用能力"，但未说明：
- LPK 如何注入配置（环境变量？配置文件？）
- 健康检查端点是 LPK 自动注册还是手动实现
- LPK 的日志格式要求
- LPK 的 Docker 构建流程与标准 Dockerfile 的差异

### 4.6 `conflicts` 表外键约束可能影响截断

`conflicts` 表通过外键引用 `change_log(change_id)`，但变更日志截断时会删除旧记录。如果冲突未解决，截断会导致外键冲突。

**建议:** 截断前先检查是否有 pending 状态的冲突引用，或改为不加外键约束（应用层保证一致性）。

### 4.7 部分冲突重试的客户端不变量未定义

PRD v6 详细描述了部分冲突后客户端的行为（更新 last_known_version、保留冲突变更、更新 base_version），但技术方案中离线队列 `Queue.ts` 的实现未体现这些不变量。

### 4.8 环境变量中缺少部分 PRD 要求的配置

PRD 提到的以下配置在技术方案中缺失：
- 自动同步间隔（默认 30 秒）
- 软删除物理删除天数（90 天）
- 单用户书签总数上限（100,000）
- 单次 sync 变更数上限（500）

---

## 5. 与 PRD v6 的需求追踪矩阵 (RTM)

| PRD 需求 | 技术方案覆盖 | 备注 |
|----------|-------------|------|
| §4.1 双层 ID 映射 + GUID 规范树 | ✅ | Schema 正确，映射逻辑在插件端 |
| §4.2 版本号 +1 语义 | ⚠️ | `NextVersion` 原子性不足 (P0) |
| §4.2 变更日志截断策略 | ❌ | 无截断实现，计数规则未说明 (P0) |
| §4.3 冲突检测（base_version 判定） | ❌ | 算法用 from_version 而非 base_version (P0) |
| §4.3 五种冲突类型 | ❌ | 缺少 rename_conflict 和 create_conflict (P1) |
| §4.4 排序权重设计 | ⚠️ | 算法正确，但归一化查询语法错误 (P0) |
| §4.5 Chrome 三根映射 | ✅ | PRD 中已有详细描述 |
| §4.6 回滚操作机制 | ⚠️ | 有全局写锁设计，但 sync.Mutex 多实例无效 (P1) |
| §5.1 API 鉴权管理 | ⚠️ | API Key 设计正确，但 JWT_SECRET 多余 (P1) |
| §5.2.4 离线队列管理 | ⚠️ | Queue.ts 基本框架有，但部分冲突不变量缺失 |
| §6.2 认证接口 | ✅ | Schema 支持 |
| §6.3 书签接口 | ✅ | API 设计与 PRD 对齐 |
| §6.6 限流策略 | ⚠️ | 环境变量有，但实现未描述 |
| §7 数据模型 | ⚠️ | 大部分对齐，但 ON DELETE 行为缺失 |
| §9 安全考虑 | ✅ | 基本覆盖 |
| §12 验收标准 | ❌ | 技术方案未引用验收标准或给出可测试指标 |

**覆盖统计:** ✅ 完全覆盖 6/16 | ⚠️ 部分覆盖 7/16 | ❌ 缺失 3/16

---

## 6. 修订优先级

| 优先级 | 数量 | 关键项 |
|--------|------|--------|
| **P0** | 5 | 版本号原子性、冲突检测语义、SQLite 语法、截断策略、ON DELETE 行为 |
| **P1** | 8 | Web 前端栈、鉴权模型、扩展权限、多实例锁、冲突类型补全、Docker 健康检查、Metrics 基数、PromQL 修正 |
| **P2** | 8 | Go 版本、测试示例、工具名统一、快照格式、LPK 集成、外键冲突、部分冲突不变量、缺失配置 |

---

## 7. 总结

`TECHNICAL-DESIGN-v1.md` 作为第一版技术方案，**架构方向正确，技术选型合理**。但存在以下核心问题：

1. **示例代码不能当实现用** — 特别是 `NextVersion`、`Detect`、`NormalizeWeights` 三段核心代码都有硬伤
2. **与 PRD v6 的语义对齐有偏差** — 冲突检测的入参语义、变更日志截断规则、冲突类型覆盖
3. **运维细节需收紧** — Docker 健康检查、PromQL 表达式、扩展权限

**建议路径:** 修正 P0 → 评审通过 → Phase 1 编码（同步修正 P1）→ Phase 4 前完成 P2。

---

*审计仅基于 `TECHNICAL-DESIGN-v1.md` 与 `PRD-v6.md` 的对照分析，未包含对 LPK 框架实际 API 的验证。*
