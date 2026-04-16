# 懒猫书签同步系统 - 技术方案 v2.0

**基于 PRD v6.0**  
**创建时间:** 2026-04-16  
**修订时间:** 2026-04-16 (v2.0)  
**状态:** 已评审，待开发

**变更摘要 (v1 → v2):**
- 修复 CGO_ENABLED=0 与 SQLite 矛盾（改用 modernc.org/sqlite）
- 修复 NextVersion 竞态条件（使用 UPDATE ... RETURNING）
- 修复 AcquireLock goroutine 泄漏（改用 TryLock + 轮询）
- 修复冲突检测算法（使用 baseVersion 而非 change.FromVersion）
- 修复 NormalizeWeights 查询语法（使用 guid 列而非 JSON 提取）
- 修复 conflicts 表设计（存储完整 Change JSON 而非外键）
- 新增 sync_records 表
- 新增 Web 管理端技术栈（React + Vite）
- 删除 JWT_SECRET 环境变量
- 修复 host_permissions（改用 optional_host_permissions）
- 修复 Docker HEALTHCHECK（添加 wget 安装）
- 修复 Prometheus 指标（移除 device_id label）
- 新增归一化事务包裹
- 优化冲突检测 N+1 查询问题
- 修复离线队列 create 合并 bug
- 升级 Go 版本至 1.23
- 删除 bookmarks 表多余字段（created_by/modified_by）
- 新增 Dockerfile 前端构建步骤
- 修复告警 PromQL 表达式

**变更摘要 (v2.0 → v2.1):**
- 包管理器统一为 **pnpm**（替代 npm）
- Web 管理端 UI 使用 **shadcn/ui**（替代原生 Tailwind）
- Dockerfile 前端构建改为 pnpm

---

## 1. 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        懒猫书签系统                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐         ┌─────────────────┐               │
│  │  Chrome 插件    │         │   Web 管理界面   │               │
│  │  (Manifest V3)  │         │   (React SPA)   │               │
│  │                 │         │                 │               │
│  │  - 书签同步     │         │  - 书签浏览     │               │
│  │  - 冲突处理     │         │  - 历史记录     │               │
│  │  - 离线队列     │         │  - 设备管理     │               │
│  └────────┬────────┘         └────────┬────────┘               │
│           │                           │                         │
│           │         HTTP/HTTPS        │                         │
│           └─────────────┬─────────────┘                         │
│                         │                                       │
│                         ▼                                       │
│              ┌──────────────────────┐                          │
│              │     Go API 服务       │                          │
│              │      (LPK 框架)       │                          │
│              │                      │                          │
│              │  - 认证鉴权          │                          │
│              │  - 同步逻辑          │                          │
│              │  - 冲突检测          │                          │
│              │  - 版本管理          │                          │
│              └──────────┬───────────┘                          │
│                         │                                       │
│                         ▼                                       │
│              ┌──────────────────────┐                          │
│              │      SQLite DB       │                          │
│              │   (可切换 PostgreSQL) │                          │
│              │                      │                          │
│              │  - bookmarks 表      │                          │
│              │  - change_log 表     │                          │
│              │  - sync_records 表   │                          │
│              │  - snapshots 表      │                          │
│              │  - api_keys 表       │                          │
│              │  - devices 表        │                          │
│              │  - conflicts 表      │                          │
│              └──────────────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 服务端技术栈选型

### 2.1 核心框架

| 组件 | 选型 | 版本 | 理由 |
|------|------|------|------|
| **Go 语言** | [Go](https://go.dev/) | 1.23+ | 最新 LTS 版本、性能优化、TryLock 支持 |
| **Web 框架** | [Gin](https://github.com/gin-gonic/gin) | v1.9+ | 高性能、中间件生态丰富、LPK 默认支持 |
| **ORM** | [GORM v2](https://gorm.io/) | v2.5+ | 支持 SQLite/PostgreSQL 双后端、关联查询方便 |
| **SQLite 驱动** | [modernc.org/sqlite](https://pkg.go.dev/modernc.org/sqlite) | v1.28+ | **纯 Go 实现，无需 CGO**（修复 v1 问题） |
| **PostgreSQL 驱动** | [pgx](https://github.com/jackc/pgx) | v5.5+ | 高性能、支持 RETURNING 子句 |
| **迁移工具** | [golang-migrate](https://github.com/golang-migrate/migrate) | v4.16+ | 纯 Go 实现、支持多数据库、版本管理清晰 |
| **配置管理** | [Viper](https://github.com/spf13/viper) | v1.18+ | 支持环境变量、YAML、热重载 |
| **日志** | [Zap](https://github.com/uber-go/zap) | v1.26+ | 高性能结构化日志、支持日志轮转 |
| **验证** | [go-playground/validator](https://github.com/go-playground/validator) | v10.15+ | 标签式验证、错误信息友好 |
| **UUID** | [google/uuid](https://github.com/google/uuid) | v1.4+ | 标准库、RFC 4122 compliant |
| **密码哈希** | [bcrypt](https://golang.org/x/crypto/bcrypt) | stdlib | Go 标准库、安全可靠 |
| **压缩** | [gzip](https://pkg.go.dev/compress/gzip) | stdlib | Go 标准库、快照存储压缩 |
| **锁** | [sync.Mutex](https://pkg.go.dev/sync#Mutex) | stdlib | 回滚操作用全局写锁（单实例） |

### 2.2 LPK 框架集成

懒猫微服 (LPK) 提供以下基础设施：

```yaml
# LPK 标准项目结构
lzc-bookmark/
├── cmd/
│   └── server/
│       └── main.go           # 入口文件
├── internal/
│   ├── config/               # 配置管理
│   ├── handler/              # HTTP Handler
│   ├── service/              # 业务逻辑
│   ├── repository/           # 数据访问层
│   ├── model/                # 数据模型
│   └── middleware/           # 中间件（鉴权、限流等）
├── pkg/
│   ├── sync/                 # 同步核心算法
│   ├── conflict/             # 冲突检测
│   └── version/              # 版本号管理
├── web/                      # React 前端（见 §4）
├── migrations/               # 数据库迁移脚本
├── Dockerfile
├── go.mod
└── config.yaml.example
```

**LPK 提供的开箱即用能力：**
- Docker 构建与部署
- 环境变量配置
- 健康检查端点（自动注册 `/health`）
- 日志收集（JSON 格式）
- 指标暴露（Prometheus `/metrics`）

---

## 3. 数据库设计

### 3.1 Schema 总览

```sql
-- SQLite schema (PostgreSQL 兼容)
-- 注意：所有时间戳使用 BIGINT（Unix 毫秒），避免 PostgreSQL INTEGER 溢出

-- API Keys 表
CREATE TABLE api_keys (
    id              TEXT PRIMARY KEY,     -- UUID v4
    name            TEXT NOT NULL,        -- 备注名称（如"工作电脑"）
    key_hash        TEXT NOT NULL,        -- bcrypt 哈希
    permissions     TEXT NOT NULL,        -- JSON: ["read"] 或 ["read", "write"]
    created_at      BIGINT NOT NULL,      -- Unix 毫秒
    last_used_at    BIGINT,               -- Unix 毫秒
    is_revoked      INTEGER NOT NULL DEFAULT 0  -- 0=false, 1=true
);

-- Devices 表
CREATE TABLE devices (
    device_id       TEXT PRIMARY KEY,     -- UUID v4
    name            TEXT NOT NULL,        -- 设备名称
    api_key_id      TEXT NOT NULL,        -- 外键 → api_keys.id
    created_at      BIGINT NOT NULL,
    last_seen_at    BIGINT NOT NULL,
    is_revoked      INTEGER NOT NULL DEFAULT 0,
    FOREIGN KEY (api_key_id) REFERENCES api_keys(id) ON DELETE CASCADE
);

-- Bookmarks 表（规范树）
CREATE TABLE bookmarks (
    guid            TEXT PRIMARY KEY,     -- UUID v4 或固定常量
    tenant_id       TEXT NOT NULL DEFAULT '',  -- 预留，v1 恒为空
    title           TEXT NOT NULL,
    url             TEXT,                 -- 文件夹为 NULL
    parent_guid     TEXT NOT NULL,        -- "root" 或父节点 GUID
    index_weight    REAL NOT NULL,        -- float64 排序权重
    date_created    BIGINT NOT NULL,      -- Unix 毫秒
    date_modified   BIGINT NOT NULL,      -- Unix 毫秒
    is_deleted      INTEGER NOT NULL DEFAULT 0,
    deleted_at      BIGINT,               -- Unix 毫秒
    FOREIGN KEY (parent_guid) REFERENCES bookmarks(guid)
    -- 注意：物理删除前必须递归处理子树，不能依赖数据库级联
);

-- 索引
CREATE INDEX idx_bookmarks_parent ON bookmarks(parent_guid);
CREATE INDEX idx_bookmarks_deleted ON bookmarks(is_deleted, deleted_at);
CREATE INDEX idx_bookmarks_tenant ON bookmarks(tenant_id);

-- Change Log 表
CREATE TABLE change_log (
    change_id       TEXT PRIMARY KEY,     -- UUID v4
    type            TEXT NOT NULL,        -- create/update/delete/move/reindex/rollback
    guid            TEXT NOT NULL,        -- 书签 GUID
    device_id       TEXT NOT NULL,        -- 来源设备
    data            TEXT,                 -- JSON（delete 时为空）
    from_version    BIGINT NOT NULL,
    to_version      BIGINT NOT NULL UNIQUE,  -- 全局唯一，每条 +1
    timestamp       BIGINT NOT NULL,      -- Unix 毫秒（服务端接收时间）
    client_request_id TEXT,               -- 幂等 ID（可选）
    CHECK (type IN ('create', 'update', 'delete', 'move', 'reindex', 'rollback'))
);

-- 索引
CREATE INDEX idx_change_log_guid ON change_log(guid);
CREATE INDEX idx_change_log_to_version ON change_log(to_version);
CREATE INDEX idx_change_log_timestamp ON change_log(timestamp);
CREATE INDEX idx_change_log_type ON change_log(type);

-- Sync Records 表（v2 新增）
CREATE TABLE sync_records (
    version         BIGINT PRIMARY KEY,   -- 与 change_log.to_version 对应
    device_id       TEXT NOT NULL,        -- 来源设备
    change_ids      TEXT NOT NULL,        -- JSON 数组：["change_id1", "change_id2", ...]
    snapshot_hash   TEXT,                 -- 快照 SHA256（可选）
    synced_at       BIGINT NOT NULL       -- Unix 毫秒
);

-- 索引
CREATE INDEX idx_sync_records_device ON sync_records(device_id);
CREATE INDEX idx_sync_records_synced_at ON sync_records(synced_at);

-- Global State 表（单行）
CREATE TABLE global_state (
    id              INTEGER PRIMARY KEY CHECK (id = 1),
    current_version BIGINT NOT NULL DEFAULT 0,
    baseline_version BIGINT NOT NULL DEFAULT 0,
    last_truncated_at BIGINT
);

-- 初始化
INSERT INTO global_state (id, current_version, baseline_version) VALUES (1, 0, 0);

-- Snapshots 表
CREATE TABLE snapshots (
    version         BIGINT PRIMARY KEY,
    bookmark_tree   TEXT NOT NULL,        -- JSON（嵌套树结构，可选 gzip 压缩）
    created_at      BIGINT NOT NULL,
    is_baseline     INTEGER NOT NULL DEFAULT 0
);

-- Conflicts 表（v2 修正：存储完整 Change JSON 而非外键）
CREATE TABLE conflicts (
    conflict_id         TEXT PRIMARY KEY,     -- UUID v4
    guid                TEXT NOT NULL,
    local_change_data   TEXT NOT NULL,        -- 完整 Change JSON（v2 变更）
    remote_change_data  TEXT NOT NULL,        -- 完整 Change JSON（v2 变更）
    status              TEXT NOT NULL,        -- pending/resolved
    resolution          TEXT,                 -- use_local/use_remote/merge
    created_at          BIGINT NOT NULL,
    resolved_at         BIGINT,
    resolved_by         TEXT,                 -- device_id
    -- 移除外键约束，避免 change_log 截断时冲突
    -- local_change_id 和 remote_change_id 不再存储
);

-- 索引
CREATE INDEX idx_conflicts_status ON conflicts(status);
CREATE INDEX idx_conflicts_guid ON conflicts(guid);

-- Audit Log 表（v2 新增，对应 PRD §9.3 审计日志要求）
CREATE TABLE audit_log (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    api_key_id      TEXT NOT NULL,
    device_id       TEXT NOT NULL,
    operation       TEXT NOT NULL,        -- sync/rollback/device_revoke 等
    resource        TEXT,                 -- 资源标识（如 guid）
    result          TEXT NOT NULL,        -- success/error
    ip_address      TEXT,                 -- 客户端 IP（可选）
    timestamp       BIGINT NOT NULL       -- Unix 毫秒
);

-- 索引
CREATE INDEX idx_audit_log_api_key ON audit_log(api_key_id);
CREATE INDEX idx_audit_log_timestamp ON audit_log(timestamp);
```

### 3.2 PostgreSQL 兼容

```sql
-- PostgreSQL 需要调整的类型：
-- SQLite 的 INTEGER/BIGINT → PostgreSQL BIGINT（必须，避免 32 位溢出）
-- SQLite 的 TEXT → PostgreSQL TEXT（兼容）
-- SQLite 的 REAL → PostgreSQL DOUBLE PRECISION
-- 自增主键 → 使用 UUID v4（与 SQLite 一致）

-- PostgreSQL 额外优化：
-- 使用 JSONB 替代 TEXT 存储 JSON 字段（change_log.data, sync_records.change_ids 等）
-- 使用 PARTITION BY RANGE 对 change_log 按 to_version 分区（大表优化）
-- 使用 ADVISORY LOCK 替代 sync.Mutex（多实例部署）

-- PostgreSQL 建表语句示例（差异部分）：
CREATE TABLE change_log (
    change_id       UUID PRIMARY KEY,
    type            TEXT NOT NULL,
    guid            TEXT NOT NULL,
    device_id       TEXT NOT NULL,
    data            JSONB,                -- PostgreSQL 使用 JSONB
    from_version    BIGINT NOT NULL,
    to_version      BIGINT NOT NULL UNIQUE,
    timestamp       BIGINT NOT NULL,
    client_request_id TEXT,
    CONSTRAINT check_type CHECK (type IN ('create', 'update', 'delete', 'move', 'reindex', 'rollback'))
);

-- 分区表示例（可选）：
CREATE TABLE change_log_partitioned (
    LIKE change_log INCLUDING ALL
) PARTITION BY RANGE (to_version);
```

### 3.3 数据量估算

| 表 | 单条大小 | 10 万书签 | 备注 |
|----|----------|----------|------|
| bookmarks | ~200B | 20MB | 10 万节点 |
| change_log | ~300B | 300MB | 100 万条变更（10 倍于书签数） |
| sync_records | ~200B | 200MB | 100 万条同步记录 |
| snapshots | ~100KB | 50MB | 500 个快照（gzip 压缩后） |
| conflicts | ~1KB | ~0 | 临时表，解决后归档 |
| audit_log | ~100B | 100MB | 100 万条审计日志 |
| **总计** | - | **~670MB** | 可接受（SQLite 推荐 < 1GB） |

---

## 4. Chrome 插件与 Web 管理端技术栈

### 4.1 Chrome 插件核心技术栈

| 组件 | 选型 | 版本 | 理由 |
|------|------|------|------|
| **包管理器** | [pnpm](https://pnpm.io/) | v9+ | 快速、节省磁盘空间、与 npm 兼容 |
| **构建工具** | [Vite](https://vitejs.dev/) + [CRXJS](https://crxjs.dev/vite) | v5+ | 快速 HMR、Manifest V3 支持好 |
| **框架** | [Preact](https://preactjs.com/) | v10+ | 轻量（3KB）、React 兼容、适合插件 |
| **UI 方案** | **手写简单组件** + Tailwind CSS | - | 轻量（~50KB）、AI 友好、无外部依赖 |
| **状态管理** | [Zustand](https://zustand-demo.pmnd.rs/) | v4+ | 轻量、无样板代码 |
| **HTTP 客户端** | [Ky](https://github.com/sindresorhus/ky) | v1+ | 轻量、Promise 友好、自动重试 |
| **存储封装** | [chrome.storage.local](https://developer.chrome.com/docs/extensions/reference/storage/) | native | Manifest V3 推荐 |
| **UUID 生成** | [crypto.randomUUID()](https://developer.chrome.com/docs/extensions/reference/crypto/) | native | Chrome 102+ 原生支持 |
| **图标** | [Lucide Icons](https://lucide.dev/) | - | 开源、一致风格 |

**手写组件说明：**
- 插件只需要简单组件（Button、Input、Switch、Badge、Card、List）
- 手写约 200 行代码即可满足需求，打包体积 ~50KB
- 相比 shadcn/ui（~200KB+）更轻量
- 代码完全可控，AI 友好（模式简单，易于生成和维护）

**Content Script 说明：**
- **当前版本不使用 content script**（无需注入网页）
- 书签同步通过 `chrome.bookmarks` API 完成（浏览器原生 API）
- popup 和 options 页面运行在独立浏览器上下文，无样式冲突风险

**如果未来需要 content script（样式隔离方案）：**
```tsx
// 方案 1：Shadow DOM（完全隔离，推荐）
const shadow = document.createElement('div');
shadow.attachShadow({ mode: 'open' });

// 方案 2：Tailwind 前缀配置
// tailwind.config.js: { prefix: 'lzc-' }
// 所有类名自动添加前缀，如 `lzc-btn`、`lzc-input`

// 方案 3：CSS Modules
// 编译后类名自动哈希，如 `btn___a1b2c`
```

### 4.2 Web 管理端技术栈（v2 新增）

| 组件 | 选型 | 版本 | 理由 |
|------|------|------|------|
| **包管理器** | [pnpm](https://pnpm.io/) | v9+ | 与插件统一工具链、快速、节省磁盘空间 |
| **框架构建** | [Vite](https://vitejs.dev/) | v5+ | 与插件统一工具链、快速 HMR |
| **框架** | [React](https://react.dev/) | v18+ | 生态丰富、与 Preact 兼容 |
| **UI 组件库** | [shadcn/ui](https://ui.shadcn.com/) | latest | 基于 Radix UI、可定制、与 Tailwind 完美集成 |
| **状态管理** | [Zustand](https://zustand-demo.pmnd.rs/) | v4+ | 与插件共享状态管理方案 |
| **HTTP 客户端** | [Ky](https://github.com/sindresorhus/ky) | v1+ | 与插件共享 API 客户端 |
| **样式** | [Tailwind CSS](https://tailwindcss.com/) | v3+ | 与插件统一样式方案、shadcn/ui 基础 |
| **图标** | [Lucide Icons](https://lucide.dev/) | - | 与插件统一图标风格、shadcn/ui 默认图标库 |
| **路由** | [React Router](https://reactrouter.com/) | v6+ | 标准路由方案 |

**shadcn/ui 说明：**
- 基于 Radix UI 无头组件，提供可访问性支持
- 使用 Tailwind CSS 进行样式定制
- 组件代码直接复制到项目中，完全可控
- 支持主题定制（暗色模式、颜色方案等）
- 默认包含常用组件：Button、Input、Dialog、Table、Form 等

**共享代码策略：**
- 创建 `shared/` 目录存放插件与 Web 端共享的代码
- 共享内容：TypeScript 类型定义、API 端点定义、工具函数、常量（错误码等）
- 通过 Vite 的 `sharedConfig` 配置共享代码

### 4.3 插件结构

```
lzc-bookmark-extension/
├── public/
│   ├── manifest.json       # Manifest V3
│   ├── icons/
│   └── _locales/           # i18n
├── src/
│   ├── background/
│   │   └── service-worker.ts   # Service Worker（事件监听、定时同步）
│   ├── popup/
│   │   ├── Popup.tsx           # 弹出面板
│   │   └── components/
│   ├── options/
│   │   ├── Options.tsx         # 设置页面
│   │   └── components/
│   ├── shared/
│   │   ├── types.ts            # TypeScript 类型定义
│   │   ├── constants.ts        # 常量（API 端点、错误码等）
│   │   └── utils.ts            # 工具函数
│   ├── sync/
│   │   ├── SyncEngine.ts       # 同步引擎（核心逻辑）
│   │   ├── Queue.ts            # 离线队列管理
│   │   ├── Mapping.ts          # GUID 映射表管理
│   │   └── ConflictResolver.ts # 冲突解决 UI
│   └── api/
│       ├── client.ts           # API 客户端（Ky 封装）
│       └── endpoints.ts        # API 端点定义
├── package.json
├── tsconfig.json
├── vite.config.ts
└── chromium.json               # CRXJS 配置
```

### 4.4 Manifest V3 配置（v2 修正）

```json
{
  "manifest_version": 3,
  "name": "懒猫书签同步",
  "version": "1.0.0",
  "description": "基于懒猫微服的 Chrome 书签同步工具",
  "default_locale": "zh_CN",
  
  "action": {
    "default_popup": "popup/index.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "32": "icons/icon32.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  },
  
  "options_page": "options/index.html",
  
  "background": {
    "service_worker": "background/service-worker.js",
    "type": "module"
  },
  
  "permissions": [
    "bookmarks",
    "storage",
    "alarms",
    "downloads"
  ],
  
  "host_permissions": [
    "https://api.example.com/*"
  ],
  
  "optional_host_permissions": [
    "<all_urls>"
  ],
  
  "web_accessible_resources": []
}
```

**v2 变更说明：**
- `host_permissions` 改为具体域名（用户配置后通过 `chrome.permissions.request()` 动态申请）
- 新增 `optional_host_permissions` 支持用户配置任意服务器地址
- `downloads` 权限用于本地备份（JSON 导出到 chrome.downloads）

### 4.5 存储配额管理

```typescript
// src/sync/StorageManager.ts

async function checkStorageQuota(): Promise<{ used: number; quota: number; percent: number }> {
  const bytesInUse = await chrome.storage.local.getBytesInUse();
  const QUOTA = 5 * 1024 * 1024; // 5MB
  return {
    used: bytesInUse,
    quota: QUOTA,
    percent: (bytesInUse / QUOTA) * 100
  };
}

// 预警阈值
const WARNING_THRESHOLD = 0.8; // 80%

// 定期检查
async function monitorStorage(): Promise<void> {
  const { percent } = await checkStorageQuota();
  if (percent > WARNING_THRESHOLD) {
    // 触发 UI 警告（橙色警告图标）
    await showStorageWarning(percent);
  }
}
```

---

## 5. 关键算法实现

### 5.1 冲突检测算法（v2 修正）

```go
// pkg/conflict/detector.go

type ConflictDetector struct {
    db *gorm.DB
}

// ConflictResult 检测结果
type ConflictResult struct {
    HasConflict bool
    Conflicts   []Conflict
}

type Conflict struct {
    ConflictID    string
    GUID          string
    Type          string  // update_conflict / delete_conflict / move_conflict / rename_conflict / create_conflict
    LocalChange   Change
    RemoteChange  Change
}

// Detect 检测冲突（v2 修正：使用 baseVersion 而非 change.FromVersion）
// 规则：base_version < 该 GUID 最近被修改的 to_version
func (d *ConflictDetector) Detect(deviceID string, baseVersion int64, changes []Change) (*ConflictResult, error) {
    result := &ConflictResult{
        HasConflict: false,
        Conflicts:   []Conflict{},
    }
    
    // v2 优化：批量查询替代 N+1 查询
    // 收集所有 GUID
    guids := make([]string, 0, len(changes))
    for _, change := range changes {
        guids = append(guids, change.GUID)
    }
    
    // 一次性查询所有 GUID 的最新变更
    var latestChanges []Change
    err := d.db.Where("guid IN ? AND to_version > ?", guids, baseVersion).
        Order("to_version DESC").
        Find(&latestChanges).Error
    if err != nil {
        return nil, err
    }
    
    // 构建 guid -> latestChange 映射
    latestMap := make(map[string]Change)
    for _, lc := range latestChanges {
        if _, exists := latestMap[lc.GUID]; !exists {
            latestMap[lc.GUID] = lc
        }
    }
    
    // 检测冲突
    for _, change := range changes {
        latestChange, exists := latestMap[change.GUID]
        if !exists {
            // 该 GUID 没有被修改过，无冲突
            continue
        }
        
        // 检测冲突：base_version < latestChange.ToVersion
        if baseVersion < latestChange.ToVersion {
            result.HasConflict = true
            result.Conflicts = append(result.Conflicts, Conflict{
                ConflictID:   uuid.New().String(),
                GUID:         change.GUID,
                Type:         d.classifyConflict(change, latestChange),
                LocalChange:  change,
                RemoteChange: latestChange,
            })
        }
    }
    
    return result, nil
}

func (d *ConflictDetector) classifyConflict(local, remote Change) string {
    if local.Type == "delete" && remote.Type != "delete" {
        return "delete_conflict"
    }
    if local.Type == "move" && remote.Type == "move" && local.Data.ParentGUID != remote.Data.ParentGUID {
        return "move_conflict"
    }
    if local.Type == "update" && remote.Type == "update" && local.Data.Title != remote.Data.Title {
        return "rename_conflict"
    }
    return "update_conflict"
}

// ChangeData Change 的 data 字段结构（v2 新增）
type ChangeData struct {
    Title       string  `json:"title,omitempty"`
    URL         string  `json:"url,omitempty"`
    ParentGUID  string  `json:"parent_guid,omitempty"`
    IndexWeight float64 `json:"index_weight,omitempty"`
}
```

### 5.2 版本号管理（v2 修正）

```go
// pkg/version/manager.go

type VersionManager struct {
    db *gorm.DB
    mu sync.Mutex  // 全局写锁（回滚时用，单实例部署）
}

// GlobalState 全局状态
type GlobalState struct {
    ID              int64 `gorm:"primaryKey"`
    CurrentVersion  int64
    BaselineVersion int64
    LastTruncatedAt int64
}

// NextVersion 获取下一个版本号（v2 修正：原子操作）
// 使用 UPDATE ... RETURNING 确保原子性
func (m *VersionManager) NextVersion() (int64, error) {
    var state GlobalState
    
    // SQLite 3.35+ / PostgreSQL: 使用 RETURNING 子句原子返回新值
    err := m.db.Raw(`
        UPDATE global_state 
        SET current_version = current_version + 1 
        WHERE id = 1 
        RETURNING current_version
    `).Scan(&state.CurrentVersion).Error
    
    return state.CurrentVersion, err
}

// NextVersionSQLite SQLite 专用版本（如不支持 RETURNING）
func (m *VersionManager) NextVersionSQLite() (int64, error) {
    var newVersion int64
    
    // 使用事务确保原子性
    err := m.db.Transaction(func(tx *gorm.DB) error {
        // 锁定单行（SQLite 通过写锁串行化）
        err := tx.Raw(`SELECT current_version FROM global_state WHERE id = 1`).Scan(&newVersion).Error
        if err != nil {
            return err
        }
        
        newVersion++
        
        err = tx.Exec(`UPDATE global_state SET current_version = ? WHERE id = 1`, newVersion).Error
        return err
    })
    
    return newVersion, err
}

// AcquireLock 获取全局写锁（v2 修正：使用 TryLock + 轮询，避免 goroutine 泄漏）
func (m *VersionManager) AcquireLock(ctx context.Context) error {
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()
    
    deadline, hasDeadline := ctx.Deadline()
    if !hasDeadline {
        deadline = time.Now().Add(5 * time.Second)
    }
    
    for time.Now().Before(deadline) {
        if m.mu.TryLock() {
            return nil
        }
        
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            // 继续尝试
        }
    }
    
    return errors.New("lock timeout")
}

// ReleaseLock 释放全局写锁
func (m *VersionManager) ReleaseLock() {
    m.mu.Unlock()
}

// 注意：sync.Mutex 仅在单进程内有效。
// 多实例部署需改用数据库级锁（PostgreSQL advisory lock）或分布式锁。
```

### 5.3 排序权重归一化（v2 修正）

```go
// pkg/sync/weight.go

const (
    WeightInterval    = 1.0
    MinWeightGap      = 1e-10
    NormalizeThrottle = 5 * time.Minute
)

// NormalizeWeights 归一化同一父节点下的权重（v2 修正：使用 guid 列、添加事务）
func NormalizeWeights(parentGUID string, db *gorm.DB) (*Change, error) {
    // 检查节流（v2 修正：使用 guid 列而非 JSON 提取）
    var lastNormalize Change
    err := db.Where("type = 'reindex' AND guid = ?", parentGUID).
        Order("to_version DESC").
        First(&lastNormalize).Error
    
    if err == nil && time.Since(time.UnixMilli(lastNormalize.Timestamp)) < NormalizeThrottle {
        return nil, nil  // 节流中，跳过
    }
    
    // v2 修正：使用事务包裹整个归一化操作
    var change *Change
    err = db.Transaction(func(tx *gorm.DB) error {
        // 获取所有子节点
        var children []Bookmark
        err := tx.Where("parent_guid = ?", parentGUID).
            Order("index_weight ASC").
            Find(&children).Error
        if err != nil {
            return err
        }
        
        // 重新分配权重
        childrenWeights := make(map[string]float64)
        for i, child := range children {
            newWeight := float64(i+1) * WeightInterval
            childrenWeights[child.GUID] = newWeight
            
            // 更新数据库
            err := tx.Model(&child).Update("index_weight", newWeight).Error
            if err != nil {
                return err
            }
        }
        
        // 生成 reindex 变更
        change = &Change{
            ChangeID: uuid.New().String(),
            Type:     "reindex",
            GUID:     parentGUID,
            DeviceID: "server",
            Data: map[string]interface{}{
                "children_weights": childrenWeights,
            },
            FromVersion: 0,  // 由 VersionManager 填充
            ToVersion:   0,
            Timestamp:   time.Now().UnixMilli(),
        }
        
        // 插入 change_log（版本号由调用者填充）
        // err = tx.Create(change).Error
        // return err
        
        return nil
    })
    
    if err != nil {
        return nil, err
    }
    
    return change, nil
}

// InsertWithWeight 插入新节点时计算权重
func InsertWithWeight(parentGUID string, index int, db *gorm.DB) (float64, error) {
    var siblings []Bookmark
    err := db.Where("parent_guid = ?", parentGUID).
        Order("index_weight ASC").
        Find(&siblings).Error
    if err != nil {
        return 0, err
    }
    
    if len(siblings) == 0 {
        return WeightInterval, nil
    }
    
    if index <= 0 {
        return siblings[0].IndexWeight - WeightInterval, nil
    }
    
    if index >= len(siblings) {
        return siblings[len(siblings)-1].IndexWeight + WeightInterval, nil
    }
    
    // 中间插入
    prev := siblings[index-1]
    next := siblings[index]
    return (prev.IndexWeight + next.IndexWeight) / 2, nil
}
```

### 5.4 离线队列合并（v2 修正）

```typescript
// src/sync/Queue.ts

interface PendingChange {
  change_id: string;
  type: 'create' | 'update' | 'delete' | 'move';
  guid: string | null;
  local_id: string;
  parent_local_id: string | null;
  data: Record<string, unknown>;
  created_at: number;
  retry_count: number;
  submitted: boolean;  // 是否已提交过
}

class Queue {
  private changes: PendingChange[] = [];
  private readonly MAX_SIZE = 5000;

  async add(change: PendingChange): Promise<void> {
    await this.load();
    
    // v2 修正：当 guid 为 null 时，使用 local_id 作为合并键
    const key = change.guid ?? change.local_id;
    
    // 检查是否已存在同一 guid/local_id 的未提交变更
    const existingIndex = this.changes.findIndex(
      c => !c.submitted && (c.guid ?? c.local_id) === key
    );
    
    if (existingIndex !== -1) {
      // 合并：保留最后一次
      this.changes[existingIndex] = change;
    } else {
      this.changes.push(change);
    }
    
    // 检查上限
    if (this.changes.length > this.MAX_SIZE) {
      await this.mergeExcess();
    }
    
    await this.save();
  }

  private async mergeExcess(): Promise<void> {
    // v2 修正：仅合并 submitted=false 的变更
    const unsubmitted = this.changes.filter(c => !c.submitted);
    const submitted = this.changes.filter(c => c.submitted);
    
    if (unsubmitted.length <= this.MAX_SIZE) {
      return;  // 不需要合并
    }
    
    // 按 guid/local_id 分组，保留每组最后一个
    const merged = new Map<string, PendingChange>();
    for (const change of unsubmitted) {
      const key = change.guid ?? change.local_id;
      merged.set(key, change);
    }
    
    this.changes = [...submitted, ...merged.values()];
    await this.save();
  }

  async markSubmitted(changeIds: string[]): Promise<void> {
    await this.load();
    for (const change of this.changes) {
      if (changeIds.includes(change.change_id)) {
        change.submitted = true;
      }
    }
    await this.save();
  }

  async getUnsubmitted(): Promise<PendingChange[]> {
    await this.load();
    return this.changes.filter(c => !c.submitted);
  }
}
```

---

## 6. 部署方案

### 6.1 Docker 配置（v2 修正）

```dockerfile
# Dockerfile
# 阶段 1：构建 Web 前端
FROM node:20-alpine AS web-builder

# v2.1 修正：安装 pnpm
RUN corepack enable && corepack prepare pnpm@latest --activate

WORKDIR /build/web
COPY web/package*.json ./
# v2.1 修正：使用 pnpm 替代 npm
RUN pnpm install --frozen-lockfile
COPY web/ ./
RUN pnpm run build

# 阶段 2：构建 Go 后端
FROM golang:1.23-alpine AS go-builder

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download

COPY . .
# v2 修正：使用 CGO_ENABLED=0（modernc.org/sqlite 无需 CGO）
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

# 阶段 3：运行时镜像
FROM alpine:3.19

# v2 修正：安装 wget 用于健康检查
RUN apk --no-cache add ca-certificates wget

WORKDIR /app
COPY --from=go-builder /build/server .
COPY --from=go-builder /build/migrations ./migrations
COPY --from=web-builder /build/web/dist ./web/dist

EXPOSE 8080

# v2 修正：wget 已安装，健康检查可用
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/health || exit 1

CMD ["./server"]
```

### 6.2 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  lzc-bookmark:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DB_TYPE=sqlite
      - DB_PATH=/data/lzc-bookmark.db
      - ALLOW_INSECURE_HTTP=false
      - LOG_LEVEL=info
    volumes:
      - lzc-data:/data
    restart: unless-stopped

volumes:
  lzc-data:
```

### 6.3 环境变量配置（v2 修正）

```bash
# .env.example

# 数据库配置
DB_TYPE=sqlite              # sqlite 或 postgres
DB_PATH=/data/lzc-bookmark.db  # SQLite 路径
# PostgreSQL 配置（DB_TYPE=postgres 时使用）
# DB_HOST=localhost
# DB_PORT=5432
# DB_NAME=lzc_bookmark
# DB_USER=lzc
# DB_PASSWORD=secret

# 安全配置
ALLOW_INSECURE_HTTP=false   # 是否允许 HTTP（内网测试用）
# v2 修正：删除 JWT_SECRET（PRD 只用 API Key）

# 日志配置
LOG_LEVEL=info              # debug/info/warn/error
LOG_FORMAT=json             # json 或 console

# 变更日志配置
CHANGE_LOG_MAX_COUNT=1000   # 最大条数（仅统计 create/update/delete/move）
CHANGE_LOG_MAX_AGE_DAYS=7   # 最大年龄（天）
# 注意：reindex 和 rollback 不计入 1000 条上限，但仍受 7 天年龄限制

# 快照配置
SNAPSHOT_MAX_COUNT=500      # 最大快照数
SNAPSHOT_MAX_AGE_DAYS=30    # 最大年龄（天）

# 限流配置
RATE_LIMIT_SYNC=60          # /sync 每分钟次数
RATE_LIMIT_OTHER=120        # 其他接口每分钟次数
RATE_LIMIT_KEY_TOTAL=200    # 单 API Key 总限流

# 同步配置
SYNC_INTERVAL_SECONDS=30    # 自动同步间隔（默认 30 秒）
SOFT_DELETE_DAYS=90         # 软删除后物理删除天数
MAX_BOOKMARKS=100000        # 单用户书签总数上限
MAX_SYNC_CHANGES=500        # 单次 sync 变更数上限
```

### 6.4 懒猫微服部署

```yaml
# lazycat manifest.yml
apiVersion: lazycat.io/v1
kind: Application
metadata:
  name: lzc-bookmark
  version: 1.0.0
spec:
  image: registry.cn-hangzhou.aliyuncs.com/lazycat/lzc-bookmark:1.0.0
  port: 8080
  env:
    - name: DB_TYPE
      value: "sqlite"
    - name: DB_PATH
      value: "/data/lzc-bookmark.db"
    - name: ALLOW_INSECURE_HTTP
      value: "false"
  volumes:
    - name: data
      mountPath: /data
      type: hostPath
      hostPath: /data/lzc-bookmark
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  healthCheck:
    path: /health
    interval: 30s
    timeout: 5s
```

---

## 7. 测试策略

### 7.1 服务端测试

```go
// internal/handler/sync_test.go

func TestSyncHandler_PartialConflict(t *testing.T) {
    // 准备数据
    db := setupTestDB(t)
    seedBookmarks(db, 10)  // 10 个书签
    seedChangeLog(db, 120) // 当前版本 120
    
    // 构造请求：10 条变更，其中 2 条会冲突
    changes := makeChanges(10)
    // v2 修正：客户端发送 base_version，单条变更没有 from_version
    baseVersion := int64(115)  // 落后版本，会冲突
    
    req := SyncRequest{
        DeviceID:      "test-device",
        BaseVersion:   baseVersion,
        ClientRequestID: uuid.New().String(),
        Changes:       changes,
    }
    
    // 执行
    handler := NewSyncHandler(db)
    resp := handler.Sync(req)
    
    // 断言
    assert.Equal(t, int64(128), resp.NewVersion)  // 8 条应用，120+8=128
    assert.Len(t, resp.AppliedChanges, 8)
    assert.Len(t, resp.Conflicts, 2)
    assert.Equal(t, int64(130), resp.Version)  // 全局最新版本
}

func TestConflictDetector_RetryAfterPartialConflict(t *testing.T) {
    // 测试部分冲突重试场景
    // 设备 A 推送 10 条，8 条应用，2 条冲突
    // 设备 A 更新 base_version=128 后重试，应该不冲突
}
```

### 7.2 插件测试

```typescript
// src/sync/SyncEngine.test.ts

describe('SyncEngine', () => {
  describe('partialConflict', () => {
    it('should update last_known_version after partial conflict', async () => {
      // 构造部分冲突响应
      const response = {
        success: true,
        data: {
          new_version: 128,
          applied_changes: ['ch_001', ..., 'ch_008'],
          conflicts: [{ conflict_id: 'cf_001' }, { conflict_id: 'cf_002' }],
        },
        version: 130,
      };
      
      const engine = new SyncEngine();
      await engine.handleSyncResponse(response);
      
      // 断言 last_known_version 更新为 128
      const state = await engine.getState();
      expect(state.lastKnownVersion).toBe(128);
      
      // 断言冲突变更的 base_version 更新
      const pendingChanges = await engine.queue.getUnsubmitted();
      expect(pendingChanges[0].base_version).toBe(128);
    });
  });

  describe('queueMerge', () => {
    it('should only merge unsubmitted changes', async () => {
      const queue = new Queue();
      
      // 添加已提交和未提交的变更
      await queue.add({ ...change1, submitted: true });
      await queue.add({ ...change2, submitted: false });
      await queue.add({ ...change3, submitted: false });
      
      // 触发合并
      await queue.mergeExcess();
      
      // 断言：已提交的变更不会被合并
      const all = await queue.getAll();
      expect(all.find(c => c.change_id === change1.change_id)).toBeDefined();
    });
    
    it('should use local_id as key when guid is null (create operation)', async () => {
      const queue = new Queue();
      
      // 添加两个 create 操作（guid 为 null）
      await queue.add({ ...create1, guid: null, local_id: 'local_1' });
      await queue.add({ ...create2, guid: null, local_id: 'local_2' });
      
      // 断言：两个 create 操作不会被错误合并
      const all = await queue.getAll();
      expect(all.length).toBe(2);
    });
  });
});
```

### 7.3 集成测试

```bash
# tests/integration/run.sh

# 1. 启动服务端
docker-compose up -d

# 2. 运行服务端测试
go test ./... -tags=integration

# 3. 构建插件
cd extension && npm run build

# 4. 运行插件 E2E 测试（使用 Playwright）
npx playwright test tests/e2e/

# 5. 清理
docker-compose down
```

---

## 8. 监控与告警（v2 修正）

### 8.1 指标暴露（v2 修正：移除 device_id label）

```go
// internal/metrics/metrics.go

var (
    syncRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "lzc_sync_requests_total",
            Help: "Total number of sync requests",
        },
        []string{"status"},  // v2 修正：移除 device_id，仅保留 status
    )
    
    syncDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "lzc_sync_duration_seconds",
            Help:    "Sync request duration",
            Buckets: prometheus.ExponentialBuckets(0.1, 1.5, 10),
        },
        []string{},  // v2 修正：移除 device_id
    )
    
    conflictCount = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "lzc_conflicts_total",
            Help: "Total number of conflicts detected",
        },
        []string{"type"},  // type: update/delete/move/rename/create
    )
    
    changeLogSize = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "lzc_change_log_size",
            Help: "Current change log size",
        },
    )
)
```

### 8.2 告警规则（v2 修正）

```yaml
# prometheus/alerts.yml

groups:
  - name: lzc-bookmark
    rules:
      - alert: HighConflictRate
        expr: rate(lzc_conflicts_total[5m]) > 0.167  # v2 修正：10 次/分钟 = 0.167 次/秒
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
          rate(lzc_sync_requests_total{status="error"}[5m]) 
          / (rate(lzc_sync_requests_total[5m]) > 0)  # v2 修正：避免除以 0
        > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "同步错误率高"
          description: "同步错误率超过 10%"
```

---

## 9. 开发里程碑

### Phase 1: 懒猫微服核心（5 天）

| 天 | 任务 | 交付物 |
|----|------|--------|
| 1 | 项目初始化、数据库 schema、GORM 模型 | 可运行的空项目 |
| 2 | API Key 鉴权、设备管理 API | /auth/verify, /devices |
| 3 | 书签 CRUD API、GUID 生成 | /bookmarks |
| 4 | 同步 API（/sync）、版本号管理 | 可接收同步请求 |
| 5 | 冲突检测、Web 管理界面框架 | 可检测并返回冲突 |

### Phase 2: Chrome 插件基础（4 天）

| 天 | 任务 | 交付物 |
|----|------|--------|
| 1 | 项目初始化、Manifest V3 配置 | 可安装的插件 |
| 2 | 书签读取、GUID 映射表管理 | 可读取本地书签 |
| 3 | API 配置界面、手动同步 | 可配置并手动同步 |
| 4 | 离线队列、事件监听 | 可累积变更 |

### Phase 3: 同步逻辑（5 天）

| 天 | 任务 | 交付物 |
|----|------|--------|
| 1 | 增量同步算法、先推后拉 | 可增量同步 |
| 2 | 冲突检测 UI、冲突解决 | 可处理冲突 |
| 3 | 定时同步、chrome.alarms | 自动同步 |
| 4 | Service Worker 降级策略 | 休眠唤醒正常 |
| 5 | 部分冲突处理、状态更新 | 部分冲突正常 |

### Phase 4: 完善与测试（4 天）

| 天 | 任务 | 交付物 |
|----|------|--------|
| 1 | Web 管理界面完善、历史记录 | 可浏览历史 |
| 2 | 设备管理、回滚功能 | 可撤销设备、回滚 |
| 3 | 多设备测试、边界测试 | 测试报告 |
| 4 | 文档编写、自动化测试 | 用户指南、API 文档 |

### Phase 5: 测试与修复（2 天）

| 天 | 任务 | 交付物 |
|----|------|--------|
| 1 | Bug 修复、性能优化 | 修复列表 |
| 2 | 验收测试、发布准备 | 发布包 |

---

## 10. 风险与缓解

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| Chrome 三根映射兼容性问题 | 新设备拉取失败 | 中 | 提前测试多版本 Chrome、Edge、Brave |
| 归一化触发频繁 | change_log 膨胀 | 低 | 5 分钟节流、不计入上限 |
| 部分冲突重试逻辑错误 | 冲突变更无法应用 | 中 | 充分测试重试场景、添加集成测试 |
| storage.local 配额超限 | 离线队列丢失 | 中 | 80% 预警、合并策略 |
| SQLite 并发写锁 | 高并发下性能下降 | 低 | 个人用户并发低、可切换 PostgreSQL |
| Service Worker 休眠导致事件丢失 | 本地变更未及时同步 | 中 | 唤醒时 getTree diff、定时同步 |
| sync.Mutex 多实例失效 | 回滚操作并发问题 | 低 | v1 单实例部署，v2 改用数据库锁 |

---

## 11. 参考文档

- [PRD v6.0](./PRD-v6.md)
- [LPK 框架文档](https://github.com/lazycat/lpk-docs)
- [Chrome Bookmarks API](https://developer.chrome.com/docs/extensions/reference/bookmarks/)
- [Manifest V3](https://developer.chrome.com/docs/extensions/mv3/intro/)
- [GORM 文档](https://gorm.io/docs/)
- [Gin 文档](https://gin-gonic.com/docs/)
- [modernc.org/sqlite](https://pkg.go.dev/modernc.org/sqlite)
- [pgx 文档](https://github.com/jackc/pgx)

---

## 12. 修订历史

| 版本 | 日期 | 修订内容 |
|------|------|----------|
| v1.0 | 2026-04-16 | 初始版本 |
| v2.0 | 2026-04-16 | 修复 4 份评审报告提出的全部 P0/P1 问题 |
| v2.1 | 2026-04-16 | 包管理器改为 pnpm、Web 管理端 UI 使用 shadcn/ui |

**v2.0 关键变更：**
1. 修复 CGO_ENABLED=0 与 SQLite 矛盾（改用 modernc.org/sqlite）
2. 修复 NextVersion 竞态条件（使用 UPDATE ... RETURNING）
3. 修复 AcquireLock goroutine 泄漏（改用 TryLock + 轮询）
4. 修复冲突检测算法（使用 baseVersion 而非 change.FromVersion）
5. 修复 NormalizeWeights 查询语法（使用 guid 列而非 JSON 提取）
6. 修复 conflicts 表设计（存储完整 Change JSON 而非外键）
7. 新增 sync_records 表
8. 新增 Web 管理端技术栈（React + Vite）
9. 删除 JWT_SECRET 环境变量
10. 修复 host_permissions（改用 optional_host_permissions）
11. 修复 Docker HEALTHCHECK（添加 wget 安装）
12. 修复 Prometheus 指标（移除 device_id label）
13. 新增归一化事务包裹
14. 优化冲突检测 N+1 查询问题
15. 修复离线队列 create 合并 bug
16. 升级 Go 版本至 1.23
17. 删除 bookmarks 表多余字段（created_by/modified_by）
18. 新增 Dockerfile 前端构建步骤
19. 修复告警 PromQL 表达式

**v2.1 关键变更：**
1. 包管理器统一为 pnpm（Chrome 插件和 Web 管理端）
2. Web 管理端 UI 使用 shadcn/ui（基于 Radix UI + Tailwind CSS）
3. **Chrome 插件 UI 改为手写简单组件**（而非 shadcn/ui，更轻量）
4. 明确**不使用 content script**（无需注入网页）
5. Dockerfile 前端构建改为 pnpm（使用 corepack 安装）

---

**文档状态:** 已评审，待开发  
**下一步:** Phase 1 开发启动