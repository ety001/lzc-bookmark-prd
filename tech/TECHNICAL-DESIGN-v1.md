# 懒猫书签同步系统 - 技术方案选型

**基于 PRD v6.0**  
**创建时间:** 2026-04-16  
**状态:** 待评审

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
| **Web 框架** | [Gin](https://github.com/gin-gonic/gin) | v1.9+ | 高性能、中间件生态丰富、LPK 默认支持 |
| **ORM** | [GORM v2](https://gorm.io/) | v2.5+ | 支持 SQLite/PostgreSQL 双后端、关联查询方便 |
| **迁移工具** | [golang-migrate](https://github.com/golang-migrate/migrate) | v4.16+ | 纯 Go 实现、支持多数据库、版本管理清晰 |
| **配置管理** | [Viper](https://github.com/spf13/viper) | v1.18+ | 支持环境变量、YAML、热重载 |
| **日志** | [Zap](https://github.com/uber-go/zap) | v1.26+ | 高性能结构化日志、支持日志轮转 |
| **验证** | [go-playground/validator](https://github.com/go-playground/validator) | v10.15+ | 标签式验证、错误信息友好 |
| **UUID** | [google/uuid](https://github.com/google/uuid) | v1.4+ | 标准库、RFC 4122  compliant |
| **密码哈希** | [bcrypt](https://golang.org/x/crypto/bcrypt) | stdlib | Go 标准库、安全可靠 |
| **压缩** | [gzip](https://pkg.go.dev/compress/gzip) | stdlib | Go 标准库、快照存储压缩 |
| **锁** | [sync.Mutex](https://pkg.go.dev/sync#Mutex) | stdlib | 回滚操作用全局写锁 |

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
├── migrations/               # 数据库迁移脚本
├── web/                      # React 前端
├── Dockerfile
├── go.mod
└── config.yaml.example
```

**LPK 提供的开箱即用能力：**
- Docker 构建与部署
- 环境变量配置
- 健康检查端点
- 日志收集
- 指标暴露（Prometheus）

---

## 3. 数据库设计

### 3.1 Schema 总览

```sql
-- SQLite schema (PostgreSQL 兼容)

-- API Keys 表
CREATE TABLE api_keys (
    id              TEXT PRIMARY KEY,     -- UUID v4
    name            TEXT NOT NULL,        -- 备注名称（如"工作电脑"）
    key_hash        TEXT NOT NULL,        -- bcrypt 哈希
    permissions     TEXT NOT NULL,        -- JSON: ["read"] 或 ["read", "write"]
    created_at      INTEGER NOT NULL,     -- Unix 毫秒
    last_used_at    INTEGER,              -- Unix 毫秒
    is_revoked      INTEGER NOT NULL DEFAULT 0  -- 0=false, 1=true
);

-- Devices 表
CREATE TABLE devices (
    device_id       TEXT PRIMARY KEY,     -- UUID v4
    name            TEXT NOT NULL,        -- 设备名称
    api_key_id      TEXT NOT NULL,        -- 外键 → api_keys.id
    created_at      INTEGER NOT NULL,
    last_seen_at    INTEGER NOT NULL,
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
    date_created    INTEGER NOT NULL,     -- Unix 毫秒
    date_modified   INTEGER NOT NULL,     -- Unix 毫秒
    is_deleted      INTEGER NOT NULL DEFAULT 0,
    deleted_at      INTEGER,              -- Unix 毫秒
    created_by      TEXT,                 -- device_id（可选）
    modified_by     TEXT,                 -- device_id（可选）
    FOREIGN KEY (parent_guid) REFERENCES bookmarks(guid)
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
    from_version    INTEGER NOT NULL,
    to_version      INTEGER NOT NULL UNIQUE,  -- 全局唯一，每条 +1
    timestamp       INTEGER NOT NULL,     -- Unix 毫秒（服务端接收时间）
    client_request_id TEXT,               -- 幂等 ID（可选）
    CHECK (type IN ('create', 'update', 'delete', 'move', 'reindex', 'rollback'))
);

-- 索引
CREATE INDEX idx_change_log_guid ON change_log(guid);
CREATE INDEX idx_change_log_to_version ON change_log(to_version);
CREATE INDEX idx_change_log_timestamp ON change_log(timestamp);
CREATE INDEX idx_change_log_type ON change_log(type);

-- 全局状态表（单行）
CREATE TABLE global_state (
    id              INTEGER PRIMARY KEY CHECK (id = 1),
    current_version INTEGER NOT NULL DEFAULT 0,
    baseline_version INTEGER NOT NULL DEFAULT 0,
    last_truncated_at INTEGER
);

-- 初始化
INSERT INTO global_state (id, current_version, baseline_version) VALUES (1, 0, 0);

-- Snapshots 表
CREATE TABLE snapshots (
    version         INTEGER PRIMARY KEY,
    bookmark_tree   TEXT NOT NULL,        -- JSON（gzip 压缩后 base64）
    created_at      INTEGER NOT NULL,
    is_baseline     INTEGER NOT NULL DEFAULT 0
);

-- Conflicts 表
CREATE TABLE conflicts (
    conflict_id     TEXT PRIMARY KEY,     -- UUID v4
    guid            TEXT NOT NULL,
    local_change_id TEXT NOT NULL,        -- change_log.change_id
    remote_change_id TEXT NOT NULL,       -- change_log.change_id
    status          TEXT NOT NULL,        -- pending/resolved
    resolution      TEXT,                 -- use_local/use_remote/merge
    created_at      INTEGER NOT NULL,
    resolved_at     INTEGER,
    resolved_by     TEXT,                 -- device_id
    FOREIGN KEY (local_change_id) REFERENCES change_log(change_id),
    FOREIGN KEY (remote_change_id) REFERENCES change_log(change_id)
);

-- 索引
CREATE INDEX idx_conflicts_status ON conflicts(status);
CREATE INDEX idx_conflicts_guid ON conflicts(guid);

-- Rate Limits 表（内存替代方案：使用 go-cache）
-- 实际实现建议使用内存缓存，不持久化
```

### 3.2 PostgreSQL 兼容

```sql
-- PostgreSQL 需要调整的类型：
-- INTEGER → BIGINT（Unix 毫秒可能超出 INTEGER 范围）
-- TEXT → TEXT（兼容）
-- REAL → DOUBLE PRECISION
-- 自增主键 → 使用 UUID v4（与 SQLite 一致）

-- PostgreSQL 额外优化：
-- 使用 JSONB 替代 TEXT 存储 JSON 字段
-- 使用 PARTITION BY RANGE 对 change_log 按 to_version 分区
```

### 3.3 数据量估算

| 表 | 单条大小 | 10 万书签 | 备注 |
|----|----------|----------|------|
| bookmarks | ~200B | 20MB | 10 万节点 |
| change_log | ~300B | 300MB | 100 万条变更（10 倍于书签数） |
| snapshots | ~100KB | 50MB | 500 个快照（gzip 压缩后） |
| conflicts | ~500B | ~0 | 临时表，解决后归档 |
| **总计** | - | **~370MB** | 可接受 |

---

## 4. Chrome 插件技术栈

### 4.1 核心技术栈

| 组件 | 选型 | 版本 | 理由 |
|------|------|------|------|
| **构建工具** | [Vite](https://vitejs.dev/) + [CRXJS](https://crxjs.dev/vite) | v5+ | 快速 HMR、Manifest V3 支持好 |
| **框架** | [Preact](https://preactjs.com/) | v10+ | 轻量（3KB）、React 兼容、适合插件 |
| **状态管理** | [Zustand](https://zustand-demo.pmnd.rs/) | v4+ | 轻量、无样板代码 |
| **HTTP 客户端** | [Ky](https://github.com/sindresorhus/ky) | v1+ | 轻量、Promise 友好、自动重试 |
| **存储封装** | [chrome.storage.local](https://developer.chrome.com/docs/extensions/reference/storage/) | native | Manifest V3 推荐 |
| **UUID 生成** | [crypto.randomUUID()](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID) | native | Chrome 102+ 原生支持 |
| **样式** | [Tailwind CSS](https://tailwindcss.com/) | v3+ | 原子化 CSS、体积小 |
| **图标** | [Lucide Icons](https://lucide.dev/) | - | 开源、一致风格 |

### 4.2 插件结构

```
lzc-bookmark-extension/
├── public/
│   ├── manifest.json       # Manifest V3
│   ├── icons/
│   └── _locales/           # i18n
├── src/
│   ├── background/
│   │   └── service-worker.ts   # Service Worker（事件监听、定时同步）
│   ├── content/
│   │   └── content-script.ts   # 内容脚本（可选，用于页面内操作）
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

### 4.3 Manifest V3 配置

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
    "https://*/*",
    "http://*/*"
  ],
  
  "web_accessible_resources": []
}
```

### 4.4 存储配额管理

```typescript
// storage.local 配额监控
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
```

---

## 5. 关键算法实现

### 5.1 冲突检测算法

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
    Type          string  // update_conflict / delete_conflict / move_conflict
    LocalChange   Change
    RemoteChange  Change
}

// Detect 检测冲突
// 规则：base_version < 该 GUID 最近被修改的 to_version
func (d *ConflictDetector) Detect(deviceID string, baseVersion int64, changes []Change) (*ConflictResult, error) {
    result := &ConflictResult{
        HasConflict: false,
        Conflicts:   []Conflict{},
    }
    
    for _, change := range changes {
        // 获取该 GUID 最近被修改的 to_version
        var latestChange Change
        err := d.db.Where("guid = ? AND to_version > ?", change.GUID, change.FromVersion).
            Order("to_version DESC").
            First(&latestChange).Error
        
        if err == gorm.ErrRecordNotFound {
            // 该 GUID 没有被修改过，无冲突
            continue
        }
        if err != nil {
            return nil, err
        }
        
        // 检测冲突：base_version < latestChange.ToVersion
        if change.FromVersion < latestChange.ToVersion {
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
    return "update_conflict"
}
```

### 5.2 版本号管理

```go
// pkg/version/manager.go

type VersionManager struct {
    db *gorm.DB
    mu sync.Mutex  // 全局写锁（回滚时用）
}

// GlobalState 全局状态
type GlobalState struct {
    ID              int64 `gorm:"primaryKey"`
    CurrentVersion  int64
    BaselineVersion int64
    LastTruncatedAt int64
}

// NextVersion 获取下一个版本号（原子操作）
func (m *VersionManager) NextVersion() (int64, error) {
    var state GlobalState
    err := m.db.First(&state, 1).Error
    if err != nil {
        return 0, err
    }
    
    // SQLite: UPDATE ... RETURNING
    // PostgreSQL: 原生支持 RETURNING
    err = m.db.Exec(`
        UPDATE global_state 
        SET current_version = current_version + 1 
        WHERE id = 1
    `).Error
    if err != nil {
        return 0, err
    }
    
    // 读取新值
    err = m.db.First(&state, 1).Error
    return state.CurrentVersion, err
}

// AcquireLock 获取全局写锁（回滚操作）
func (m *VersionManager) AcquireLock(ctx context.Context) error {
    done := make(chan struct{})
    go func() {
        m.mu.Lock()
        close(done)
    }()
    
    select {
    case <-done:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(5 * time.Second):
        return errors.New("lock timeout")
    }
}

// ReleaseLock 释放全局写锁
func (m *VersionManager) ReleaseLock() {
    m.mu.Unlock()
}
```

### 5.3 排序权重归一化

```go
// pkg/sync/weight.go

const (
    WeightInterval  = 1.0
    MinWeightGap    = 1e-10
    NormalizeThrottle = 5 * time.Minute
)

// NormalizeWeights 归一化同一父节点下的权重
func NormalizeWeights(parentGUID string, db *gorm.DB) (*Change, error) {
    // 检查节流
    var lastNormalize Change
    err := db.Where("type = 'reindex' AND data->>'$.guid' = ?", parentGUID).
        Order("to_version DESC").
        First(&lastNormalize).Error
    
    if err == nil && time.Since(time.UnixMilli(lastNormalize.Timestamp)) < NormalizeThrottle {
        return nil, nil  // 节流中，跳过
    }
    
    // 获取所有子节点
    var children []Bookmark
    err = db.Where("parent_guid = ?", parentGUID).
        Order("index_weight ASC").
        Find(&children).Error
    if err != nil {
        return nil, err
    }
    
    // 重新分配权重
    childrenWeights := make(map[string]float64)
    for i, child := range children {
        newWeight := float64(i+1) * WeightInterval
        childrenWeights[child.GUID] = newWeight
        
        // 更新数据库
        db.Model(&child).Update("index_weight", newWeight)
    }
    
    // 生成 reindex 变更
    change := Change{
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
    
    return &change, nil
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

### 5.4 离线队列合并

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
    
    // 检查是否已存在同一 guid 的未提交变更
    const existingIndex = this.changes.findIndex(
      c => c.guid === change.guid && !c.submitted
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
    // 仅合并 submitted=false 的变更
    const unsubmitted = this.changes.filter(c => !c.submitted);
    const submitted = this.changes.filter(c => c.submitted);
    
    if (unsubmitted.length <= this.MAX_SIZE) {
      return;  // 不需要合并
    }
    
    // 按 guid 分组，保留每组最后一个
    const merged = new Map<string, PendingChange>();
    for (const change of unsubmitted) {
      merged.set(change.guid || change.local_id, change);
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

### 6.1 Docker 配置

```dockerfile
# Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

# 运行时镜像
FROM alpine:3.19

RUN apk --no-cache add ca-certificates

WORKDIR /app
COPY --from=builder /build/server .
COPY --from=builder /build/migrations ./migrations

EXPOSE 8080

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

### 6.3 环境变量配置

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
JWT_SECRET=your-secret-key  # JWT 密钥（如果使用）

# 日志配置
LOG_LEVEL=info              # debug/info/warn/error
LOG_FORMAT=json             # json 或 console

# 变更日志配置
CHANGE_LOG_MAX_COUNT=1000   # 最大条数
CHANGE_LOG_MAX_AGE_DAYS=7   # 最大年龄（天）

# 快照配置
SNAPSHOT_MAX_COUNT=500      # 最大快照数
SNAPSHOT_MAX_AGE_DAYS=30    # 最大年龄（天）

# 限流配置
RATE_LIMIT_SYNC=60          # /sync 每分钟次数
RATE_LIMIT_OTHER=120        # 其他接口每分钟次数
RATE_LIMIT_KEY_TOTAL=200    # 单 API Key 总限流
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
    changes[8].FromVersion = 115  // 落后版本，会冲突
    changes[9].FromVersion = 115  // 落后版本，会冲突
    
    req := SyncRequest{
        DeviceID:      "test-device",
        BaseVersion:   115,
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

# 4. 运行插件 E2E 测试（使用 Puppeteer）
npx playwright test tests/e2e/

# 5. 清理
docker-compose down
```

---

## 8. 监控与告警

### 8.1 指标暴露

```go
// internal/metrics/metrics.go

var (
    syncRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "lzc_sync_requests_total",
            Help: "Total number of sync requests",
        },
        []string{"device_id", "status"},  // status: success/conflict/error
    )
    
    syncDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "lzc_sync_duration_seconds",
            Help:    "Sync request duration",
            Buckets: prometheus.ExponentialBuckets(0.1, 1.5, 10),
        },
        []string{"device_id"},
    )
    
    conflictCount = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "lzc_conflicts_total",
            Help: "Total number of conflicts detected",
        },
        []string{"type"},  // type: update/delete/move
    )
    
    changeLogSize = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "lzc_change_log_size",
            Help: "Current change log size",
        },
    )
)
```

### 8.2 告警规则

```yaml
# prometheus/alerts.yml

groups:
  - name: lzc-bookmark
    rules:
      - alert: HighConflictRate
        expr: rate(lzc_conflicts_total[5m]) > 10
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
        expr: rate(lzc_sync_requests_total{status="error"}[5m]) / rate(lzc_sync_requests_total[5m]) > 0.1
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

---

## 11. 参考文档

- [PRD v6.0](./PRD-v6.md)
- [LPK 框架文档](https://github.com/lazycat/lpk-docs)
- [Chrome Bookmarks API](https://developer.chrome.com/docs/extensions/reference/bookmarks/)
- [Manifest V3](https://developer.chrome.com/docs/extensions/mv3/intro/)
- [GORM 文档](https://gorm.io/docs/)
- [Gin 文档](https://gin-gonic.com/docs/)

---

**文档状态:** 待评审  
**下一步:** 技术方案评审 → Phase 1 开发启动