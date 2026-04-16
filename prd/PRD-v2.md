# 懒猫书签同步系统 - PRD v2.0

**版本变更：** v1.0 → v2.0  
**变更摘要：** 修复跨设备书签 ID 映射、明确增量同步模型、删除 WebSocket 简化架构、补充删除冲突处理、完善 API 规范  
**评审来源：** GLM-5、Composer 2、Kimi-k2.5 三方审计汇总

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
- 团队用户（未来）：小型团队共享书签集合（v2 规划）

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
└─────────────────┘         └─────────────────┘         └─────────────────┘
```

**说明：** v1 版本仅使用 HTTP REST API，删除 WebSocket 以简化实现。实时性通过插件轮询（默认 30 秒间隔）保证。

### 3.2 组件划分

| 组件 | 技术栈 | 职责 |
|------|--------|------|
| 懒猫微服 | LPK (Go + React) | API 服务、数据存储、冲突处理、Web 管理界面 |
| Chrome 插件 | Manifest V3 | 书签读写、同步触发、UI 展示 |

### 3.3 技术约束

| 约束项 | 要求 | 理由 |
|--------|------|------|
| 懒猫框架 | 必须使用 LPK (Go+React) | 与懒猫微服生态统一 |
| 插件标准 | 必须使用 Manifest V3 | Chrome 官方要求 |
| 通信协议 | 必须 HTTPS | 安全传输 |
| 数据存储 | SQLite（默认）/ PostgreSQL（可选，通过环境变量切换） | 轻量部署 + 扩展性 |
| 部署方式 | Docker 容器 | 懒猫微服标准 |

### 3.4 数据流向

```
插件本地变更 → 本地队列 → POST /sync → 服务端合并 → 返回新版本
                                           ↓
                                      检测冲突？
                                           ↓
                                      是 → 返回冲突列表 → 用户解决
                                           ↓
                                      否 → 更新全局版本
```

---

## 4. 核心设计决策

### 4.1 书签身份模型（关键）

**问题：** Chrome 在不同设备上为同一书签分配不同的本地 ID，无法直接用于跨设备同步。

**解决方案：双层 ID 映射**

```typescript
// 服务端存储结构
Bookmark {
  guid: string           // 全局唯一 ID (UUID v4)，跨设备识别同一书签
  device_id: string      // 来源设备 ID
  local_id: string       // Chrome 本地 ID（仅用于该设备上的引用）
  title: string
  url: string | null
  parent_guid: string    // 父节点的 guid（文件夹为空表示根目录）
  index: integer
  date_added: integer    // Unix 毫秒时间戳
  date_modified: integer
  is_deleted: boolean    // 软删除标记
}

// 插件本地映射表（Chrome storage.local）
IdMapping {
  local_id: string       // Chrome 本地 ID
  guid: string           // 服务端 GUID
  last_synced_version: integer  // 最后同步的版本号
}
```

**工作流程：**
1. 首次同步：插件上传完整书签树，服务端为每个节点生成 GUID，返回映射表
2. 后续同步：插件使用 GUID 标识书签，服务端基于 GUID 检测冲突
3. 新设备：拉取服务端数据时，建立本地 ID 与 GUID 的映射

### 4.2 增量同步模型

**设计选择：版本号 + 变更日志（Change Log）**

```typescript
// 服务端维护
GlobalState {
  current_version: integer    // 全局版本号，每次成功同步 +1
  change_log: Change[]        // 变更记录（保留最近 1000 条）
}

// 变更类型
Change {
  change_id: string           // 变更 ID（用于幂等）
  type: "create" | "update" | "delete" | "move"
  guid: string                // 书签 GUID
  device_id: string           // 来源设备
  data: Bookmark | null       // create/update 时包含数据，delete 为 null
  timestamp: integer          // Unix 毫秒时间戳
  base_version: integer       // 变更基于的版本号
}
```

**同步流程：**
1. 插件携带 `last_known_version` 请求 `/api/v1/bookmarks/sync`
2. 服务端返回 `change_log` 中 `base_version > last_known_version` 的所有变更
3. 插件应用变更，更新本地映射表
4. 插件提交本地变更（包含 `changes` 数组）
5. 服务端检测冲突，无冲突则追加到 `change_log`，版本号 +1

### 4.3 冲突检测与解决

**冲突类型：**
| 类型 | 检测条件 | 解决策略 |
|------|----------|----------|
| 更新冲突 | 同一 GUID，两个设备在版本 N 到 N+1 之间都修改了 | 用户选择 / 最新优先 |
| 删除冲突 | 设备 A 删除，设备 B 修改同一 GUID | 默认保留修改（防误删） |
| 移动冲突 | 同一 GUID，两个设备移动到不同父文件夹 | 用户选择 |
| 重命名冲突 | 同一 GUID，两个设备修改了标题 | 最新优先 |

**冲突解决流程：**
```
服务端检测冲突 → 返回 conflicts 数组（HTTP 200，非错误）
         ↓
插件标记为"待解决"状态，用户可见但未应用
         ↓
用户选择解决方案（插件 UI）
         ↓
POST /api/v1/bookmarks/conflict/resolve
         ↓
服务端应用解决方案，生成新版本
```

---

## 5. 功能需求

### 5.1 懒猫微服 (lzc-bookmark)

#### 5.1.1 API 鉴权管理
- API Key 生成：用户在 Web 管理界面手动生成，可设置备注（如"工作电脑"）
- 权限控制：每个 API Key 可设置为只读或读写
- 设备绑定：API Key 可绑定多个设备（默认不限制），可在管理界面查看和撤销

#### 5.1.2 书签存储
- 数据结构：使用 GUID 作为主键，保留完整树结构（通过 parent_guid）
- 软删除：删除操作标记 `is_deleted=true`，保留 30 天后物理删除
- 版本历史：每次成功同步生成快照，保留最近 50 个版本

#### 5.1.3 同步逻辑
- 增量同步：基于版本号的变更日志
- 冲突检测：基于 GUID 和时间窗口（服务端接收时间，非客户端时间）
- 冲突解决：支持三种策略（use_local / use_remote / merge）

#### 5.1.4 Web 管理界面
- 书签浏览：树形结构展示，支持搜索、展开/折叠
- 历史记录：版本列表，支持预览和回滚
- 设备管理：显示已注册设备列表，支持撤销
- API Key 管理：生成、查看、撤销 API Key

### 5.2 Chrome 浏览器插件

#### 5.2.1 配置管理
- API 配置：服务端地址（URL）、API Key
- 同步设置：自动同步间隔（默认 30 秒，可配置 10s-300s）
- 设备标识：首次启动时生成 UUID 作为 device_id，存储在 storage.local

#### 5.2.2 同步功能
- 全量备份：首次同步时上传完整书签树，接收 GUID 映射表
- 增量备份：监听 chrome.bookmarks 事件，累积变更，定时推送
- 拉取同步：轮询服务端，应用变更日志
- 冲突提示：工具栏图标变红，点击弹出冲突列表

#### 5.2.3 UI 展示
- 工具栏图标：
  - 绿色勾：同步完成
  - 蓝色旋转：同步中
  - 红色感叹号：有冲突待解决
  - 灰色叉：离线/同步失败
- 弹出面板：显示上次同步时间、待同步变更数、冲突数、手动同步按钮
- 设置页面：API 配置、同步间隔、设备名称、清除本地数据

#### 5.2.4 后台服务
- 事件监听：使用 chrome.bookmarks.onCreated/onRemoved/onChanged/onMoved
- 定时同步：使用 chrome.alarms API（Manifest V3 推荐方式）
- 离线队列：变更存储在 storage.local（配额 5MB），网络恢复后自动同步
- 降级策略：storage.local 满时，暂停队列，提示用户清理

---

## 6. API 设计

### 6.1 通用规范

**请求头：**
```
Authorization: Bearer <API_Key>
Content-Type: application/json
X-Device-Id: <device_uuid>
X-Client-Version: 1.0.0
```

**成功响应：**
```json
{
  "success": true,
  "data": { ... },
  "version": 123,
  "synced_at": "2026-04-16T10:00:00Z"
}
```

**错误响应：**
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
| E002 | 403 | 权限不足 |
| E003 | 409 | 版本冲突 |
| E004 | 400 | 设备未注册（首次同步自动注册） |
| E005 | 400 | 数据格式错误 |
| E006 | 429 | 请求频率超限 |
| E007 | 500 | 服务端错误 |

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
  }
}
```

### 6.3 书签接口

#### 6.3.1 获取书签树

```http
GET /api/v1/bookmarks
Query: since_version=<integer>  // 可选，增量拉取
```

**响应（全量）：**
```json
{
  "success": true,
  "data": {
    "bookmarks": {
      "guid": "root",
      "title": "",
      "children": [...]
    },
    "id_mappings": [
      { "local_id": "1", "guid": "550e8400-e29b-41d4-a716-446655440001" }
    ]
  },
  "version": 123,
  "synced_at": "2026-04-16T10:00:00Z"
}
```

**响应（增量）：**
```json
{
  "success": true,
  "data": {
    "changes": [
      {
        "change_id": "ch_001",
        "type": "create",
        "guid": "550e8400-...",
        "data": { "title": "New Bookmark", "url": "https://..." }
      }
    ]
  },
  "version": 123,
  "synced_at": "2026-04-16T10:00:00Z"
}
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
  "expected_version": 122,
  "client_request_id": "req_uuid_001",
  "changes": [
    {
      "change_id": "ch_local_001",
      "type": "create",
      "guid": "550e8400-...",
      "data": { ... }
    }
  ]
}
```

**字段说明：**
- `base_version`: 客户端变更基于的版本号
- `expected_version`: 乐观锁，期望当前版本号，不匹配则拒绝（防丢失更新）
- `client_request_id`: 幂等 ID，重复提交时返回相同结果

**响应（成功）：**
```json
{
  "success": true,
  "data": {
    "new_version": 125,
    "applied_changes": 3,
    "conflicts": []
  },
  "version": 125,
  "synced_at": "2026-04-16T10:00:00Z"
}
```

**响应（有冲突）：**
```json
{
  "success": true,
  "data": {
    "new_version": 123,
    "applied_changes": 2,
    "conflicts": [
      {
        "conflict_id": "cf_001",
        "guid": "550e8400-...",
        "local_change": { "type": "update", "data": {...} },
        "remote_change": { "type": "update", "data": {...} },
        "resolution_hint": "use_local"
      }
    ]
  },
  "version": 123,
  "synced_at": "2026-04-16T10:00:00Z"
}
```

#### 6.3.3 解决冲突

```http
POST /api/v1/bookmarks/conflict/resolve
```

**请求体：**
```json
{
  "conflict_id": "cf_001",
  "resolution": "use_local"
}
```

**resolution 选项：**
- `use_local`: 使用客户端提交的变更
- `use_remote`: 使用服务端已有的变更
- `merge`: 合并（仅标题/URL 字段级合并，需额外 `merged_data` 字段）

### 6.4 设备接口

```http
# 注册设备（首次同步时自动调用，也可手动）
POST /api/v1/devices
Body: { "name": "Work Computer", "device_id": "chrome-abc123" }

# 列出设备
GET /api/v1/devices
Response: {
  "devices": [
    { "device_id": "chrome-abc123", "name": "Work Computer", "last_seen": "..." }
  ]
}

# 撤销设备
DELETE /api/v1/devices/<device_id>
```

### 6.5 历史接口

```http
# 获取历史版本列表
GET /api/v1/bookmarks/history
Query: limit=50&offset=0
Response: {
  "versions": [
    { "version": 123, "synced_at": "...", "device_id": "...", "changes_count": 5 }
  ],
  "total": 50
}

# 获取指定版本的快照
GET /api/v1/bookmarks/history/<version>
Response: { "bookmarks": {...}, "version": 123 }

# 回滚到指定版本
POST /api/v1/bookmarks/rollback
Body: { "version": 100 }
Response: { "success": true, "new_version": 126 }
```

**说明：** 回滚操作生成新版本（而非原地覆盖），便于审计和再次回滚。

### 6.6 限流策略

| 接口 | 限流 |
|------|------|
| /api/v1/bookmarks/sync | 每设备每分钟 60 次 |
| 其他接口 | 每设备每分钟 120 次 |

**超限响应：** HTTP 429，Header: `Retry-After: 60`

---

## 7. 数据模型

### 7.1 书签节点 (Bookmark)

```typescript
{
  guid: string           // UUID v4，全局唯一
  device_id: string      // 来源设备 ID
  local_id: string       // Chrome 本地 ID（仅在该设备上有意义）
  title: string
  url: string | null     // 文件夹为 null
  parent_guid: string    // 父节点 GUID，根目录为 "root"
  index: integer         // 在父文件夹中的排序
  date_added: integer    // Unix 毫秒时间戳
  date_modified: integer
  is_deleted: boolean    // 软删除标记
  synced_version: integer // 最后同步的版本号
}
```

### 7.2 同步记录 (SyncRecord)

```typescript
{
  version: integer
  device_id: string
  change_log: Change[]
  snapshot_hash: string   // 快照的 SHA256（用于快速比对）
  synced_at: integer
}
```

### 7.3 变更记录 (Change)

```typescript
{
  change_id: string       // 变更 ID（UUID），用于幂等
  type: "create" | "update" | "delete" | "move"
  guid: string
  device_id: string
  data: {
    title?: string
    url?: string
    parent_guid?: string
    index?: integer
  } | null
  timestamp: integer
  base_version: integer
}
```

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
  api_key_hash: string    // API Key 的哈希
  permissions: string[]
  created_at: integer
  last_seen: integer
  is_revoked: boolean
}
```

---

## 8. 同步逻辑详解

### 8.1 完整同步流程

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
┌──────────────┐     ┌──────────────┐
│ GET /bookmarks│    │ POST /sync   │
│ ?since_version│    │ 提交变更      │
└──────┬───────┘     └──────┬───────┘
       │                    │
       ▼                    ▼
┌──────────────┐     ┌──────────────┐
│ 有远程变更？  │     │ 有冲突？      │───是──► 提示用户解决
└──────┬───────┘     └──────┬───────┘
       │ 是                 │ 否
       ▼                    ▼
┌──────────────┐     ┌──────────────┐
│ 应用变更到本地│     │ 更新本地      │
│ 更新映射表    │     │ last_known_  │
└──────┬───────┘     │ version      │
       │             └──────┬───────┘
       ▼                    │
┌──────────────┐            │
│ 更新本地      │◄───────────┘
│ last_known_  │
│ version      │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 同步完成      │
└──────────────┘
```

### 8.2 首次同步流程

```
1. 插件上传完整书签树（POST /sync，base_version=0）
2. 服务端为每个节点生成 GUID
3. 服务端返回 id_mappings 映射表
4. 插件存储映射表到 storage.local
5. 后续同步使用 GUID 标识书签
```

### 8.3 冲突检测规则

| 条件 | 说明 |
|------|------|
| 同一 GUID | 两个设备修改了同一个书签 |
| 版本区间重叠 | 两个变更的 base_version 相同或重叠 |
| 时间窗口 | 服务端接收时间相差 < 10 秒（宽容窗口） |

### 8.4 删除同步策略

| 场景 | 行为 |
|------|------|
| 单设备删除 | 标记 is_deleted=true，同步到其他设备后删除 |
| 删除 vs 修改冲突 | 默认保留修改（防误删），用户可手动选择删除 |
| 删除恢复 | 从历史版本回滚，重新创建书签 |

---

## 9. 安全考虑

### 9.1 传输安全
- 强制 HTTPS（自签名证书需用户手动信任）
- API Key 通过 Header 传递，不在 URL 中暴露
- 敏感操作（撤销设备、回滚历史）记录审计日志

### 9.2 存储安全
- API Key 在插件侧使用 storage.local 存储（不同步到 Google）
- 服务端 API Key 加盐哈希存储（bcrypt）
- 书签数据明文存储（v1 暂不加密，用户私有部署）
- 不记录书签内容到应用日志

### 9.3 权限控制
- API Key 分只读/读写两种权限
- 设备级权限撤销（立即生效）
- 操作审计日志：记录 API Key、设备 ID、操作类型、时间戳

### 9.4 速率限制
- 每 API Key 独立限流
- 限流信息通过 Response Header 返回：`X-RateLimit-Remaining`

---

## 10. 开发计划

### Phase 1: 懒猫微服核心 (预计 5 天)
- [ ] LPK 项目创建 (lzc-bookmark) - Go 后端 + React 前端
- [ ] 数据库设计（SQLite schema）
- [ ] API 鉴权模块（API Key 生成/验证）
- [ ] 书签 CRUD API（含 GUID 生成）
- [ ] 同步 API（/sync 接口，含冲突检测）
- [ ] React 管理界面框架（书签列表、API Key 管理）

### Phase 2: Chrome 插件基础 (预计 4 天)
- [ ] Manifest V3 项目搭建
- [ ] chrome.bookmarks API 封装
- [ ] ID 映射表管理（storage.local）
- [ ] API 配置界面（popup + options 页）
- [ ] 手动同步功能

### Phase 3: 同步逻辑 (预计 5 天)
- [ ] 事件监听（onCreated/onRemoved/onChanged/onMoved）
- [ ] 离线队列（storage.local 持久化）
- [ ] 增量同步算法（变更累积与提交）
- [ ] 冲突检测与提示 UI
- [ ] chrome.alarms 定时同步

### Phase 4: 完善与测试 (预计 4 天)
- [ ] Web 管理界面完善（历史记录、回滚）
- [ ] 设备管理功能
- [ ] 多设备同步测试（2-3 设备）
- [ ] 边界测试（1000+ 书签、弱网、离线）
- [ ] 文档编写（用户指南、API 文档）

### Phase 5: 测试与修复 (预计 2 天)
- [ ] Bug 修复
- [ ] 性能优化
- [ ] 验收测试

**总计：20 天**（原 14 天 + 6 天 buffer）

---

## 11. 风险与依赖

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| Chrome API 限制 | 书签 API 有速率限制，storage.local 配额 5MB | 本地缓存 + 批量操作，队列满时提示用户 |
| 懒猫微服稳定性 | 服务不可用导致同步失败 | 离线队列 + 指数退避重试（1m, 5m, 15m, 1h） |
| 数据冲突复杂 | 多设备并发修改难处理 | 简化策略（最新优先）+ 用户确认，保留完整历史 |
| 自签名证书 | 企业网络可能拦截 | 提供 HTTP 模式（仅内网使用），文档说明风险 |
| Manifest V3 限制 | Service Worker 5 分钟休眠 | 使用 chrome.alarms API，接受延迟同步 |
| 数据丢失 | 同步错误导致书签被覆盖 | 同步前自动本地备份（导出 JSON 到本地） |

---

## 12. 验收标准

### 12.1 功能验收
- [ ] 单设备备份/恢复正常
- [ ] 双设备同步正常（添加、修改、删除）
- [ ] 冲突检测准确（模拟并发修改）
- [ ] 历史版本可回滚
- [ ] 离线后自动重连同步
- [ ] API Key 撤销后立即失效

### 12.2 性能验收
| 指标 | 目标 | 测试条件 |
|------|------|----------|
| 1000+ 书签全量同步 | < 10 秒 | 本地网络，冷启动 |
| 增量同步（10 条变更） | < 2 秒 | 本地网络 |
| 冲突检测延迟 | < 500ms | 服务端处理时间 |
| 插件内存占用 | < 50MB | Chrome 任务管理器 |
| 服务端响应时间 (P95) | < 200ms | 10 并发请求 |

### 12.3 体验验收
- [ ] 首次配置流程 < 3 分钟（从安装到首次同步成功）
- [ ] 同步状态清晰可见（工具栏图标 + popup）
- [ ] 错误提示友好（中文，含解决建议）
- [ ] 冲突解决 UI 直观（显示差异，一键选择）

### 12.4 兼容性验收
- [ ] Chrome 102+ 正常
- [ ] Edge 102+ 正常
- [ ] 服务端 SQLite / PostgreSQL 均可切换

---

## 13. 附录

### 13.1 术语表
| 术语 | 说明 |
|------|------|
| GUID | 全局唯一标识符，服务端生成，跨设备识别同一书签 |
| 本地 ID | Chrome 为每个书签分配的 ID，仅在当前设备有效 |
| 版本号 | 全局递增整数，每次成功同步 +1 |
| 变更日志 | 记录所有变更的有序列表，用于增量同步 |
| 软删除 | 标记 is_deleted=true，而非物理删除 |

### 13.2 参考文档
- Chrome Bookmarks API: https://developer.chrome.com/docs/extensions/reference/bookmarks/
- Manifest V3: https://developer.chrome.com/docs/extensions/mv3/intro/
- LPK 框架文档：https://github.com/lazycat/lpk-docs

---

**PRD 版本:** v2.0  
**创建时间:** 2026-04-16  
**状态:** 已评审，待开发  
**评审者:** GLM-5, Composer 2, Kimi-k2.5