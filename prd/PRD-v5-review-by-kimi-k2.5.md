# 懒猫书签同步系统 PRD v5.0 - 审计报告

**审计者:** kimi-k2.5  
**审计时间:** 2026-04-16  
**PRD 版本:** v5.0  

---

## 总体评价

PRD v5.0 相比 v4.0 有**重要补充和修正**，核心机制更加严谨和完整。主要改进包括：
- ✅ 修正变更日志截断逻辑（OR/AND 统一为"满足任一即触发，保留最近 1000 条且 ≤ 7 天"）
- ✅ 补充同批 create 的 GUID 分配协议（parent_local_id 机制）
- ✅ 明确 Chrome 三根映射策略（root_bar/root_other/root_mobile ↔ id="1"/"2"/"3"）
- ✅ 完善冲突检测时间窗口语义（仅用于 UI 提示，不影响判定）
- ✅ 补充 reindex 变更格式（children_weights 详细结构）
- ✅ 增加 limit 分页参数（默认 1000，最大 1000）
- ✅ 明确只读 API Key 权限矩阵（完整接口权限表）
- ✅ 补充物理删除条件（最近 30 天快照未引用）
- ✅ 明确离线队列合并策略（仅合从未提交过的变更）
- ✅ 补充 409 E003 vs 200 + conflicts 边界

文档已达到**生产级 PRD** 标准，细节充分，可直接指导开发。

---

## 详细审计意见

### 1. 产品概述 ✅

无变更，保持 v4 质量。

---

### 2. 用户场景 ✅

无变更。

---

### 3. 系统架构 ✅

无重大变更。

---

### 4. 核心设计决策 ✅

#### 4.1 书签身份模型 ✅

**同批 create 的 GUID 分配协议（v5 新增）:**
```typescript
PendingChange {
  parent_local_id: string | null  // create 时父节点的 local_id（用于同批创建）
}
```

**协议清晰:**
1. 客户端按拓扑顺序排序变更（父节点在前，子节点在后）
2. 子节点的 `parent_guid` 填写服务端规范树中的 GUID（如果父节点已存在）
3. 如果父节点也是本次上传的新节点：
   - 使用 `parent_local_id` 字段指向父节点的 local_id
   - 服务端按顺序处理，遇到父节点时生成 GUID 并缓存
   - 处理子节点时，从缓存中查找父节点的 GUID

**示例清晰:**
```json
[
  { "type": "create", "local_id": "100", "guid": null, "data": { "title": "文件夹 A" } },
  { "type": "create", "local_id": "101", "parent_local_id": "100", "data": { "title": "书签 1" } }
]
```

**Chrome 三根映射策略（v5 新增）:**

**Chrome 原生根节点:**
| Chrome ID | 名称（中文） | 名称（英文） |
|-----------|-------------|-------------|
| 0 | 根节点 | Root |
| 1 | 书签栏 | Bookmarks Bar |
| 2 | 其他书签 | Other Bookmarks |
| 3 | 移动设备书签 | Mobile Bookmarks |

**映射规则:**
```
服务端 "root_bar" → Chrome id="1"（书签栏）
服务端 "root_other" → Chrome id="2"（其他书签）
服务端 "root_mobile" → Chrome id="3"（移动设备书签）
```

**首次上传流程明确:**
1. 提取 id="1"、id="2"、id="3" 三个子树
2. 为每个子树生成对应的规范树节点（root_bar、root_other、root_mobile）
3. 按拓扑顺序上传

**新设备拉取流程明确:**
1. 解析规范树第一层，映射到 Chrome 固定根
2. 从第二层开始调用 `chrome.bookmarks.create()`
3. 不创建第一层（Chrome 已有固定根）

**设计合理:** 避免创建 Chrome 系统根节点，直接复用现有根。

#### 4.2 增量同步模型 ✅

**变更日志截断策略修正（v5）:**
```
v4: 保留条件：COUNT <= 1000 OR age <= 7 days（满足任一即保留）
v5: 截断条件：条数 > 1000 OR 最旧记录年龄 > 7 天（满足任一即触发截断）
     保留结果：最近 1000 条 且 年龄 ≤ 7 天 的记录
```

**示例清晰:**
- 场景 A：7 天内产生了 2000 条变更 → 截断到最近 1000 条
- 场景 B：100 条变更但最旧的已 10 天 → 截断到 7 天内的记录
- 场景 C：500 条变更且都在 5 天内 → 不截断

**逻辑统一:** 触发条件和保留结果使用相同的条件（OR），避免歧义。

#### 4.3 冲突检测与解决 ✅

**冲突检测时间窗口明确（v5）:**
```
冲突条件（满足全部即判定为冲突）：
1. 同一 GUID（或创建冲突的同 parent_guid + 同名 + 同 URL）
2. 版本重叠：两个变更的 from_version 相同
3. 时间窗口：服务端接收时间相差 < 10 秒（仅用于提示"并发修改"）

注意：时间窗口仅用于 UI 提示，不影响冲突判定。
即使时间相差 > 10 秒，只要 from_version 相同，仍判定为冲突。
```

**设计合理:** 时间窗口仅用于 UI 提示"并发修改"，不影响实际的冲突判定逻辑。

**部分冲突处理明确（v5）:**
```
服务端响应：
{
  "success": true,
  "data": {
    "new_version": 128,      // 8 条变更应用，版本号 +8
    "applied_changes": [...], // 8 条已应用
    "conflicts": [...]        // 2 条挂起
  },
  "version": 130  // 全局最新版本（可能包含其他设备的变更）
}
```

**客户端行为明确:**
1. 部分冲突：更新 `last_known_version = new_version`（128）
2. 全部冲突：不更新 `last_known_version`，但记录 `version` 字段（130）用于下次拉取
3. 冲突变更的 `base_version` 更新为 `new_version`

#### 4.4 排序权重设计 ✅

**reindex 变更格式补充（v5）:**
```json
{
  "change_id": "ch_reindex_001",
  "type": "reindex",
  "guid": "folder-guid",  // 被归一化的文件夹 GUID
  "device_id": "server",
  "data": {
    "children_weights": {
      "guid-1": 1.0,
      "guid-2": 2.0,
      "guid-3": 3.0
    }
  },
  "from_version": 199,
  "to_version": 200,
  "timestamp": 1713254400000
}
```

**客户端行为明确:**
- 应用权重更新（不触发 UI 刷新）
- 不计入变更日志 1000 条上限判断
- 但仍写入 change_log 表（客户端可拉取）

---

### 5. 功能需求 ✅

#### 5.1 懒猫微服 ✅

**只读 API Key 权限矩阵（v5 新增）:**
| 接口 | 只读 | 读写 |
|------|------|------|
| GET /bookmarks | ✅ 允许 | ✅ 允许 |
| POST /sync | ❌ 禁止 (E002) | ✅ 允许 |
| POST /conflicts/resolve | ❌ 禁止 (E002) | ✅ 允许 |
| POST /rollback | ❌ 禁止 (E002) | ✅ 允许 |
| GET /devices | ✅ 允许 | ✅ 允许 |
| POST /devices | ✅ 允许 | ✅ 允许 |
| DELETE /devices | ❌ 禁止 (E002) | ✅ 允许 |
| GET /history | ✅ 允许 | ✅ 允许 |

**物理删除条件明确（v5）:**
- `is_deleted=true` AND `deleted_at > 90 days` AND **最近 30 天快照未引用**

#### 5.2 Chrome 浏览器插件 ✅

**离线队列合并策略明确（v5）:**
- **幂等保护：仅合并从未提交过的变更（无 client_request_id 记录）**

**设计合理:** 避免已提交变更被错误合并导致数据丢失。

---

### 6. API 设计 ✅

#### 6.1 通用规范 ✅

**409 E003 vs 200 + conflicts 边界（v5 新增）:**
```
返回 409 E003 的场景：
- 客户端 base_version 落后超过阈值（如 1000 个版本）
- 客户端 base_version 早于 baseline_version（变更日志已截断）
- 其他需要强制客户端全量重拉的场景

返回 200 + conflicts 的场景：
- 正常的业务冲突（同一 GUID 的并发修改）
- 部分变更可应用，部分变更有冲突
```

**设计合理:** 区分"需要全量重拉的技术错误"和"正常的业务冲突"。

#### 6.2 认证接口 ✅

无变更。

#### 6.3 书签接口 ✅

**GET /api/v1/bookmarks 分页（v5 新增）:**
```
Query: since_version=<integer>&limit=1000
- limit 可选，默认 1000，最大 1000
```

**分页行为:**
```
请求：GET /bookmarks?since_version=120&limit=100
响应：
- 如果变更数 <= 100：has_more=false，返回所有变更
- 如果变更数 > 100：has_more=true，返回前 100 条

客户端续页：
- 使用最后一条变更的 to_version 作为下一次请求的 since_version
- GET /bookmarks?since_version=220&limit=100
```

**设计合理:** 避免单次响应过大，支持增量拉取大数据量。

**POST /api/v1/bookmarks/sync ✅**

**同批 create 协议明确（v5）:**
```
协议：
1. 客户端按拓扑顺序排序变更（父节点在前，子节点在后）
2. 子节点的 parent_guid 字段填写服务端规范树中的 GUID（如果父节点已存在）
3. 如果父节点也是本次上传的新节点：
   - 使用 parent_local_id 字段指向父节点的 local_id
   - 服务端按顺序处理，遇到父节点时生成 GUID 并缓存
   - 处理子节点时，从缓存中查找父节点的 GUID
```

**分批提交策略（v5 新增）:**
```
- 第一批：base_version=0，client_request_id="batch_001"
- 第二批：base_version=第一批返回的 new_version，client_request_id="batch_002"
- 如果某批失败：重试该批；如果最终部分成功，基于当前状态继续同步
- 书签是覆盖语义，不会丢失数据
```

**客户端状态更新规则明确（v5）:**
```
收到部分冲突响应：
1. 更新 last_known_version = new_version（与已应用的变更对齐）
2. 从离线队列移除 applied_changes 中的变更
3. 保留冲突变更，标记为"待解决"
4. 冲突变更的 base_version 更新为 new_version
5. 下次推送时，仅推送冲突变更 + 新增变更

收到全部冲突响应（applied_changes 为空）：
1. 不更新 last_known_version
2. 所有变更保留在离线队列，标记为"待解决"
3. 下次推送时，重新推送所有变更（base_version 不变）

收到 409 E003 响应：
1. 不更新 last_known_version
2. 触发全量重拉（GET /bookmarks 无 since_version）
```

#### 6.4 设备接口 ✅

无变更。

#### 6.5 历史接口 ✅

**回滚原子性明确（v5）:**
```
3. 生成新的变更记录（类型为 "rollback"）
   - data 字段包含：{ "rollback_to_version": 100, "affected_guids": [...] }
```

#### 6.6 限流策略 ✅

无变更。

#### 6.7 数据上限 ✅

**新增上限:**
| 限制项 | 上限 | 超限行为 |
|--------|------|----------|
| GET /bookmarks 响应大小 | 10 MB | 返回 E007，建议全量同步 |

---

### 7. 数据模型 ✅

**特殊变更类型明确（v5）:**
- `reindex`: index_weight 归一化生成
  - `guid`: 被归一化的文件夹 GUID
  - `data`: `{ "children_weights": { "guid-1": 1.0, "guid-2": 2.0, ... } }`
  - 不计入变更日志 1000 条上限判断
  - 但仍写入 change_log 表，客户端可拉取应用
- `rollback`: 回滚操作生成
  - `data`: `{ "rollback_to_version": 100, "affected_guids": ["guid1", "guid2", ...] }`
  - 包含完整树差异（通过快照 + 差分）

---

### 8. 同步逻辑 ✅

**首次同步流程补充（v5）:**
```
分批提交策略：
- 第一批：base_version=0，client_request_id="batch_001"
- 第二批：base_version=第一批返回的 new_version，client_request_id="batch_002"
- 如果某批失败：重试该批；如果最终部分成功，基于当前状态继续同步
- 书签是覆盖语义，不会丢失数据
```

**新设备首次同步流程补充（v5）:**
```
4. 解析规范树第一层：
   - guid="root_bar" → 映射到 Chrome id="1"（书签栏）
   - guid="root_other" → 映射到 Chrome id="2"（其他书签）
   - guid="root_mobile" → 映射到 Chrome id="3"（移动设备书签）
```

**BFS 创建顺序示例:**
```
Level 0: root（服务端虚拟根，不创建）
Level 1: root_bar、root_other、root_mobile（映射到 Chrome 固定根，不创建）
Level 2: 书签栏下的文件夹 A、文件夹 B → 调用 create({parentId: "1"})
Level 3: 文件夹 A 下的书签 1、书签 2 → 调用 create({parentId: local_id_of_A})
```

**错误恢复场景补充（v5）:**
| 场景 | 行为 |
|------|------|
| 收到 409 E003 | 触发全量重拉（GET /bookmarks 无 since_version） |

---

### 9. 安全考虑 ✅

无重大变更。

---

### 10. 开发计划 ✅

无变更。

---

### 11. 风险与依赖 ✅

无变更。

---

### 12. 验收标准 ✅

**功能验收新增:**
- [ ] Chrome 三根映射正确（书签栏、其他书签、移动设备）

**性能验收新增:**
| 指标 | 目标 | 测试条件 |
|------|------|----------|
| 弱网同步（10 条变更） | < 5 秒 | 延迟 200ms，丢包 5% |

**压力验收（可选，不阻塞 MVP 发布）:**
- [ ] 10 设备全量拉取 P95 < 60 秒
- [ ] 单 API Key 下 20 设备同时同步，无数据丢失
- [ ] 变更日志截断后，落后设备全量重拉正常

---

### 13. 附录 ✅

**术语表新增:**
- Chrome 三根

