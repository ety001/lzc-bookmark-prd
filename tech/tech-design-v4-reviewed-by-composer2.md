# 懒猫书签同步系统技术方案 v4.0 审计报告

**审计者:** Composer2  
**审计日期:** 2026-04-17  
**审计对象:** `TECHNICAL-DESIGN-v4.md`（v4.0，声称基于 PRD v6.0）  
**交叉核对:** `prd/PRD-v6.md`（§4.2 溢出、§6.1错误码、§6.3.1 拉取、§6.3.2 同步）

---

## 执行摘要

v4.0 在 v3 基础上做了大量工程化修补（`ChangeData` 与 GORM `serializer:json`、空 `guids` 短路、截断语义说明、Dockerfile 最小拷贝、`busy_timeout`、健康检查与验收表等），**作为「实现草案」质量明显高于 v3**。

但与 **PRD v6.0 的逐条对齐尚未成立**：错误码表与 HTTP 语义、**全量同步触发时的状态码**、同步响应字段（`generated_mappings`、`synced_at` 等）与 **GET `/api/v1/bookmarks` 分页拉取**在 v4 中缺失或与 PRD 矛盾。若产品以 PRD 为唯一契约，当前 v4 **不能**标注为「已与 PRD v6 一致」；建议要么修订技术方案，要么在 PRD 中显式改版并统一版本号。

| 级别 | 数量 | 说明 |
|------|------|------|
| **P0** | 2 | 与 PRD 冲突：错误码体系；全量同步（截断/落后）时 POST sync 的 HTTP/业务码 |
| **P1** | 6 | 响应体字段缺失；不变式过简；里程碑/文末路径；纯 create 批次的冲突检测空白等 |
| **P2** | 5 | 表述精确性、伪代码一致性、修订摘要与正文是否兑现 |

---

## P0 — 阻塞「PRD 契约一致性」声明的问题

### P0.1 错误码 `E001`–`E008` 与 PRD v6 不一致

**位置:** v4 §4.4；PRD §6.1 错误码表（约 L709–L719）

**PRD v6 定义（摘要）:**

| 错误码 | HTTP | 含义（PRD） |
|--------|------|-------------|
| E001 | 401 | API Key 无效或缺失 |
| E002 | 403 | 权限不足（只读 Key 写接口） |
| E003 | **409** | **版本冲突 / base严重落后 / 需全量同步** |
| E004 | 400 | 设备未注册 |
| E005 | 400 | 数据格式错误 |
| E006 | 429 | 限流 |
| E007 | 500 | 服务端错误 |
| E008 | 410 | 维护模式 |

**v4 §4.4** 将 `E001` 设为 base_version 无效、`E002` 为变更数超限、`E003` 为 401、`E004` 为截断等，**与 PRD 同一符号完全不同义**。

**影响:** 客户端、共享包 `@lzc/shared`、集成测试若按 PRD 实现，将与 v4 文档直接冲突。

**建议:** 全文统一采用 **PRD 错误码表**；v4 中「参数类」错误归入 PRD 已有码（如设备、格式、限流）或走 **PRD 变更流程**新增独立码，**禁止复用 E001–E008 表示另一套语义**。

---

### P0.2 「版本溢出 / 截断」时 POST sync 的响应：v4 与 PRD 矛盾

**位置:** v4 §4.3（版本溢出行）、§4.5、§4.4 `E004`；PRD §6.3.2「客户端状态更新规则」（约 L986–L988）、§6.1 `E003` 说明（约 L721–L726）

**PRD 明确要求:** 需强制全量同步时，客户端 **收到 `409` + `E003`**，然后 **GET `/api/v1/bookmarks` 无 `since_version`**（全量拉树）。

**v4 要求:**同类场景下 **HTTP 200**，`success: true`，`data.full_sync: true`（§4.3、§4.5），与 PRD **不一致**。

**说明:** PRD §4.2 同时规定 **GET 书签**在溢出场景可返回 **200 + 规范树 + `is_baseline`**；那是 **拉取接口**行为，**不**等同于 POST sync 仍返回 200 + `full_sync` 即可替代 PRD 对 **409 E003** 的约定。当前 v4 将两类场景混为一谈，且未定义与 PRD 一致的 **`409` + 错误体**。

**建议:**  
- **方案 A（推荐，贴合现有 PRD）:** POST sync 在 `base_version < baseline_version` 等场景返回 **409 + E003**（`details` 中带 `current_version` / `baseline_version` 等）；全量树仍由 **GET `/bookmarks`** 完成。  
- **方案 B:** 推动 **PRD v7** 正式改为「POST sync 200 + `full_sync`」，并废弃 §6.3.2 中「收到 409 E003」的表述，全仓库统一。

在未改版 PRD 前，**以 PRD 为准**。

---

## P1 — 重要缺口与易误导实现的问题

### P1.1 同步响应体相对 PRD 缺字段

**位置:** v4 §4.2.2；PRD §6.3.2 成功/部分冲突示例（约 L933–L970）

PRD 响应含 **`generated_mappings`**（`local_id` → `guid`）、顶层 **`synced_at`**、冲突项上的 **`resolution_hint`** 等；v4 `SyncResponse` / `Conflict` **未包含**。

**建议:** 与 PRD 示例逐项对齐，或在文档中显式写「v1 实现裁剪清单」并同步改 PRD，避免插件与 Web 各写一套。

---

### P1.2 §4.3「响应不变式」与 PRD 示例数值关系不一致

**位置:** v4 §4.3

「完全成功」行写 `new_version = base_version + len(changes)` —仅在 **每条变更严格对应全局版本 +1 且全部应用** 时成立。PRD 部分冲突示例中 **`new_version`（123）与顶层 `version`（128）可分离**（其他设备变更），**不能**用 `base + len(applied)` 一言以蔽之。

**建议:** 改为不变式描述：例如「`new_version` = 本请求应用完成后**本客户端应对齐的服务端版本**；`version` = **当前全局最新**；并引用 PRD 示例」。

---

### P1.3 修订摘要称「增量同步分页」但正文几乎未定义拉取协议

**位置:** v4 文首 P2 第 8 条「增量同步返回添加分页支持」；正文仅 §4.1 一句「从 new_version 到 version」

PRD **§6.3.1** 对 `GET /api/v1/bookmarks?since_version=&limit=`、`has_more`、续页与截断连续性有完整约定。**v4 未独立成节**，实现易与 PRD 分叉。

**建议:** 增加 **§4.x「增量拉取（GET /bookmarks）」**，或明确引用 PRD 章节编号与字段名。

---

### P1.4 `classifyConflict` 中「create + create 相同则 `update_conflict`」可读性与语义

**位置:** v4 §6.1

两笔 `create` 且 title/URL 相同返回 `update_conflict`（L665），对读者不直观；若语义为「等价于已存在节点上的更新」，建议在注释或协议中写清，并与 PRD「创建冲突」定义（同 parent + 同名 + 同 URL）对照，避免与 **create_conflict** 枚举混用。

---

### P1.5 `Detect` 在 `len(guids)==0` 时直接返回与 PRD「创建冲突」

**位置:** v4 §6.1

整批仅 **create、尚无 GUID** 时，提前返回则 **不会** 做「同 parent + 同名 + 同 URL」的创建冲突检测（PRD §4.3 / §6.3.2 相关语义）。v3 曾尝试同批 parent检测，v4 删除时需在 **应用层另一路径** 或 **Detect 扩展** 覆盖 PRD，否则功能缺口。

---

### P1.6 文档内部一致性与可点击引用

**位置:** §10 Phase 1 第 4 天写「同步 API（/sync）」—与 v4 正式路径 **`/api/v1/bookmarks/sync`** 不一致。§12 文末仍写「见 `./tech/` 目录下 4 份 v3 评审报告」— 本文件已在 **`tech/`**，应写 **同目录** 或具体文件名，避免相对路径歧义。

---

## P2 — 建议优化

### P2.1 「SQLite 不支持 FOR UPDATE」表述过绝对

**位置:** v4 修订摘要 P0-2；§6.2

是否支持取决于 SQLite 版本与语句形态；v4 选择 **单条 `UPDATE ... RETURNING`** 合理。建议改为「本方案不依赖 `SELECT … FOR UPDATE`，统一用 `RETURNING`（SQLite 3.35+）」。

---

### P2.2 `NormalizeWeights` 伪代码中 `FromVersion: 0` 与 `change.Data` 被二次赋值

**位置:** §6.3（约 L799–L814）

`FromVersion` 宜与 `ToVersion` 及 change_log 语义一致（通常为 `newVersion-1` /或由同一事务内状态推导）；`change.Data = ChangeData{}` 覆盖前文占位，易误导复制粘贴实现。

---

### P2.3截断删除条件 `to_version <= cutoffVersion` 与 `cutoffVersion = from_version - 1`

**位置:** §6.4

数学说明已比 v3 清晰，仍建议在实现阶段用 **小规模序列做一次表驱动推演**（含 reindex/rollback），写入单元测试注释，防止 off-by-one 与 baseline 不一致。

---

### P2.4 `VersionGaps` 告警与 `rate()` 语义

**位置:** §9.2，`rate(lzc_version_gaps_total[1h]) > 10`

`rate` 常用于 Counter的每秒速率；「1 小时内超过 10 个空洞」若意指 **计数增量**，更常见写法是 `increase(lzc_version_gaps_total[1h]) > 10`。实现落地时由可观测性负责人定稿。

---

### P2.5文首「P2 已优化」第 8、9、10 条与正文对应关系

第 8 条（分页）、第 9 条（change_log.data 压缩）、第 10 条（Dockerfile USER）— **正文或未展开或与 PRD 已有「快照压缩」并存时需写清边界**，避免评审误以为已全部写进 v4 正文。

---

## 积极方面（简要）

- **工程细节:** `busy_timeout`、Docker 最小上下文、非 root 用户、集成测试 `docker compose` + `--frozen-lockfile`、健康检查查库、冲突超时配置项等，有利于首轮可部署与可运维。  
- **协议可读性:** 「响应不变式」表格方向正确，修补后更可成为实现检查表。  
- **冲突与类型:** `ChangeData` + `serializer:json` 比泛型 JSON 更利于 Go 侧维护。  
- **参考链接:** `../prd/PRD-v6.md` 与同目录 v3 评审链接整体正确。

---

## 总结与建议

| 维度 | 评价 |
|------|------|
| 相对 v3 | 明显进步，偏实现与运维的细节更完整 |
| 相对 PRD v6 | **错误码与全量同步（POST）语义未对齐**，属阻断级文档缺陷 |
| 可开发性 | 在 **冻结契约**（PRD 或 v4 修订版二选一）后，可进入 Phase 1 |

**综合建议:**

1. **立即** 将 §4.3 / §4.4 / §4.5 与 PRD §6.1、§6.3.2 **逐条 diff**，统一为单一真相源（优先 PRD，或升版 PRD）。  
2. 补齐 **GET `/bookmarks` 分页**与 **`generated_mappings` / `synced_at`** 等与 PRD 一致。  
3. 明确 **纯 create 批次**下的创建冲突检测责任归属。  
4. 修正 §10 路径表述与 §12 文末「`./tech/`」措辞。

完成上述项后，v4 可重新标注为「与 PRD v6.x 对齐」并作为开发基线。

---

**审计状态:** 已完成（Composer2）  
**输出文件:** `tech/tech-design-v4-reviewed-by-composer2.md`（与本审计对象同目录）
