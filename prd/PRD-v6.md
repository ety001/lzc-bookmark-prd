# 懒猫书签同步系统 - PRD v6.0

**版本变更：** v5.0 → v6.0  
**变更摘要：** 修复 410 触发矛盾（统一为直接返回 200 全量）、修正冲突检测规则（支持部分冲突重试）、统一 Snapshot 树类型、明确 parent_guid 优先级、补充 rollback 应用机制、简化 Service Worker 降级策略  
**评审来源：** Claude、Composer 2、GLM-5、GLM-5.1、Kimi Code、Kimi-k2.5 六方审计汇总

---

## 1. 产品概述

### 1.1 产品名称
懒猫书签 (Lazycat Bookmark)

### 1.2 产品定位
基于懒猫微服架构的 Chrome 浏览器书签同步系统，实现多设备间书签的自动备份与同步，类似 Dropbox 的同步盘体验，但专注于浏览器书签。

### 1.3 核心价值
- 跨设备同步：多个 Chrome 浏览器安装插件后，书签自动同步
- 版本历史：保留书签变更历史，支持回滚
- 私有部署：数据存储在用户自己的懒猫微服，不经过第三方
- 增量同步：高效同步，只传输变更数据

### 1.4 目标用户
- 个人用户：拥有 2-5 台设备的开发人员、知识工作者
- 团队用户（未来）：小型团队共享书签集合（v2 规划，数据模型预留 tenant_id）

### 1.5 浏览器兼容性
| 浏览器 | 最低版本 | 支持级别 |
|--------|----------|----------|
| Chrome | 102+ | 完全支持 |
| Edge | 102+ | 完全支持 |
| Brave | 1.40+ | 完全支持 |
| 其他 Chromium | 102+ | 基本支持（未全面测试） |

---

## 2. 用户场景

### 2.1 典型用户
- 拥有多台电脑的开发人员
- 需要在工作和家庭电脑间同步书签的用户
- 担心浏览器账号同步隐私问题的用户

### 2.2 使用场景

| 场景 | 描述 |
|------|------|
| 首次设置 | 用户在电脑 A 安装插件 → 配置懒猫 API → 全量备份书签 |
| 日常使用 | 用户在电脑 A 添加书签 → 插件自动备份到懒猫 |
| 多设备同步 | 用户在电脑 B 安装插件 → 配置相同 API → 拉取电脑 A 的书签 |
| 冲突处理 | 电脑 A 和 B 同时修改 → 懒猫检测冲突 → 用户选择解决方案 |
| 恢复历史 | 误删书签 → 查看历史版本 → 回滚到之前状态 |
| 设备管理 | 用户更换电脑 → 撤销旧设备访问 → 注册新设备 |
| 长期离线后同步 | 用户出差一周后打开电脑 → 插件自动全量重拉（变更日志已溢出） |
| 部分冲突处理 | 电脑 A 推送 10 条变更，其中 2 条冲突 → 8 条正常应用，2 条待解决 |

---

## 3. 系统架构

### 3.1 架构图

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│  Chrome 插件 A   │         │  懒猫微服 (LPK)  │         │  Chrome 插件 B   │
│  (电脑 1)        │         │  lzc-bookmark   │         │  (电脑 2)        │
│                 │         │                 │         │                 │
│ - 书签读取       │   HTTP  │ - Go API 服务   │  HTTP   │ - 书签读取       │
│ - 书签写入       │────────►│ - React 管理界面│◄────────│ - 书签写入       │
│ - 冲突检测       │         │ - SQLite 存储   │         │ - 冲突检测       │
│ - 本地缓存       │         │ - 版本历史      │         │ - 本地缓存       │
│ - 离线队列       │         │ - 冲突记录      │         │ - 离线队列       │
│ - GUID 映射表    │         │ - GUID 规范树   │         │ - GUID 映射表    │
└─────────────────┘         └─────────────────┘         └─────────────────┘
```

**关键说明：**
- v1 版本仅使用 HTTP REST API，删除 WebSocket 以简化实现
- 实时性通过插件轮询（默认 30 秒间隔）保证
- **服务端仅存储 GUID 规范树，不存储 Chrome local_id**
- **local_id ↔ GUID 映射仅存在于各插件本地 storage.local**

### 3.2 组件划分

| 组件 | 技术栈 | 职责 |
|------|--------|------|
| 懒猫微服 | LPK (Go + React) | API 服务、数据存储、冲突处理、Web 管理界面 |
| Chrome 插件 | Manifest V3 | 书签读写、同步触发、UI 展示、GUID 映射管理 |

### 3.3 技术约束

| 约束项 | 要求 | 理由 |
|--------|------|------|
| 懒猫框架 | 必须使用 LPK (Go+React) | 与懒猫微服生态统一 |
| 插件标准 | 必须使用 Manifest V3 | Chrome 官方要求 |
| 通信协议 | HTTPS（默认）/ HTTP（内网可选，需显式配置 `ALLOW_INSECURE_HTTP=true`） | 安全传输，内网测试例外 |
| 数据存储 | SQLite（默认）/ PostgreSQL（可选，通过 `DB_TYPE=postgres` 环境变量切换） | 轻量部署 + 扩展性 |
| 部署方式 | Docker 容器 | 懒猫微服标准 |
| 数据模型 | 预留 `tenant_id` 字段（当前阶段恒为空字符串，团队功能启用） | 未来扩展性 |

### 3.4 数据流向（明确顺序：先推后拉）

```
1. 插件启动
       │
       ▼
2. 检查本地未同步变更
       │
       ├── 有变更 ──► 加入离线队列
       │
       ▼
3. 网络可用？─── 否 ──► 等待下次同步（队列持久化）
       │ 是
       ▼
4. POST /api/v1/bookmarks/sync（推送本地变更）
       │
       ├── 返回 conflicts（部分应用）──► 无冲突变更已应用，冲突变更挂起
       │                                  客户端更新 last_known_version = new_version
       │                                  下次推送仅推送冲突变更 + 新变更
       │
       ├── 返回 conflicts（全部冲突）──► 无变更应用，客户端不更新 last_known_version
       │                                  但 version 字段返回全局最新版本
       │
       ▼
5. GET /api/v1/bookmarks?since_version=last_known_version（拉取远程变更）
       │
       ├── 返回 410 GONE ──► 全量重拉
       │
       ▼
6. 应用远程变更到本地书签树
       │
       ▼
7. 更新本地 last_known_version = version from response
       │
       ▼
8. 同步完成
```

**设计决策：先推后拉**
- 避免拉取后到提交前产生新的竞争窗口
- 推送后服务端版本号已更新，拉取的是最新状态

---

## 4. 核心设计决策

### 4.1 书签身份模型（关键）

**问题：** Chrome 在不同设备上为同一书签分配不同的本地 ID，无法直接用于跨设备同步。

**解决方案：双层 ID 映射 + 服务端规范树**

```typescript
// 服务端存储（规范树，不含 local_id）
Bookmark {
  guid: string           // 全局唯一 ID (UUID v4)，跨设备识别同一书签
  tenant_id: string      // 预留，当前阶段恒为空字符串
  title: string
  url: string | null
  parent_guid: string    // 父节点的 guid（"root" 表示根目录，特殊常量）
  index_weight: float64  // 排序权重（避免 index 冲突，见 §4.4）
  date_created: integer  // 服务端时间戳（Unix 毫秒）
  date_modified: integer // 服务端时间戳
  is_deleted: boolean    // 软删除标记
  deleted_at: integer    // 删除时间（可选，用于物理删除倒计时）
}

// 插件本地映射表（Chrome storage.local，不上传）
IdMapping {
  local_id: string       // Chrome 本地 ID（仅在该设备有效）
  guid: string           // 服务端 GUID
  last_synced_version: integer  // 最后同步的版本号
}

// 插件本地变更队列（Chrome storage.local）
PendingChange {
  change_id: string      // 变更 ID（UUID）
  type: "create" | "update" | "delete" | "move"
  guid: string | null    // create 时为 null（服务端生成 GUID）
  local_id: string       // Chrome 本地 ID
  parent_local_id: string | null  // create 时父节点的 local_id（用于同批创建）
  data: { ... }          // 变更数据
  created_at: integer    // 本地时间戳
  retry_count: integer   // 重试次数
  submitted: boolean     // 是否已提交过（用于队列合并判断）
}
```

**工作流程：**

1. **首次同步（全量上传）**：
   - 插件上传完整书签树（含 local_id）
   - 服务端为每个节点生成 GUID，返回 `generated_mappings: [{local_id, guid}]`
   - 插件存储映射表到 storage.local
   - 服务端存储规范树（不含 local_id）
   - **上传顺序：父节点先于子节点（拓扑有序）**
   - **同批 create 的父引用：使用 parent_local_id，服务端解析为正式 GUID**
   - **三根节点预填固定 GUID：root_bar / root_other / root_mobile**

2. **后续同步（增量）**：
   - 插件使用 GUID 标识书签（通过本地映射表查找）
   - 提交变更时携带 `base_version`
   - 服务端基于 GUID 检测冲突

3. **新设备首次同步（全量拉取）**：
   - 插件拉取服务端规范树
   - **按 BFS 顺序调用 chrome.bookmarks.create() 创建书签**
   - **Chrome 三根映射：规范树 root 的子节点映射到 Chrome 固定根**
   - Chrome 返回每个节点的 local_id
   - 插件建立本地映射表：`{local_id ↔ guid}`
   - 后续增量同步使用 GUID

### 4.2 增量同步模型

**设计选择：版本号 + 变更日志（Change Log）**

**版本号语义（关键）：**
- **每条变更（Change）对应全局版本号 +1**
- 一次 sync 请求提交 N 条变更，全局版本号 +N
- `change_log` 存储 N 条独立记录

```typescript
// 服务端维护
GlobalState {
  current_version: integer    // 全局版本号，每条变更 +1
  change_log: Change[]        // 变更记录（截断条件见下方）
  snapshots: Snapshot[]       // 版本快照（保留最近 500 个或最近 30 天）
}

// 变更记录
Change {
  change_id: string           // 变更 ID（UUID，用于幂等）
  type: "create" | "update" | "delete" | "move" | "reindex" | "rollback"
  guid: string                // 书签 GUID
  device_id: string           // 来源设备
  data: {                     // 变更数据（delete 时为空）
    title?: string
    url?: string
    parent_guid?: string
    index_weight?: float64
  } | null
  from_version: integer       // 变更前的版本号（= current_version before this change）
  to_version: integer         // 变更后的版本号（= from_version + 1）
  timestamp: integer          // 服务端接收时间戳（Unix 毫秒）
}

// 版本快照（用于全量同步和回滚）
Snapshot {
  version: integer            // 快照对应的版本号
  bookmark_tree: BookmarkNode // 完整规范树（嵌套树结构，非扁平数组）
  created_at: integer
  is_baseline: boolean        // 是否为基准快照（用于变更日志溢出时的全量参考）
}

// 嵌套树结构（v6 统一）
BookmarkNode {
  guid: string
  title: string
  url: string | null
  parent_guid: string
  index_weight: float64
  date_created: integer
  date_modified: integer
  children: BookmarkNode[]  // 子节点数组（递归）
}
```

**变更日志截断策略（v6 明确）：**
```
截断条件：条数 > 1000 OR 最旧记录年龄 > 7 天（满足任一即触发截断）
保留结果：最近 1000 条 且 年龄 ≤ 7 天 的记录

计数规则：
- 仅统计 type 为 create/update/delete/move 的变更
- reindex 和 rollback 不计入 1000 条上限，但仍受 7 天年龄限制

示例：
- 场景 A：7 天内产生了 2000 条变更 → 截断到最近 1000 条
- 场景 B：100 条变更但最旧的已 10 天 → 截断到 7 天内的记录
- 场景 C：500 条变更且都在 5 天内 → 不截断
- 场景 D：800 条普通变更 + 300 条 reindex，最旧 5 天 → 不截断（普通变更 < 1000）

截断时操作：
1. 删除旧记录
2. 更新基准快照：is_baseline=true 的 snapshot.version 更新为截断点的最小 from_version
3. 新设备或落后设备（last_known_version < baseline_version）走全量同步
```

**同步流程：**
1. 插件携带 `last_known_version` 请求 `GET /api/v1/bookmarks?since_version=last_known_version`
2. 服务端返回 `change_log` 中 `to_version > last_known_version` 的所有变更
3. 插件应用变更，更新本地映射表
4. 插件提交本地变更（包含 `changes` 数组）
5. 服务端检测冲突，无冲突则追加到 `change_log`，每条变更版本号 +1

**增量拉取过滤谓词（形式化）：**
```
客户端已知版本：last_known_version = N
服务端返回：所有满足 to_version > N 的变更
排序：按 to_version 升序

示例：
last_known_version = 120
变更日志：
  - Change A: from_version=119, to_version=120  ← 不返回（to_version 不大于 120）
  - Change B: from_version=120, to_version=121  ← 返回
  - Change C: from_version=121, to_version=122  ← 返回
  - Change D: from_version=122, to_version=123  ← 返回
```

**变更日志溢出策略（v6 修正）：**
```
溢出条件：last_known_version < baseline_version
溢出行为：
  - 服务端直接返回 HTTP 200 + 完整规范树（is_baseline=true）
  - 不再返回 410 GONE（v6 变更，简化流程）
  - 客户端收到 is_baseline=true 响应后，清空本地映射表，重建全量树

注意：410 GONE 仅用于服务端主动拒绝服务（如维护模式），正常溢出场景用 200 全量响应。
```

### 4.3 冲突检测与解决

**冲突类型：**
| 类型 | 检测条件 | 解决策略 |
|------|----------|----------|
| 更新冲突 | 同一 GUID，该变更基于的版本 < 该 GUID 最近被修改的版本 | 用户选择 / 最新优先 |
| 删除冲突 | 设备 A 删除，设备 B 修改同一 GUID | 默认进入冲突流程，推荐保留修改 |
| 移动冲突 | 同一 GUID，两个变更的 `parent_guid` 不同 | 用户选择 |
| 重命名冲突 | 同一 GUID，两个变更的 `title` 不同 | 最新优先（服务端接收时间） |
| 创建冲突 | 同一 `parent_guid` + 同名 + 同 URL 的书签（无 GUID） | **仅同文件夹内合并** |

**冲突检测规则（v6 修正）：**
```
冲突条件（满足全部即判定为冲突）：
1. 同一 GUID（或创建冲突的同 parent_guid + 同名 + 同 URL）
2. 该变更的 base_version < 该 GUID 最近一次被修改的 to_version

示例：
- 设备 A 推送变更：guid="X", base_version=120
- 服务端检查：guid="X" 最近被修改的 to_version=125
- 判定：120 < 125 → 冲突

部分冲突重试场景：
- 设备 A 推送 10 条变更，8 条应用（version 120→128），2 条冲突
- 设备 A 更新冲突变更的 base_version=128，重新推送
- 服务端检查：guid="Y" 最近被修改的 to_version=128（就是刚才那 8 条之一）
- 判定：128 < 128 为假 → 不冲突，正常应用
```

**注意：** 时间窗口（< 10 秒）仅用于 UI 提示"并发修改"，不影响冲突判定。

**冲突解决流程：**
```
服务端检测冲突 → 返回 conflicts 数组（HTTP 200，非错误）
         ↓
插件标记为"待解决"状态，用户可见但未应用
         ↓
用户选择解决方案（插件 UI）
         ↓
POST /api/v1/bookmarks/conflicts/{id}/resolve
         ↓
服务端应用解决方案，生成新版本
         ↓
插件拉取新版本，应用解决结果
```

**部分冲突处理（v6 明确）：**

场景：客户端推送 10 条变更，其中 8 条无冲突，2 条有冲突。

```
服务端响应：
{
  "success": true,
  "data": {
    "new_version": 128,           // 8 条变更应用，版本号 +8
    "applied_changes": ["ch_001", ..., "ch_008"],
    "conflicts": [
      {"conflict_id": "cf_001", ...},
      {"conflict_id": "cf_002", ...}
    ]
  },
  "version": 130  // 全局最新版本（可能包含其他设备的变更）
}

客户端行为：
1. 部分冲突：更新 last_known_version = new_version（128）
2. 全部冲突：不更新 last_known_version，但记录 version 字段（130）用于下次拉取
3. 从离线队列移除已应用的 8 条变更
4. 保留 2 条冲突变更，标记为"待解决"，更新 base_version = new_version（128）
5. 下次推送时，仅推送冲突变更 + 新增变更（不重复推送已应用变更）

幂等重试：
- 如果客户端重试相同的 client_request_id，服务端返回相同的 conflicts 数组
- conflict_id 不变，客户端可据此重建"待解决"状态
```

**设计决策：冲突非阻塞**
- 无冲突变更正常应用，版本号推进
- 只有冲突变更挂起等待用户解决
- 客户端可以继续推送新的非冲突变更

### 4.4 排序权重设计（避免 index 冲突）

**问题：** `index` 表示书签在父文件夹中的排序位置，但不同设备可能对同一文件夹下的书签有不同的排序偏好。

**解决方案：使用 float64 权重代替绝对 index**

```typescript
// 服务端
Bookmark {
  parent_guid: string
  index_weight: float64  // 双精度浮点数
}
```

**插入算法：**
```
在节点 A(weight=1.0) 和节点 B(weight=2.0) 之间插入新节点：
  新节点 weight = (1.0 + 2.0) / 2 = 1.5

在第一个节点前插入：
  新节点 weight = first.weight - 1.0

在最后一个节点后插入：
  新节点 weight = last.weight + 1.0

权重空间耗尽时（最小间隔 < 1e-10）：
  触发归一化，重新分配权重为 1.0, 2.0, 3.0...
```

**归一化策略（v6 明确）：**
- 触发条件：同一 `parent_guid` 下，相邻节点的最小 `index_weight` 间隔 < 1e-10
- 归一化行为：
  - 重新分配权重为 1.0, 2.0, 3.0...（间隔 1.0）
  - **生成 `reindex` 类型变更记录**（特殊类型，不计入变更日志上限）
  - 影响范围：仅当前文件夹下的节点
  - 节流：同一文件夹 5 分钟内最多触发一次归一化
- `reindex` 变更格式：
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
- 客户端收到 `reindex` 变更：
  - 应用权重更新（不触发 UI 刷新）
  - 不计入变更日志 1000 条上限判断
  - 但仍写入 change_log 表，客户端可拉取应用

**优点：**
- 避免多设备同时移动书签导致的 index 冲突
- 客户端可本地维护显示 index（按 weight 排序后生成）
- 归一化是低频操作（正常用户几乎不会触发）

### 4.5 Chrome 三根映射策略

**问题：** Chrome 有 3 个固定的系统根节点，服务端规范树只有一个 `root`。

**Chrome 原生根节点：**
| Chrome ID | 名称（中文） | 名称（英文） | 说明 |
|-----------|-------------|-------------|------|
| 0 | 根节点 | Root | 虚拟根，不可直接操作 |
| 1 | 书签栏 | Bookmarks Bar | 浏览器书签栏 |
| 2 | 其他书签 | Other Bookmarks | 其他书签文件夹 |
| 3 | 移动设备书签 | Mobile Bookmarks | 移动设备同步书签（可选） |

**映射规则：**
```
服务端规范树结构：
root (guid="root")
├── bookmarks_bar (guid 固定为 "root_bar")
│   └── 用户自定义文件夹和书签
├── other (guid 固定为 "root_other")
│   └── 用户自定义文件夹和书签
└── mobile (guid 固定为 "root_mobile")
    └── 用户自定义文件夹和书签

映射关系：
- 服务端 "root_bar" → Chrome id="1"（书签栏）
- 服务端 "root_other" → Chrome id="2"（其他书签）
- 服务端 "root_mobile" → Chrome id="3"（移动设备书签）

注意：第一层节点的 title 字段仅供参考展示，不参与任何逻辑匹配。
```

**首次上传流程：**
```
1. 插件读取 Chrome 完整书签树（chrome.bookmarks.getTree()）
2. 提取 id="1"、id="2"、id="3" 三个子树
3. 为每个子树生成对应的规范树节点：
   - bookmarks_bar (guid="root_bar", title 从 Chrome 读取)
   - other (guid="root_other", title 从 Chrome 读取)
   - mobile (guid="root_mobile", title 从 Chrome 读取)
4. 按拓扑顺序上传（父节点先于子节点）
   - 三根节点的 change.guid 预填固定值（非 null），服务端直接使用
5. 服务端返回 generated_mappings
6. 插件存储映射表
```

**新设备拉取流程：**
```
1. 插件 GET /bookmarks 拉取完整规范树
2. 解析规范树第一层：
   - guid="root_bar" → 映射到 Chrome id="1"（书签栏）
   - guid="root_other" → 映射到 Chrome id="2"（其他书签）
   - guid="root_mobile" → 映射到 Chrome id="3"（移动设备书签）
3. 按 BFS 顺序创建书签（从第二层开始）：
   a. 遍历规范树第二层节点
   b. 对每个节点调用 chrome.bookmarks.create({parentId: "1" 或 "2" 或 "3", ...})
   c. Chrome 返回新节点的 local_id
   d. 建立映射：{local_id ↔ guid}
   e. 递归处理子节点（BFS 顺序）
4. 存储映射表到 storage.local

注意：如果 Chrome 不存在 id="3"（某些环境禁用 Mobile 根），跳过 root_mobile 子树。
```

**设计决策：**
- 服务端规范树第一层固定为 3 个系统根（bookmarks_bar、other、mobile）
- 用户自定义文件夹只能作为第二层及以下的节点
- 新设备拉取时不创建第一层，直接映射到 Chrome 现有根
- 如果规范树中某个根为空（如 mobile），客户端跳过该根

### 4.6 回滚操作机制（v6 新增）

**回滚变更格式：**
```json
{
  "change_id": "ch_rollback_001",
  "type": "rollback",
  "guid": "root",
  "device_id": "server",
  "data": {
    "rollback_to_version": 100,
    "snapshot_tree": { ... }  // 目标版本的完整规范树（嵌套结构）
  },
  "from_version": 199,
  "to_version": 200,
  "timestamp": 1713254400000
}
```

**回滚原子性：**
```
1. 获取全局写锁（或基于版本号的乐观锁）
2. 检查目标快照是否存在（最近 500 个或 30 天内）
   - 如果不存在，返回 E005（数据格式错误），提示"无法回滚到该版本"
3. 加载指定版本的快照
4. 生成新的变更记录（类型为 "rollback"）
   - data.snapshot_tree 包含完整规范树
5. 追加到 change_log，版本号 +1
6. 释放锁

并发推送行为：
- 回滚过程中收到的 POST /sync 请求，等待锁释放
- 超时（5 秒）后返回 E007（服务端错误），客户端重试
```

**客户端应用机制：**
```
客户端收到 rollback 变更：
1. 清空本地书签树（从 Chrome 删除所有同步的书签）
2. 按 snapshot_tree 重建完整书签树（BFS 顺序）
3. 重建本地映射表（local_id ↔ guid）
4. 更新 last_known_version = to_version

注意：rollback 变更不触发增量应用，强制全量重建。
```

---

## 5. 功能需求

### 5.1 懒猫微服 (lzc-bookmark)

#### 5.1.1 API 鉴权管理
- API Key 生成：用户在 Web 管理界面手动生成，可设置备注（如"工作电脑"）
- 权限控制：每个 API Key 可设置为**只读**或**读写**
- 设备绑定：API Key 可绑定多个设备（默认不限制），可在管理界面查看和撤销
- 密钥存储：API Key 加盐哈希存储（bcrypt，cost=10）
- **只读 API Key 权限矩阵：**

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

#### 5.1.2 书签存储
- 数据结构：使用 GUID 作为主键，保留完整树结构（通过 parent_guid）
- 规范树：服务端仅存储 GUID、title、url、parent_guid、index_weight，**不存储 local_id**
- 软删除：删除操作标记 `is_deleted=true`，保留 90 天后物理删除
- 版本历史：每次成功同步生成快照，保留最近 500 个版本或最近 30 天
- 物理删除条件：`is_deleted=true` AND `deleted_at > 90 days`（v6 简化，不检查快照引用）

#### 5.1.3 同步逻辑
- 增量同步：基于版本号的变更日志（每条变更 +1 版本）
- 冲突检测：基于 GUID 和 base_version（时间窗口仅用于 UI 提示）
- 冲突解决：支持三种策略（use_local / use_remote / merge）
- 溢出处理：`last_known_version < baseline_version` 时直接返回 200 全量
- 部分冲突：无冲突变更正常应用，冲突变更挂起

#### 5.1.4 Web 管理界面
- 书签浏览：树形结构展示，支持搜索、展开/折叠
- 历史记录：版本列表，支持预览和回滚
- 设备管理：显示已注册设备列表，支持撤销
- API Key 管理：生成、查看、撤销 API Key
- 数据导出：导出完整书签树为 JSON（用于备份）

### 5.2 Chrome 浏览器插件

#### 5.2.1 配置管理
- API 配置：服务端地址（URL）、API Key
- 同步设置：自动同步间隔（默认 30 秒，可配置 10s-300s）
- 设备标识：首次启动时生成 UUID 作为 device_id，存储在 storage.local
- 本地备份：**默认关闭**，用户可开启（导出 JSON 到 chrome.downloads，每日首次同步时）

#### 5.2.2 同步功能
- 全量备份：首次同步时上传完整书签树，接收 GUID 映射表
- 增量备份：监听 chrome.bookmarks 事件，累积变更到离线队列
- 拉取同步：轮询服务端，应用变更日志
- 冲突提示：工具栏图标变红，点击弹出冲突列表
- 全量重拉：收到 `is_baseline=true` 响应时，清空本地映射表，重新全量拉取
- 部分冲突处理：无冲突变更正常应用，冲突变更挂起

#### 5.2.3 UI 展示
- 工具栏图标：
  - 绿色勾：同步完成
  - 蓝色旋转：同步中
  - 红色感叹号：有冲突待解决
  - 灰色叉：离线/同步失败
  - 橙色警告：离线队列接近上限（>80%）
- 弹出面板：显示上次同步时间、待同步变更数、冲突数、队列占用率、手动同步按钮
- 设置页面：API 配置、同步间隔、设备名称、清除本地数据、本地备份开关

#### 5.2.4 后台服务
- 事件监听：使用 chrome.bookmarks.onCreated/onRemoved/onChanged/onMoved
- 定时同步：使用 chrome.alarms API（Manifest V3 推荐方式）
- 离线队列：变更存储在 storage.local（配额 5MB）
- 队列管理：
  - 队列上限：5000 条变更
  - 超限策略：合并同一 GUID 的连续变更（保留最后一次）
  - **幂等保护：仅合并 submitted=false 的变更（从未提交过的）**
  - 存储估算：每条变更约 500 字节，5000 条约 2.5MB，预留 50% 余量
- 降级策略（Service Worker 休眠）：
  - 休眠期间书签变更通过 `chrome.bookmarks.getTree()` 全量 diff 检测
  - 唤醒时调用 `chrome.bookmarks.getTree()` 与本地映射表对比，生成未同步变更
  - 然后走正常的先推后拉流程

---

## 6. API 设计

### 6.1 通用规范

**请求头：**
```
Authorization: Bearer <API_Key>
Content-Type: application/json
X-Device-Id: <device_uuid>
X-Client-Version: 1.0.0
X-Request-Id: <request_uuid>  // 可选，用于全链路追踪
```

**成功响应（统一格式）：**
```json
{
  "success": true,
  "data": { ... },
  "version": 123,
  "synced_at": "2026-04-16T10:00:00Z"
}
```

**错误响应（统一格式）：**
```json
{
  "success": false,
  "error": {
    "code": "E003",
    "message": "版本冲突，当前版本为 125",
    "details": {
      "expected_version": 120,
      "current_version": 125
    }
  }
}
```

**错误码表：**
| 错误码 | HTTP 状态码 | 说明 |
|--------|-------------|------|
| E001 | 401 | API Key 无效或缺失 |
| E002 | 403 | 权限不足（如只读 Key 调用写接口） |
| E003 | 409 | 版本冲突（base_version 严重落后，需全量同步） |
| E004 | 400 | 设备未注册（仅显式调用设备 API 时） |
| E005 | 400 | 数据格式错误（如 GUID 不存在、父节点不存在） |
| E006 | 429 | 请求频率超限 |
| E007 | 500 | 服务端错误 |
| E008 | 410 | 服务端维护模式，稍后重试 |

**409 E003 vs 200 + conflicts 边界：**
```
返回 409 E003 的场景：
- 客户端 base_version 落后超过阈值（如 1000 个版本）
- 客户端 base_version 早于 baseline_version（变更日志已截断）
- 其他需要强制客户端全量同步的场景

返回 200 + conflicts 的场景：
- 正常的业务冲突（同一 GUID 的并发修改）
- 部分变更可应用，部分变更有冲突
```

### 6.2 认证接口

```http
POST /api/v1/auth/verify
```

**响应：**
```json
{
  "success": true,
  "data": {
    "valid": true,
    "permissions": ["read", "write"],
    "key_name": "Work Computer"
  },
  "version": null,
  "synced_at": "2026-04-16T10:00:00Z"
}
```

**说明：** 非书签接口的 `version` 字段为 `null`。

### 6.3 书签接口

#### 6.3.1 获取书签树

```http
GET /api/v1/bookmarks
Query: since_version=<integer>&limit=1000  // limit 可选，默认 1000，最大 1000
```

**响应（全量，无 since_version 或 since_version < baseline_version）：**
```json
{
  "success": true,
  "data": {
    "bookmarks": {
      "guid": "root",
      "title": "",
      "children": [
        {
          "guid": "root_bar",
          "title": "书签栏",
          "parent_guid": "root",
          "index_weight": 1.0,
          "children": [...]
        },
        {
          "guid": "root_other",
          "title": "其他书签",
          "parent_guid": "root",
          "index_weight": 2.0,
          "children": [...]
        },
        {
          "guid": "root_mobile",
          "title": "移动设备书签",
          "parent_guid": "root",
          "index_weight": 3.0,
          "children": [...]
        }
      ]
    },
    "is_baseline": true,
    "baseline_version": 100
  },
  "version": 123,
  "synced_at": "2026-04-16T10:00:00Z"
}
```

**响应（增量，since_version >= baseline_version）：**
```json
{
  "success": true,
  "data": {
    "changes": [
      {
        "change_id": "ch_001",
        "type": "create",
        "guid": "550e8400-...",
        "from_version": 120,
        "to_version": 121,
        "data": { "title": "New Bookmark", "url": "https://...", "parent_guid": "root_bar", "index_weight": 1.0 }
      }
    ],
    "has_more": false
  },
  "version": 123,
  "synced_at": "2026-04-16T10:00:00Z"
}
```

**分页行为：**
```
请求：GET /bookmarks?since_version=120&limit=100
响应：
- 如果变更数 <= 100：has_more=false，返回所有变更
- 如果变更数 > 100：has_more=true，返回前 100 条

客户端续页：
- 使用最后一条变更的 to_version 作为下一次请求的 since_version
- GET /bookmarks?since_version=220&limit=100

分页连续性保证：
- 服务端应保证同一分页序列的连续性
- 若客户端基于上一页继续请求，即使期间触发截断，也应返回剩余变更或空结果，而非 410/全量
```

#### 6.3.2 提交同步

```http
POST /api/v1/bookmarks/sync
```

**请求体：**
```json
{
  "device_id": "chrome-abc123",
  "base_version": 120,
  "client_request_id": "req_uuid_001",
  "changes": [
    {
      "change_id": "ch_local_001",
      "type": "create",
      "guid": null,
      "local_id": "123",
      "parent_local_id": "100",
      "data": { "title": "New Bookmark", "url": "https://...", "parent_guid": "root_bar" }
    },
    {
      "change_id": "ch_local_002",
      "type": "update",
      "guid": "550e8400-...",
      "data": { "title": "Updated Title" }
    }
  ]
}
```

**字段说明：**
- `base_version`: 客户端变更基于的版本号，用于冲突检测
- `client_request_id`: 幂等 ID，重复提交时不重复应用变更，但返回当前最新状态
- `changes`: 变更数组，上限 500 条/请求
- `parent_local_id`: 同批 create 时，父节点的 local_id（服务端解析为正式 GUID）
- **上传顺序：父节点先于子节点（拓扑有序），服务端按顺序处理**

**parent_guid 与 parent_local_id 优先级（v6 明确）：**
```
1. 如果 data.parent_guid 存在且非空，且该 GUID 已存在于规范树 → 使用 parent_guid
2. 否则如果 parent_local_id 存在 → 从同批缓存查找对应 GUID
3. 否则 → 报错 E005（数据格式错误）

两者同时存在且不一致时：
- 优先使用 parent_guid（假设父节点已存在）
- 如果 parent_guid 不存在，fallback 到 parent_local_id
```

**同批 create 的 GUID 分配协议：**
```
场景：客户端一次性上传 500 条 create 变更，其中包含父子关系

协议：
1. 客户端按拓扑顺序排序变更（父节点在前，子节点在后）
2. 子节点的 parent_guid 字段填写服务端规范树中的 GUID（如果父节点已存在）
3. 如果父节点也是本次上传的新节点：
   - 使用 parent_local_id 字段指向父节点的 local_id
   - 服务端按顺序处理，遇到父节点时生成 GUID 并缓存
   - 处理子节点时，从缓存中查找父节点的 GUID
4. 三根节点（root_bar/root_other/root_mobile）是特例：
   - change.guid 预填固定值（非 null）
   - 服务端直接使用这些固定 GUID，不重新生成

示例：
[
  { "type": "create", "local_id": "100", "guid": "root_bar", "data": { "title": "书签栏" } },
  { "type": "create", "local_id": "101", "parent_local_id": "100", "data": { "title": "文件夹 A" } }
]

服务端处理：
1. 处理第一条：识别 guid="root_bar" 为固定值，直接使用，缓存 { "100": "root_bar" }
2. 处理第二条：从缓存查找 "100" → "root_bar"，设置 parent_guid="root_bar"
```

**幂等行为（v6 明确）：**
```
检测到重复的 client_request_id：
1. 不重复应用变更（幂等）
2. 返回当前最新状态（可能包含其他设备的变更）
3. version 字段返回当前全局最新版本
4. 如果原始请求产生了冲突记录，重试响应仍返回相同的 conflicts 数组

示例：
首次请求：client_request_id="req_001", base_version=120
  → 应用变更，返回 version=125，conflicts=[cf_001]

重试请求：client_request_id="req_001", base_version=120（相同）
  → 不应用变更，返回 version=128，conflicts=[cf_001]（相同的冲突记录）
```

**响应（成功，无冲突）：**
```json
{
  "success": true,
  "data": {
    "new_version": 125,
    "applied_changes": ["ch_local_001", "ch_local_002"],
    "conflicts": [],
    "generated_mappings": [
      { "local_id": "123", "guid": "550e8400-e29b-41d4-a716-446655440001" }
    ]
  },
  "version": 125,
  "synced_at": "2026-04-16T10:00:00Z"
}
```

**响应（部分冲突）：**
```json
{
  "success": true,
  "data": {
    "new_version": 123,
    "applied_changes": ["ch_local_001"],
    "conflicts": [
      {
        "conflict_id": "cf_001",
        "guid": "550e8400-...",
        "type": "update_conflict",
        "local_change": { "change_id": "ch_local_002", "data": {...} },
        "remote_change": { "change_id": "ch_remote_005", "data": {...} },
        "resolution_hint": "use_local"
      }
    ]
  },
  "version": 128,  // 全局最新版本（可能包含其他设备的变更）
  "synced_at": "2026-04-16T10:00:00Z"
}
```

**客户端状态更新规则：**
```
收到部分冲突响应：
1. 更新 last_known_version = new_version（与已应用的变更对齐）
2. 从离线队列移除 applied_changes 中的变更（标记 submitted=true）
3. 保留冲突变更，标记为"待解决"，更新 base_version = new_version
4. 下次推送时，仅推送冲突变更 + 新增变更

收到全部冲突响应（applied_changes 为空）：
1. 不更新 last_known_version
2. 所有变更保留在离线队列，标记为"待解决"
3. 下次推送时，重新推送所有变更（base_version 不变）

收到 409 E003 响应：
1. 不更新 last_known_version
2. 触发全量同步（GET /bookmarks 无 since_version）
```

#### 6.3.3 解决冲突

```http
POST /api/v1/bookmarks/conflicts/{conflict_id}/resolve
```

**请求体（use_local / use_remote）：**
```json
{
  "resolution": "use_local"
}
```

**请求体（merge）：**
```json
{
  "resolution": "merge",
  "merged_data": {
    "title": "Merged Title",
    "url": "https://...",
    "parent_guid": "root_bar",
    "index_weight": 1.5
  }
}
```

**resolution 选项：**
- `use_local`: 使用客户端提交的变更
- `use_remote`: 使用服务端已有的变更
- `merge`: 字段级合并（仅 title/url 可合并，parent_guid/index_weight 必须与 local 或 remote 一致）

**响应：**
```json
{
  "success": true,
  "data": {
    "conflict_id": "cf_001",
    "resolution": "use_local",
    "new_version": 126
  },
  "version": 126,
  "synced_at": "2026-04-16T10:00:00Z"
}
```

### 6.4 设备接口

```http
# 注册设备（首次同步时自动调用，也可手动）
POST /api/v1/devices
Body: { "name": "Work Computer", "device_id": "chrome-abc123" }
Response: {
  "success": true,
  "data": { "device_id": "chrome-abc123", "registered": true },
  "version": null
}

# 列出设备
GET /api/v1/devices
Response: {
  "success": true,
  "data": {
    "devices": [
      { "device_id": "chrome-abc123", "name": "Work Computer", "last_seen": "2026-04-16T10:00:00Z" }
    ]
  },
  "version": null
}

# 撤销设备
DELETE /api/v1/devices/<device_id>
Response: {
  "success": true,
  "data": { "device_id": "chrome-abc123", "revoked": true },
  "version": null
}
```

**API Key 与 Device 关系：**
- 一个 API Key 可以绑定多个 Device
- Device 表存储 `api_key_id`（外键），不直接存储 `api_key_hash`
- 限流维度：`api_key_id + device_id` 组合
- **权限始终从 API Key 查询，Device 不冗余存储 permissions**

### 6.5 历史接口

```http
# 获取历史版本列表
GET /api/v1/bookmarks/history
Query: limit=50&offset=0
Response: {
  "success": true,
  "data": {
    "versions": [
      { "version": 123, "synced_at": "2026-04-16T10:00:00Z", "device_id": "chrome-abc123", "changes_count": 5 }
    ],
    "total": 500
  },
  "version": null
}

# 获取指定版本的快照
GET /api/v1/bookmarks/history/<version>
Response: {
  "success": true,
  "data": { "bookmarks": {...}, "version": 123 },
  "version": null
}

# 回滚到指定版本
POST /api/v1/bookmarks/rollback
Body: { "version": 100, "confirm": true }  // confirm 必填，防止误操作
Response: {
  "success": true,
  "data": { "new_version": 126, "rolled_back_to": 100 },
  "version": 126
}
```

**说明：** 回滚操作生成新版本（而非原地覆盖），便于审计和再次回滚。

**回滚限制（v6 新增）：**
- 仅可回滚到最近 30 天内的版本（或最近 500 个版本）
- 回滚跨度不超过 1000 个版本
- 需要 `confirm: true` 参数确认

### 6.6 限流策略

| 接口 | 限流 |
|------|------|
| /api/v1/bookmarks/sync | 每 API Key + 每 Device 每分钟 60 次 |
| 其他接口 | 每 API Key + 每 Device 每分钟 120 次 |
| **单 API Key 总限流** | **每分钟 200 次**（兜底上限） |

**超限响应：** HTTP 429，Header: `Retry-After: 60`

**说明：** 限流维度为 **API Key + Device** 组合，同时单 API Key 有总上限防止滥用。

### 6.7 数据上限

| 限制项 | 上限 | 超限行为 |
|--------|------|----------|
| 单用户书签总数 | 100,000 | 返回 E005，提示清理 |
| 单次 sync 变更数 | 500 | 分批提交 |
| 单文件夹子节点数 | 10,000 | 无限制，但性能可能下降 |
| 离线队列变更数 | 5000 | 合并同一 GUID 的连续变更 |
| GET /bookmarks 响应大小 | 10 MB | 分页返回（has_more=true） |

---

## 7. 数据模型

### 7.1 书签节点 (Bookmark)

```typescript
{
  guid: string           // UUID v4，全局唯一（"root"/"root_bar"等为特殊常量）
  tenant_id: string      // 预留，当前阶段恒为空字符串
  title: string
  url: string | null     // 文件夹为 null
  parent_guid: string    // 父节点 GUID，根目录为 "root"
  index_weight: float64  // 排序权重（双精度浮点数）
  date_created: integer  // 服务端时间戳（Unix 毫秒）
  date_modified: integer // 服务端时间戳
  is_deleted: boolean    // 软删除标记
  deleted_at: integer    // 删除时间（可选）
}
```

### 7.2 同步记录 (SyncRecord)

```typescript
{
  version: integer
  device_id: string
  change_ids: string[]    // 本次 sync 产生的变更 ID 列表（非完整 Change 对象）
  snapshot_hash: string   // 快照的 SHA256（用于快速比对）
  synced_at: integer
}
```

### 7.3 变更记录 (Change)

```typescript
{
  change_id: string       // 变更 ID（UUID），用于幂等
  type: "create" | "update" | "delete" | "move" | "reindex" | "rollback"
  guid: string
  device_id: string
  data: {
    title?: string
    url?: string
    parent_guid?: string
    index_weight?: float64
  } | null
  from_version: integer   // 变更前的版本号
  to_version: integer     // 变更后的版本号（= from_version + 1）
  timestamp: integer      // 服务端接收时间戳
}
```

**特殊变更类型：**
- `reindex`: index_weight 归一化生成
  - `guid`: 被归一化的文件夹 GUID
  - `data`: `{ "children_weights": { "guid-1": 1.0, "guid-2": 2.0, ... } }`
  - 不计入变更日志 1000 条上限判断
  - 但仍写入 change_log 表，客户端可拉取应用
- `rollback`: 回滚操作生成
  - `data`: `{ "rollback_to_version": 100, "snapshot_tree": {...} }`
  - `snapshot_tree`: 目标版本的完整规范树（嵌套结构）
  - 客户端收到后清空本地树，按 snapshot_tree 重建

### 7.4 冲突记录 (Conflict)

```typescript
{
  conflict_id: string
  guid: string
  local_change: Change
  remote_change: Change
  created_at: integer
  status: "pending" | "resolved"
  resolution: "use_local" | "use_remote" | "merge" | null
  resolved_at: integer | null
  resolved_by: string | null  // device_id
}
```

### 7.5 设备 (Device)

```typescript
{
  device_id: string
  name: string
  api_key_id: string      // API Key 的外键（一个 Key 可对应多个 Device）
  created_at: integer
  last_seen: integer
  is_revoked: boolean
}
```

**注意：** Device 模型不包含 `permissions` 字段，权限始终从 API Key 查询。

### 7.6 版本快照 (Snapshot)

```typescript
{
  version: integer
  bookmark_tree: BookmarkNode  // 完整规范树（嵌套树结构，非扁平数组）
  created_at: integer
  is_baseline: boolean         // 是否为基准快照（变更日志溢出参考点）
}

// 嵌套树结构
BookmarkNode {
  guid: string
  title: string
  url: string | null
  parent_guid: string
  index_weight: float64
  date_created: integer
  date_modified: integer
  children: BookmarkNode[]  // 子节点数组（递归）
}
```

**存储估算：**
- 5000 个书签的快照：约 500KB（原始 JSON）
- gzip 压缩后：约 100-150KB
- 500 个快照：约 50-75MB（可接受）

---

## 8. 同步逻辑详解

### 8.1 完整同步流程（先推后拉）

```
┌──────────────┐
│ 插件启动      │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 读取本地      │
│ last_known_  │
│ version       │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│ 有本地变更？  │──是─►│ 加入离线队列  │
└──────┬───────┘     └──────┬───────┘
       │ 否                 │
       │                    ▼
       │            ┌──────────────┐
       │            │ 网络可用？    │───否──► 等待下次同步
       │            └──────┬───────┘
       │                   │ 是
       ▼                   ▼
┌──────────────────────────────────┐
│  POST /api/v1/bookmarks/sync     │
│  推送本地变更（base_version=N）   │
└──────────────┬───────────────────┘
               │
               ▼
       ┌──────────────┐
       │ 有冲突？      │
       └──────┬───────┘
              │
    ┌─────────┴─────────┐
    │                   │
    ▼                   ▼
┌──────────┐      ┌──────────┐
│部分冲突  │      │全部冲突  │
│(部分应用)│      │(无应用)  │
└────┬─────┘      └────┬─────┘
     │                 │
     ▼                 ▼
更新 last_known   不更新
version = new_    last_known
version           version
     │                 │
     └────────┬────────┘
              │
              ▼
┌──────────────────────────────────┐
│  GET /api/v1/bookmarks           │
│  ?since_version=last_known_      │
│  version                         │
│  拉取远程变更                     │
└──────────────┬───────────────────┘
               │
               ▼
       ┌──────────────┐
       │ is_baseline  │
       │ == true?     │───是──► 清空映射表，全量重拉
       └──────┬───────┘
              │ 否
              ▼
       ┌──────────────┐
       │ 有远程变更？  │───是──► 应用变更到本地
       └──────┬───────┘
              │ 否
              ▼
       ┌──────────────┐
       │ 更新本地      │
       │ last_known_  │
       │ version      │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │ 同步完成      │
       └──────────────┘
```

### 8.2 首次同步流程（全量上传）

```
1. 插件读取本地完整书签树（chrome.bookmarks.getTree()）
2. 按拓扑顺序（父节点先于子节点）生成 changes 数组
3. 提取 Chrome 三根（id="1"、"2"、"3"）的子树
4. 为三根节点预填固定 GUID（root_bar/root_other/root_mobile）
5. POST /sync，base_version=0，changes 包含所有节点（type=create）
   - 可分批提交（每批 500 条变更）
6. 服务端为每个节点生成 GUID（三根使用预填的固定 GUID）
   - 按顺序处理，解析 parent_local_id 为正式 GUID
7. 服务端返回 generated_mappings: [{local_id, guid}]
8. 插件存储映射表到 storage.local
9. 后续同步使用 GUID 标识书签

分批提交策略：
- 第一批：base_version=0，client_request_id="batch_001"
- 第二批：base_version=第一批返回的 new_version，client_request_id="batch_002"
- 如果某批失败：重试该批；如果最终部分成功，基于当前状态继续同步
- 书签是覆盖语义，不会丢失数据
```

### 8.3 新设备首次同步流程（全量拉取）

```
1. 插件启动，last_known_version=0
2. GET /bookmarks（无 since_version）
3. 服务端返回完整规范树 + is_baseline=true
4. 解析规范树第一层：
   - guid="root_bar" → 映射到 Chrome id="1"（书签栏）
   - guid="root_other" → 映射到 Chrome id="2"（其他书签）
   - guid="root_mobile" → 映射到 Chrome id="3"（移动设备书签）
5. 按 BFS 顺序创建本地书签（从第二层开始）：
   a. 遍历规范树第二层节点（按 index_weight 升序排序）
   b. 对每个节点调用 chrome.bookmarks.create({parentId: "1" 或 "2" 或 "3", ...})
      - 不传 index 参数，依赖 Chrome 追加语义维持排序
   c. Chrome 返回新节点的 local_id
   d. 建立映射：{local_id ↔ guid}
   e. 递归处理子节点（BFS 顺序）
6. 存储映射表到 storage.local
7. 后续增量同步使用 GUID

注意：如果本地已有书签，先全量上传备份，再拉取服务端数据合并。
```

**BFS 创建顺序示例：**
```
Level 0: root（服务端虚拟根，不创建）
Level 1: root_bar、root_other、root_mobile（映射到 Chrome 固定根，不创建）
Level 2: 书签栏下的文件夹 A、文件夹 B → 调用 create({parentId: "1"})
Level 3: 文件夹 A 下的书签 1、书签 2 → 调用 create({parentId: local_id_of_A})
...
```

### 8.4 冲突检测规则

| 条件 | 说明 |
|------|------|
| 同一 GUID | 两个设备修改了同一个书签 |
| 版本检查 | 变更的 base_version < 该 GUID 最近被修改的 to_version |
| 时间窗口 | 服务端接收时间相差 < 10 秒（仅用于 UI 提示"并发修改"） |

**注意：** 时间窗口不影响冲突判定，仅用于 UI 提示。即使时间相差 > 10 秒，只要版本检查通过，仍判定为冲突。

### 8.5 删除同步策略

| 场景 | 行为 |
|------|------|
| 单设备删除 | 标记 is_deleted=true，同步到其他设备后删除 |
| 删除 vs 修改冲突 | 进入冲突流程，默认推荐保留修改（防误删） |
| 删除恢复 | 从历史版本回滚，重新创建书签 |
| 物理删除 | 软删除 90 天后直接物理删除（v6 简化） |

### 8.6 错误恢复场景

| 场景 | 行为 |
|------|------|
| POST /sync 返回 500 | 不更新 last_known_version，下次重试（client_request_id 相同，幂等） |
| POST /sync 超时 | 重试 3 次（指数退避：1s, 5s, 30s），仍失败则保留队列 |
| 部分变更已应用但响应丢失 | 重试时服务端检测 client_request_id，返回相同 conflicts 数组 |
| 收到 is_baseline=true | 清空本地映射表，全量重拉 |
| 网络完全不可用 | 离线队列持久化，上限 5000 条，超限后合并变更（仅合并 submitted=false 的） |
| 回滚操作中收到 POST /sync | 等待锁释放（最长 5 秒），超时返回 E007，客户端重试 |
| 收到 409 E003 | 触发全量同步（GET /bookmarks 无 since_version） |

---

## 9. 安全考虑

### 9.1 传输安全
- 默认强制 HTTPS（自签名证书需用户手动信任）
- 内网 HTTP 模式：需显式设置环境变量 `ALLOW_INSECURE_HTTP=true`，仅绑定内网 IP 时推荐
- API Key 通过 Header 传递，不在 URL 中暴露
- 敏感操作（撤销设备、回滚历史）记录审计日志
- HTTP 模式下 API Key 明文传输，建议使用专用低权限 API Key

### 9.2 存储安全
- API Key 在插件侧使用 storage.local 存储（不同步到 Google）
- 服务端 API Key 加盐哈希存储（bcrypt，cost=10）
- 书签数据明文存储（当前阶段暂不加密，用户私有部署）
- 不记录书签内容到应用日志

### 9.3 权限控制
- API Key 分只读/读写两种权限
- 设备级权限撤销（立即生效）
- 操作审计日志：记录 api_key_id、设备 ID、操作类型、时间戳、IP（可选）
- **只读 API Key 禁止调用写接口（返回 E002）**
- 回滚操作需要 `confirm: true` 参数确认

### 9.4 速率限制
- 每 API Key + Device 组合独立限流
- 单 API Key 总限流兜底（200 次/分钟）
- 限流信息通过 Response Header 返回：`X-RateLimit-Remaining`
- 超限返回 HTTP 429 + `Retry-After` Header

---

## 10. 开发计划

### Phase 1: 懒猫微服核心 (预计 5 天)
- [ ] LPK 项目创建 (lzc-bookmark) - Go 后端 + React 前端
- [ ] 数据库设计（SQLite schema，预留 tenant_id 字段）
- [ ] API 鉴权模块（API Key 生成/验证，bcrypt 哈希）
- [ ] 书签 CRUD API（含 GUID 生成，不含 local_id）
- [ ] 同步 API（/sync 接口，含冲突检测，先推后拉逻辑）
- [ ] React 管理界面框架（书签列表、API Key 管理）

**依赖：** 无

### Phase 2: Chrome 插件基础 (预计 4 天)
- [ ] Manifest V3 项目搭建
- [ ] chrome.bookmarks API 封装
- [ ] GUID 映射表管理（storage.local）
- [ ] API 配置界面（popup + options 页）
- [ ] 手动同步功能

**依赖：** Phase 1 基础 API 完成

### Phase 3: 同步逻辑 (预计 5 天)
- [ ] 事件监听（onCreated/onRemoved/onChanged/onMoved）
- [ ] 离线队列（storage.local 持久化，上限 5000 条）
- [ ] 增量同步算法（变更累积与提交，先推后拉）
- [ ] 冲突检测与提示 UI
- [ ] chrome.alarms 定时同步
- [ ] Service Worker 休眠降级（唤醒时 getTree diff）

**依赖：** Phase 2 完成

### Phase 4: 完善与测试 (预计 4 天)
- [ ] Web 管理界面完善（历史记录、回滚）
- [ ] 设备管理功能
- [ ] 多设备同步测试（2-3 设备）
- [ ] 边界测试（1000+ 书签、弱网、离线、溢出、部分冲突）
- [ ] 文档编写（用户指南、API 文档）
- [ ] 自动化测试用例编写

**依赖：** Phase 3 完成

### Phase 5: 测试与修复 (预计 2 天)
- [ ] Bug 修复
- [ ] 性能优化
- [ ] 验收测试

**依赖：** Phase 4 完成

**总计：20 天**

---

## 11. 风险与依赖

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| Chrome API 限制 | 书签 API 有速率限制，storage.local 配额 5MB | 本地缓存 + 批量操作，队列上限 5000 条 + 变更合并 |
| 懒猫微服稳定性 | 服务不可用导致同步失败 | 离线队列 + 指数退避重试（1s, 5s, 30s, 1m, 5m, 15m, 1h） |
| 数据冲突复杂 | 多设备并发修改难处理 | 简化策略（最新优先）+ 用户确认，保留完整历史 |
| 自签名证书 | 企业网络可能拦截 | 提供 HTTP 模式（需显式 `ALLOW_INSECURE_HTTP=true`），文档说明风险 |
| Manifest V3 限制 | Service Worker 5 分钟休眠 | 使用 chrome.alarms API，唤醒时 getTree diff |
| 数据丢失 | 同步错误导致书签被覆盖 | 本地备份（可选，默认关闭），导出 JSON 到 chrome.downloads |
| 变更日志溢出 | 长期离线设备无法增量同步 | 直接返回 200 全量响应，客户端重建 |

---

## 12. 验收标准

### 12.1 功能验收
- [ ] 单设备备份/恢复正常
- [ ] 双设备同步正常（添加、修改、删除、移动）
- [ ] 冲突检测准确（模拟并发修改）
- [ ] 历史版本可回滚
- [ ] 离线后自动重连同步
- [ ] API Key 撤销后立即失效
- [ ] 变更日志溢出后全量同步正常（is_baseline=true）
- [ ] 部分冲突处理正常（8 条应用 + 2 条挂起）
- [ ] Chrome 三根映射正确（书签栏、其他书签、移动设备）

### 12.2 性能验收
| 指标 | 目标 | 测试条件 |
|------|------|----------|
| 1000+ 书签全量同步 | < 10 秒 | 本地网络，冷启动 |
| 增量同步（10 条变更） | < 2 秒 | 本地网络 |
| 冲突检测延迟 | < 500ms | 服务端处理时间 |
| 插件内存占用 | < 50MB | Chrome 任务管理器 |
| 服务端响应时间 (P95) | < 200ms | 10 并发请求 |
| 全量重拉（5000 书签） | < 30 秒 | 本地网络 |
| 3 设备并发同步 | < 5 秒/设备 | 同时推送 |
| 10 设备全量拉取 (P95) | < 60 秒 | 同时拉取 |
| 弱网同步（10 条变更） | < 5 秒 | 延迟 200ms，丢包 5% |

### 12.3 体验验收
- [ ] 首次配置流程 < 3 分钟（从安装到首次同步成功）
- [ ] 同步状态清晰可见（工具栏图标 + popup）
- [ ] 错误提示友好（中文，含解决建议）
- [ ] 冲突解决 UI 直观（显示差异，一键选择）
- [ ] 离线队列占用率可视化（>80% 时橙色警告）
- [ ] 部分冲突时，无冲突变更正常应用

### 12.4 兼容性验收
- [ ] Chrome 102+ 正常
- [ ] Edge 102+ 正常
- [ ] 服务端 SQLite / PostgreSQL 均可切换
- [ ] HTTPS / HTTP（内网）模式均可配置

### 12.5 并发验收
- [ ] 3 设备同时同步，无数据丢失
- [ ] 同一书签多设备并发修改，冲突检测准确
- [ ] 限流策略生效（单设备 60 次/分钟后返回 429）
- [ ] 回滚操作原子性（并发推送等待或重试）

### 12.6 压力验收（可选，不阻塞 MVP 发布）
- [ ] 10 设备全量拉取 P95 < 60 秒
- [ ] 单 API Key 下 20 设备同时同步，无数据丢失
- [ ] 变更日志截断后，落后设备全量同步正常

---

## 13. 附录

### 13.1 术语表
| 术语 | 说明 |
|------|------|
| GUID | 全局唯一标识符，服务端生成，跨设备识别同一书签 |
| local_id | Chrome 为每个书签分配的 ID，仅在当前设备有效 |
| 版本号 | 全局递增整数，每条变更 +1 |
| 变更日志 | 记录所有变更的有序列表，用于增量同步 |
| 软删除 | 标记 is_deleted=true，而非物理删除 |
| 规范树 | 服务端存储的书签树（仅 GUID，不含 local_id） |
| 基准快照 | 变更日志溢出时的全量参考点 |
| 先推后拉 | 同步顺序：先提交本地变更，再拉取远程变更 |
| 部分冲突 | 推送的变更中部分有冲突，部分无冲突 |
| Chrome 三根 | 书签栏 (id="1")、其他书签 (id="2")、移动设备书签 (id="3") |

### 13.2 同步协议状态机

```
状态：IDLE → PUSHING → PULLING → IDLE
           │            │
           ▼            ▼
      有冲突？      is_baseline?
           │            │
    ┌──────┴──────┐     │
    │             │     │
    ▼             ▼     ▼
部分冲突      全部冲突  FULL_SYNC
    │             │     │
    ▼             │     │
更新 version      │     │
    │             │     │
    └──────┬──────┘     │
           │            │
           ▼            │
      用户解决 ◄────────┘
           │
           ▼
         IDLE
```

**注意：** 全部冲突后仍应继续拉取远程变更（version 字段已提供最新版本号）。

### 13.3 版本号语义

```
全局版本号规则：
- 每条变更（Change）对应版本号 +1
- 一次 sync 请求提交 N 条变更，版本号 +N
- 回滚操作生成一条 "rollback" 变更，版本号 +1

增量拉取谓词：
- 返回所有满足 to_version > since_version 的变更
- 按 to_version 升序排序
- 每条变更的 to_version 全局唯一（因为每条 +1）

示例：
客户端 last_known_version = 120
推送 5 条变更 → 服务端应用后 version = 125
拉取 since_version=125 → 返回 to_version > 125 的变更
```

### 13.4 Chrome 三根映射表

| 服务端 guid | 服务端 title | Chrome id | Chrome title（中文） | Chrome title（英文） |
|-------------|-------------|-----------|---------------------|---------------------|
| root_bar | 书签栏 | 1 | 书签栏 | Bookmarks Bar |
| root_other | 其他书签 | 2 | 其他书签 | Other Bookmarks |
| root_mobile | 移动设备书签 | 3 | 移动设备书签 | Mobile Bookmarks |

**映射规则：**
- 服务端规范树第一层固定为这 3 个节点
- 新设备拉取时，不创建第一层，直接映射到 Chrome 现有根
- 用户自定义文件夹只能作为第二层及以下的节点
- 第一层节点的 title 字段仅供参考展示，不参与任何逻辑匹配

### 13.5 参考文档
- Chrome Bookmarks API: https://developer.chrome.com/docs/extensions/reference/bookmarks/
- Manifest V3: https://developer.chrome.com/docs/extensions/mv3/intro/
- LPK 框架文档：https://github.com/lazycat/lpk-docs

---

**PRD 版本:** v6.0  
**创建时间:** 2026-04-16  
**状态:** 已评审，待开发  
**评审者:** Claude, Composer 2, GLM-5, GLM-5.1, Kimi Code, Kimi-k2.5  
**关键变更:** 修复 410 触发矛盾（统一为 200 全量）、修正冲突检测规则（支持部分冲突重试）、统一 Snapshot 树类型、明确 parent_guid 优先级、补充 rollback 应用机制、简化 Service Worker 降级策略、简化物理删除条件