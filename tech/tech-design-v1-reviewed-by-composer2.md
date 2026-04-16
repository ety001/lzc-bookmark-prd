# TECHNICAL-DESIGN-v1.md 审计报告

**审计对象:** `TECHNICAL-DESIGN-v1.md`（基于 PRD v6.0，2026-04-16）  
**审计人:** Composer 2  
**审计日期:** 2026-04-16  
**结论摘要:** 文档在架构分层、技术选型、库表轮廓与里程碑上整体可用，但算法与示例代码存在若干与 SQLite/并发/观测相关的硬伤；Web 端与鉴权链路表述不完整。建议在进入 Phase 1 前修正标为「高」的项并补齐假设说明。

---

## 1. 文档优点

- **架构清晰:** Chrome 插件、Web、Go API、存储分层与职责边界一目了然，和 PRD 方向一致。
- **双数据库取向合理:** SQLite 默认 + PostgreSQL 可切换，并单独列出类型与分区优化，利于个人部署与后续扩展。
- **Schema 覆盖面较全:** `api_keys`、`devices`、`bookmarks`、`change_log`、`global_state`、`snapshots`、`conflicts` 与 PRD 中的同步/版本/冲突/快照叙事基本对齐。
- **风险表务实:** 三根映射、SW 休眠、storage配额、部分冲突重试等风险与缓解方向正确。
- **运维向内容:** Docker/Compose、环境变量、Prometheus 指标与告警规则有雏形，便于落地讨论。

---

## 2. 严重问题（建议阻塞评审通过或必须在编码前修正）

### 2.1 权重归一化示例与 SQLite 不兼容

`NormalizeWeights` 中使用 `data->>'$.guid'` 及类似 JSON 运算符，这是 **PostgreSQL** 写法。主 schema 以 SQLite 为先，SQLite 应使用 `json_extract(data, '$.guid')` 等语法；且 `change_log` 已有 **`guid` 列**，节流查询更应直接使用 `guid = ? AND type = 'reindex'`，避免依赖 `data` 内是否冗余存 `guid`。

**影响:** 按文档照抄实现会在 SQLite 下直接查询失败或行为未定义。

### 2.2 `NextVersion` 并发与原子性不足

当前伪代码：`SELECT` → `UPDATE global_state SET current_version = current_version + 1` →再 `SELECT` 读新值。在并发请求下，若未放在 **单事务** 内或未使用 **单条语句原子递增**（如 `UPDATE ... RETURNING` / 事务内 `FOR UPDATE`），可能出现重复版本号或读到不一致状态。SQLite 虽串行化写，但 GORM 层若多连接或未来切 PG，问题会放大。

**建议:** 在文档中明确「版本递增必须在同一事务内完成」或采用数据库方言下的原子递增语句；并说明 **多副本部署下 mutex 无效**（见3.3）。

### 2.3 冲突检测条件与参数命名易误导

`Detect(deviceID string, baseVersion int64, changes []Change)` 中 **`baseVersion` 未在查询中使用**，实际用的是每条 `change.FromVersion`。若客户端某条变更的 `from_version` 与批次 `base_version` 不一致，行为与读者预期不符，也与「整批基于同一基线」的常见协议不同。

**建议:** 在文档中明确协议：是「每条变更自带 from_version」还是「整批共用 base_version」；伪代码应使用与协议一致的字段，避免未使用参数。

### 2.4冲突检测查询可能漏检或误判（需协议级说明）

当前逻辑：对每条变更，查该 `guid` 在 `to_version > change.FromVersion` 下的最新一条 `change_log`。未在文中说明：

- 同一 GUID 上 **删除后再创建**（新 GUID 与旧 GUID 不同则无碍；复用 GUID 则需规则）；
- **移动 / reindex** 与「更新冲突」分类是否覆盖 PRD 全部场景；
- 批量推送中 **多条变更针对同一 GUID** 时的检测顺序（应先拓扑排序还是拒绝同批重复）。

**建议:** 增加「冲突判定不变量」小节，与 PRD双向同步、部分应用规则逐条对照。

---

## 3. 重要问题（应在 Phase 1 内解决）

### 3.1 Web 管理端技术栈缺失且与架构图不一致

架构图写 **「Web 管理界面 (React SPA)」**，全文未给 Web 端依赖版本、构建方式、与插件共享类型/协议的说明；扩展选型为 **Preact**，两端不一致会增加重复与联调成本。

**建议:** 单列「Web 前端」章节，或明确 React 与 Preact 的边界（例如仅管理端 React、插件 Preact）及 API 契约来源（OpenAPI / 手写）。

### 3.2 鉴权模型不完整

环境变量出现 **`JWT_SECRET`**，正文未描述 JWT 颁发、刷新、与 **API Key** 的关系。PRD 若以 API Key 为主，JWT 易成冗余或实现分叉。

**建议:** 写清「仅 API Key」「JWT + Key」或「Session」之一，并删除无关配置或补全流程。

### 3.3 全局 `sync.Mutex` 与水平扩展

文档写明用 **`sync.Mutex` 做回滚全局写锁**。这在 **多进程 / 多实例** 下不成立。若懒猫微服未来多副本或滚动发布，需 **分布式锁或单主写入** 假设。

**建议:** 在架构假设中写明「v1 单实例」或引入 DB 级锁/租约。

### 3.4 Chrome 扩展权限面过大

`host_permissions` 为 `https://*/*` 与 `http://*/*`，攻击面与审核风险高。`downloads` 权限未在文中解释用途。

**建议:** 改为 **可选可配置源站** + `optional_host_permissions`，或最小化匹配用户配置的 API 基址；并说明 `downloads` 是否必需。

### 3.5 Docker 健康检查与 Alpine 工具链

`HEALTHCHECK` 使用 `wget`。Alpine 运行时镜像仅 `apk add ca-certificates`，**未必包含 wget**（需 `apk add wget` 或改用 `wget`/`curl` 镜像层）。

**建议:** 在 Dockerfile 中显式安装探测依赖，或复用带健康检查二进制的基础镜像。

### 3.6 观测指标高基数风险

`syncRequestsTotal` / `syncDuration` 使用 **`device_id` 标签**。设备数大时 Prometheus **时间序列基数** 会暴涨。

**建议:** 文档改为「仅对 device哈希分桶 / 抽样」或「日志中记 device，metrics 聚合为全局或按 tenant」。

### 3.7 告警表达式与指标语义

- `HighConflictRate` 使用 `rate(lzc_conflicts_total[5m]) > 10`：需明确 counter 是「累计」还是「按类型」；注释「10 次/分钟」与 `rate()` 单位需一致（通常 rate 已是每秒，阈值应换算）。
- `SyncErrorRate` 分母为 `rate(lzc_sync_requests_total[5m])`，若总量极低会产生 **除法不稳定**；应用 `> 0` 或 `ignoring` 规则。

**建议:** 在监控章节补「告警数学定义」与「静默窗口」说明。

---

## 4. 一般问题与笔误

### 4.1 测试示例与 API 不一致

- `TestSyncHandler_PartialConflict` 同时断言 `resp.NewVersion`、`resp.Version`，语义未在正文 API 章节定义，读者无法判断 **全局版本 vs 客户端新基线**。
- `Queue` 测试中调用 **`mergeExcess()`**（私有方法）与 **`getAll()`**（类片段中未定义），示例无法直接编译。

### 4.2集成测试脚本注释写「Puppeteer」，命令写 **`npx playwright test`**，工具名不一致。

### 4.3数据库细节

- `bookmarks` 自引用外键：删除父节点时 **ON DELETE** 行为未写；与 Chrome 树删除语义需对齐。
- `change_log` 中 `to_version` 全局唯一：与「部分变更未应用」场景下版本跳空是否一致，建议在正文解释。

### 4.4 运行时版本

`golang:1.21-alpine` 已偏旧，建议文档改为团队统一的 **1.22+ / 1.23+**（以安全与支持周期为准）。

### 4.5 参考链接

`LPK 框架文档` 指向 `github.com/lazycat/lpk-docs`：需确认为 **实际可读文档**，避免评审后链接失效。

---

## 5. 与 PRD 的衔接建议（未读 PRD 全文的保守项）

以下需在对照 PRD v6 时重点核对（本审计仅基于技术设计正文推断）：

- **部分冲突 / 重试** 后客户端 `base_version`、`last_known_version` 与未应用变更队列的 **不变量**；
- **快照 + 变更日志截断** 与 `baseline_version`、`last_truncated_at` 的协同；
- **多设备**下 `devices` 与 `api_keys` 生命周期（吊销、轮换）；
- **Web 管理** 是否需与插件 **同一套身份**（Cookie vs Key）。

---

## 6. 修订优先级建议

| 优先级 | 项 |
|--------|-----|
| P0 | 修正 SQLite 下 JSON 查询与归一化节流查询；明确版本号事务/原子递增；澄清冲突检测输入协议 |
| P1 | Web 栈章节；鉴权单一模型；扩展权限收敛；Docker HEALTHCHECK；mutex 多实例假设 |
| P2 | Metrics 标签基数；告警 PromQL 修正；测试示例可编译；Go 版本与工具链统一 |

---

## 7. 总结

`TECHNICAL-DESIGN-v1.md` 作为 **v1 技术方案** 结构完整、选型主流，适合驱动 Phase 1 讨论；但 **示例代码不能当作可直接落地的实现**（尤其 SQLite、版本并发、冲突入参），**安全与观测** 章节需要收紧假设与表达式。完成 P0/P1 修订后，文档可更好地作为评审基线与实现契约。

---

*本报告仅针对 `TECHNICAL-DESIGN-v1.md` 自身一致性与工程可行性，不替代对 `PRD-v6.md` 的逐条需求追踪（建议在技术设计中增加 RTM 表或章节引用）。*
