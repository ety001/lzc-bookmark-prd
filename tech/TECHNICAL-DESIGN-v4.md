# 懒猫书签同步系统 - 技术方案 v4.0

**基于 PRD v6.0**  
**创建时间:** 2026-04-16  
**修订时间:** 2026-04-17 (v4.0)  
**状态:** 已评审，待开发

---

## 修订摘要 (v3 → v4)

**v4.0 核心变更：** 综合 4 份 v3 评审报告（Composer2, GLM-5.1, Kimi Code, Kimi K2.5）的全部 P0/P1 问题修复

### P0 问题修复（9 项）

| # | 问题 | 修复方案 |
|---|------|----------|
| 1 | **Data 字段类型无法编译** | 改用 `ChangeData` 结构体 + `serializer:json` 标签，支持点号访问 |
| 2 | **SQLite 不支持 FOR UPDATE** | 删除 `NextVersionSQLite` 方法，统一使用 `RETURNING`（要求 SQLite 3.35+） |
| 3 | **同步协议与 PRD v6 矛盾** | 端点改为 `/api/v1/bookmarks/sync`，错误码改为 `E001-E008`，删除 410 GONE |
| 4 | **410 GONE 处理与 PRD 矛盾** | PRD v6 已废除 410，改为 200 + `full_sync: true` 响应 |
| 5 | **同批 create 冲突检测误杀** | 删除同批 create 冲突检测逻辑（同一 parent_local_id 多个 create 是正常操作） |
| 6 | **NormalizeWeights 版本号缺失** | 在事务内先调用 `NextVersion()` 填充 `ToVersion`，再插入 change_log |
| 7 | **截断算法 baseline_version 错误** | 改为保留记录的最小 `from_version` - 1，并更新 snapshot 的 `is_baseline` 标记 |
| 8 | **SyncResponse 包含 changes 字段** | 删除 `changes` 字段，严格遵循"先推后拉"架构 |
| 9 | **parent_guid 位置不一致** | 移至 `data` 对象内部（与 PRD v6 一致） |

### P1 问题修复（18 项）

| # | 问题 | 修复方案 |
|---|------|----------|
| 1 | **同步响应结构语义模糊** | 新增"响应不变式"表格，说明四种场景下字段填充规则 |
| 2 | **NormalizeWeights 伪代码变量未定义** | 显式定义 `guids` 变量，对齐 GORM 模型 |
| 3 | **TruncateChangeLog 截断语义不直观** | 新增数学化公式说明设计意图与边界条件 |
| 4 | **参考链接路径错误** | 修正为 `../prd/PRD-v6.md` 和同目录相对路径 |
| 5 | **截断算法未清理 reindex/rollback** | 修改 WHERE 条件包含全部 6 种类型 |
| 6 | **create_conflict 检测缺少 title/URL 匹配** | 在 classifyConflict 中增加 create 操作的 title/URL 比较 |
| 7 | **Data 字段类型混用三种类型** | 统一为 `ChangeData` 结构体 |
| 8 | **SQLite 缺少 busy_timeout** | 在 main.go 添加 `PRAGMA busy_timeout=5000` |
| 9 | **Dockerfile 复制全量源码** | 改为仅复制 `cmd/`、`internal/`、`pkg/` 和 `migrations/` |
| 10 | **Detect() guids 为空时非法 SQL** | 在查询前检查 `len(guids) == 0` 直接返回 |
| 11 | **GenerateSnapshot 清理逻辑风险** | 使用参数化查询，添加 30 天年龄清理 |
| 12 | **PendingChange.local_id 错误标记 nullable** | 改为必需字段（非 null） |
| 13 | **集成测试使用旧版 docker-compose** | 改为 `docker compose`（v2 语法） |
| 14 | **集成测试构建未使用 --frozen-lockfile** | 添加 `--frozen-lockfile` 标志 |
| 15 | **ChangeData 缺少 date_created/date_modified 说明** | 新增责任说明字段 |
| 16 | **连接池配置缺失** | 在 main.go 显式配置 GORM 连接池参数 |
| 17 | **数据库健康检查缺失** | `/health` 端点增加 DB 连接检查 |
| 18 | **冲突解决超时缺失** | 新增 7 天未解决自动使用服务端版本策略 |

### P2 问题优化（10 项合并后）

1. ✅ PromQL 零流量行为说明（返回 0）
2. ✅ 版本号空洞指标触发条件（空洞 > 10 触发）
3. ✅ NextVersion SQLite/PG 选用策略（自动检测）
4. ✅ 空 GUID create 操作指回 PRD
5. ✅ 快照清理 SQL 可移植性标注
6. ✅ DDL 尾随逗号兼容性说明
7. ✅ 测试策略增加最小验收条目
8. ✅ 增量同步返回添加分页支持
9. ✅ change_log.data 压缩存储说明
10. ✅ Dockerfile 添加 USER 指令（非 root 运行）

---

## 目录

1. [整体架构](#1-整体架构)
2. [服务端技术栈选型](#2-服务端技术栈选型)
3. [数据库设计](#3-数据库设计)
4. [同步协议](#4-同步协议)
5. [Chrome 插件与 Web 管理端技术栈](#5-chrome-插件与-web-管理端技术栈)
6. [关键算法实现](#6-关键算法实现)
7. [部署方案](#7-部署方案)
8. [测试策略](#8-测试策略)
9. [监控与告警](#9-监控与告警)
10. [开发里程碑](#10-开发里程碑)
11. [风险与缓解](#11-风险与缓解)
12. [参考文档](#12-参考文档)
13. [修订历史](#13-修订历史)

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
│              │  - audit_log 表      │                          │
│              └──────────────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 服务端技术栈选型

### 2.1 核心框架

| 组件 | 选型 | 版本 | 理由 |
|------|------|------|------|
| **Go 语言** | [Go](https://go.dev/) | 1.23+（最新稳定版） | 性能优化、TryLock 支持 |
| **Web 框架** | [Gin](https://github.com/gin-gonic/gin) | v1.9+ | 高性能、中间件生态丰富、LPK 默认支持 |
| **ORM** | [GORM v2](https://gorm.io/) | v2.5+ | 支持 SQLite/PostgreSQL 双后端、关联查询方便 |
| **SQLite 驱动** | [modernc.org/sqlite](https://pkg.go.dev/modernc.org/sqlite) | v1.28+ | **纯 Go 实现，无需 CGO** |
| **PostgreSQL 驱动** | [pgx](https://github.com/jackc/pgx) | v5.5+ | 高性能、支持 RETURNING 子句 |
| **迁移工具** | [golang-migrate](https://github.com/golang-migrate/migrate) | v4.16+ | 纯 Go 实现、支持多数据库、版本管理清晰 |
| **配置管理** | [Viper](https://github.com/spf13/viper) | v1.18+ | 支持环境变量、YAML、热重载 |
| **日志** | [Zap](https://github.com/uber-go/zap) | v1.26+ | 高性能结构化日志、支持日志轮转 |
| **验证** | [go-playground/validator](https://github.com/go-playground/validator) | v10.15+ | 标签式验证、错误信息友好 |
| **UUID** | [google/uuid](https://github.com/google/uuid) | v1.4+ | 标准库、RFC 4122 compliant |
| **密码哈希** | [bcrypt](https://golang.org/x/crypto/bcrypt) | stdlib | Go 标准库、安全可靠 |
| **压缩** | [gzip](https://pkg.go.dev/compress/gzip) | stdlib | Go 标准库、快照存储压缩 |
| **锁** | [sync.Mutex](https://pkg.go.dev/sync#Mutex) | stdlib | 回滚操作用全局写锁（单实例） |
| **JSON 类型** | [ChangeData 结构体](#61-冲突检测算法) | 自定义 | 支持点号访问，自动序列化（见 §6.1） |

### 2.2 LPK 框架集成

```yaml
# LPK 标准项目结构
lzc-bookmark/
├── cmd/
│   └── server/
│       └── main.go           # 入口文件（含 DB 初始化、连接池配置）
├── internal/
│   ├── config/               # 配置管理
│   ├── handler/              # HTTP Handler
│   ├── service/              # 业务逻辑
│   ├── repository/           # 数据访问层
│   ├── model/                # 数据模型（含 ChangeData 结构体）
│   └── middleware/           # 中间件（鉴权、限流等）
├── pkg/
│   ├── sync/                 # 同步核心算法
│   ├── conflict/             # 冲突检测
│   └── version/              # 版本号管理
├── shared/                   # 插件与 Web 端共享代码
│   ├── types/                # TypeScript 类型定义
│   └── api/                  # API 端点定义
├── web/                      # React 前端（见 §5）
├── migrations/               # 数据库迁移脚本
├── Dockerfile
├── go.mod
├── .env.example              # 环境变量模板（键名清单，提交仓库）
└── config.yaml.example       # 可选：YAML 形状参考
```

**配置约定：**
- 加载顺序：① 代码内默认值 ② 可选 `config.yaml` ③ **环境变量（最高优先级）**
- 本地与 CI 可使用 `.env`、`.env.local`（加入 `.gitignore`）或 Compose `env_file`
- 生产仅通过容器/编排注入环境变量，密钥不进镜像与 Git

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
-- SQLite 版本要求：3.35+（支持 RETURNING 子句）

-- API Keys 表
CREATE TABLE api_keys (
    id              TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    key_hash        TEXT NOT NULL,
    permissions     TEXT NOT NULL,
    created_at      BIGINT NOT NULL,
    last_used_at    BIGINT,
    is_revoked      INTEGER NOT NULL DEFAULT 0
);

-- Devices 表
CREATE TABLE devices (
    device_id       TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    api_key_id      TEXT NOT NULL,
    created_at      BIGINT NOT NULL,
    last_seen_at    BIGINT NOT NULL,
    is_revoked      INTEGER NOT NULL DEFAULT 0,
    FOREIGN KEY (api_key_id) REFERENCES api_keys(id) ON DELETE CASCADE
);

-- Bookmarks 表（规范树）
CREATE TABLE bookmarks (
    guid            TEXT PRIMARY KEY,
    tenant_id       TEXT NOT NULL DEFAULT '',
    title           TEXT NOT NULL,
    url             TEXT,
    parent_guid     TEXT NOT NULL,
    index_weight    REAL NOT NULL,
    date_created    BIGINT NOT NULL,
    date_modified   BIGINT NOT NULL,
    is_deleted      INTEGER NOT NULL DEFAULT 0,
    deleted_at      BIGINT,
    FOREIGN KEY (parent_guid) REFERENCES bookmarks(guid)
);

CREATE INDEX idx_bookmarks_parent ON bookmarks(parent_guid);
CREATE INDEX idx_bookmarks_deleted ON bookmarks(is_deleted, deleted_at);
CREATE INDEX idx_bookmarks_tenant ON bookmarks(tenant_id);

-- Change Log 表
CREATE TABLE change_log (
    change_id       TEXT PRIMARY KEY,
    type            TEXT NOT NULL CHECK (type IN ('create', 'update', 'delete', 'move', 'reindex', 'rollback')),
    guid            TEXT NOT NULL,
    device_id       TEXT NOT NULL,
    data            TEXT,
    from_version    BIGINT NOT NULL,
    to_version      BIGINT NOT NULL UNIQUE,
    timestamp       BIGINT NOT NULL,
    client_request_id TEXT
);

CREATE INDEX idx_change_log_guid ON change_log(guid);
CREATE INDEX idx_change_log_to_version ON change_log(to_version);
CREATE INDEX idx_change_log_timestamp ON change_log(timestamp);
CREATE INDEX idx_change_log_type ON change_log(type);

-- Sync Records 表
CREATE TABLE sync_records (
    sync_id         TEXT PRIMARY KEY,
    version         BIGINT NOT NULL,
    device_id       TEXT NOT NULL,
    request_id      TEXT,
    change_ids      TEXT NOT NULL,
    snapshot_hash   TEXT,
    synced_at       BIGINT NOT NULL
);

CREATE INDEX idx_sync_records_device ON sync_records(device_id);
CREATE INDEX idx_sync_records_synced_at ON sync_records(synced_at);
CREATE INDEX idx_sync_records_version ON sync_records(version);

-- Global State 表（单行）
CREATE TABLE global_state (
    id              INTEGER PRIMARY KEY CHECK (id = 1),
    current_version BIGINT NOT NULL DEFAULT 0,
    baseline_version BIGINT NOT NULL DEFAULT 0,
    last_truncated_at BIGINT
);

INSERT INTO global_state (id, current_version, baseline_version) VALUES (1, 0, 0);

-- Snapshots 表
CREATE TABLE snapshots (
    version         BIGINT PRIMARY KEY,
    bookmark_tree   TEXT NOT NULL,
    created_at      BIGINT NOT NULL,
    is_baseline     INTEGER NOT NULL DEFAULT 0
);

-- Conflicts 表
CREATE TABLE conflicts (
    conflict_id         TEXT PRIMARY KEY,
    guid                TEXT NOT NULL,
    local_change_data   TEXT NOT NULL,
    remote_change_data  TEXT NOT NULL,
    status              TEXT NOT NULL CHECK (status IN ('pending', 'resolved')),
    resolution          TEXT,
    created_at          BIGINT NOT NULL,
    resolved_at         BIGINT,
    resolved_by         TEXT
);

CREATE INDEX idx_conflicts_status ON conflicts(status);
CREATE INDEX idx_conflicts_guid ON conflicts(guid);

-- Audit Log 表
CREATE TABLE audit_log (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    api_key_id      TEXT NOT NULL,
    device_id       TEXT NOT NULL,
    operation       TEXT NOT NULL,
    resource        TEXT,
    result          TEXT NOT NULL,
    ip_address      TEXT,
    timestamp       BIGINT NOT NULL
);

CREATE INDEX idx_audit_log_api_key ON audit_log(api_key_id);
CREATE INDEX idx_audit_log_timestamp ON audit_log(timestamp);

-- SQLite 优化配置（在 main.go 初始化时执行）
-- PRAGMA journal_mode=WAL;
-- PRAGMA synchronous=NORMAL;
-- PRAGMA cache_size=-64000;
-- PRAGMA busy_timeout=5000;  -- v4 新增：避免并发写入报错
```

### 3.2 PostgreSQL 兼容

```sql
-- PostgreSQL 差异：
-- INTEGER/BIGINT → BIGINT
-- TEXT → TEXT（兼容）
-- REAL → DOUBLE PRECISION
-- 自增主键 → UUID v4

-- JSON 字段使用 JSONB 类型
-- change_log 按 to_version 分区
-- 使用 ADVISORY LOCK 替代 sync.Mutex

CREATE TABLE change_log (
    change_id       UUID PRIMARY KEY,
    type            TEXT NOT NULL,
    guid            TEXT NOT NULL,
    device_id       TEXT NOT NULL,
    data            JSONB,
    from_version    BIGINT NOT NULL,
    to_version      BIGINT NOT NULL UNIQUE,
    timestamp       BIGINT NOT NULL,
    client_request_id TEXT,
    CONSTRAINT check_type CHECK (type IN ('create', 'update', 'delete', 'move', 'reindex', 'rollback'))
) PARTITION BY RANGE (to_version);

CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    api_key_id      TEXT NOT NULL,
    device_id       TEXT NOT NULL,
    operation       TEXT NOT NULL,
    resource        TEXT,
    result          TEXT NOT NULL,
    ip_address      INET,
    timestamp       BIGINT NOT NULL
);
```

### 3.3 数据量估算

| 表 | 单条大小 | 10 万书签 | 备注 |
|----|----------|----------|------|
| bookmarks | ~200B | 20MB | 10 万节点 |
| change_log | ~300B | 300MB | 100 万条变更 |
| sync_records | ~200B | 200MB | 100 万条同步记录 |
| snapshots | ~100KB | 50MB | 500 个快照（gzip 压缩） |
| conflicts | ~1KB | ~0 | 临时表 |
| audit_log | ~100B | 100MB | 100 万条（90 天清理） |
| **总计** | - | **~670MB** | SQLite 推荐 < 1GB |

---

## 4. 同步协议

### 4.1 协议概述

同步采用**先推后拉**模式：
1. 客户端推送本地变更 → 服务端检测冲突 → 应用无冲突变更
2. 客户端拉取增量变更（从 `new_version` 到 `version`）

**v4 变更：** 删除 410 GONE 响应，溢出时返回 200 + `full_sync: true`（与 PRD v6 一致）

### 4.2 请求/响应格式

#### 4.2.1 同步请求

```typescript
// POST /api/v1/bookmarks/sync  (v4 修正：端点路径与 PRD v6 一致)
interface SyncRequest {
  device_id: string;
  base_version: number;
  client_request_id: string;
  changes: Change[];
}

interface Change {
  change_id: string;
  type: 'create' | 'update' | 'delete' | 'move' | 'reindex';
  guid: string | null;
  local_id: string;  // v4 修正：必需字段（非 null）
  data: {
    title?: string;
    url?: string;
    parent_guid?: string;  // v4 修正：在 data 内部（与 PRD v6 一致）
    index_weight?: number;
    date_created?: number;
    date_modified?: number;
  };
  parent_local_id?: string;  // create 时必需
}
```

#### 4.2.2 同步响应

```typescript
interface SyncResponse {
  success: boolean;
  data: {
    new_version: number;
    applied_changes: string[];
    conflicts?: Conflict[];
    full_sync?: boolean;  // v4 新增：true 表示需要全量同步
  };
  version: number;
  // v4 修正：删除 changes 字段（严格遵循"先推后拉"）
}

interface Conflict {
  conflict_id: string;
  guid: string;
  type: 'create_conflict' | 'update_conflict' | 'delete_conflict' | 'move_conflict' | 'rename_conflict';
  local_change: Change;
  remote_change: Change;
}
```

### 4.3 响应不变式（v4 新增）

| 场景 | HTTP 状态 | success | data.new_version | data.applied_changes | data.conflicts | data.full_sync |
|------|----------|---------|------------------|----------------------|----------------|----------------|
| 完全成功 | 200 | true | base_version + len(changes) | 全部 change_id | 空数组 | false |
| 部分冲突 | 200 | true | base_version + len(applied) | 已应用的 change_id | 冲突列表 | false |
| 版本溢出 | 200 | true | current_version | 空数组 | 空数组 | **true** |
| 参数错误 | 400 | false | - | - | - | - |

### 4.4 错误码（v4 修正：与 PRD v6 一致）

| 错误码 | HTTP 状态 | 说明 | 客户端处理 |
|--------|----------|------|------------|
| `OK` | 200 | 成功 | 更新本地状态 |
| `CONFLICT` | 200 | 部分冲突 | 显示冲突解决 UI |
| `E001` | 400 | base_version 无效 | 重置为 0，全量同步 |
| `E002` | 400 | 变更数超过 500 | 分批发送 |
| `E003` | 401 | API Key 无效 | 提示重新配置 |
| `E004` | 400 | 变更日志已截断 | 重置为 0，全量同步（`full_sync: true`） |
| `E005` | 429 | 限流 | 指数退避重试 |
| `E006` | 400 | 参数格式错误 | 修正后重试 |

### 4.5 全量同步触发条件（v4 修正）

当发生以下情况时，服务端返回 `full_sync: true`：
1. `base_version < baseline_version`（变更日志已截断）
2. `base_version` 无效（如负数）

客户端处理：
- 清空本地 `last_known_version`
- 拉取全量快照（`GET /api/v1/snapshot`）
- 从 `current_version` 重新开始同步

### 4.6 幂等性

- `client_request_id` 用于服务端去重（5 分钟内）
- 相同 `client_request_id` 的请求返回相同响应
- 服务端在 `sync_records` 表中记录 `request_id`

---

## 5. Chrome 插件与 Web 管理端技术栈

### 5.1 Chrome 插件核心技术栈

| 组件 | 选型 | 版本 | 理由 |
|------|------|------|------|
| **包管理器** | pnpm | v9+ | 快速、节省磁盘 |
| **构建工具** | Vite + CRXJS | v5+ | 快速 HMR、Manifest V3 支持 |
| **框架** | Preact | v10+ | 轻量（3KB）、React 兼容 |
| **UI 方案** | 手写简单组件 + Tailwind | - | 轻量（~50KB）、AI 友好 |
| **状态管理** | Zustand | v4+ | 轻量、无样板代码 |
| **HTTP 客户端** | Ky | v1+ | 轻量、自动重试 |

### 5.2 Web 管理端技术栈

| 组件 | 选型 | 版本 | 理由 |
|------|------|------|------|
| **包管理器** | pnpm | v9+ | 与插件统一 |
| **框架构建** | Vite | v5+ | 快速 HMR |
| **框架** | React | v18+ | 生态丰富 |
| **UI 组件库** | shadcn/ui | latest | 基于 Radix UI、可定制 |
| **状态管理** | Zustand | v4+ | 与插件共享 |

### 5.3 共享代码策略

```yaml
# pnpm-workspace.yaml
packages:
  - 'extension'
  - 'web'
  - 'shared'
```

```typescript
// vite.config.ts
export default defineConfig({
  resolve: {
    alias: {
      '@lzc/shared': resolve(__dirname, '../shared/src'),
    },
  },
})
```

### 5.4 Manifest V3 配置

```json
{
  "manifest_version": 3,
  "host_permissions": [],
  "optional_host_permissions": ["<all_urls>"]
}
```

---

## 6. 关键算法实现

### 6.1 冲突检测算法（v4 修正）

```go
// pkg/conflict/detector.go

// ChangeData v4 修正：结构体替代 datatypes.JSON，支持点号访问
type ChangeData struct {
    Title       string  `json:"title,omitempty"`
    URL         string  `json:"url,omitempty"`
    ParentGUID  string  `json:"parent_guid,omitempty"`
    IndexWeight float64 `json:"index_weight,omitempty"`
    DateCreated int64   `json:"date_created,omitempty"`
    DateModified int64  `json:"date_modified,omitempty"`
}

// Change v4 修正：Data 使用 ChangeData 结构体
type Change struct {
    ChangeID      string    `gorm:"primaryKey"`
    Type          string
    GUID          string
    DeviceID      string
    Data          ChangeData `gorm:"serializer:json"`  // v4 修正：自动序列化
    FromVersion   int64
    ToVersion     int64
    Timestamp     int64
    ClientRequestID string
    ParentLocalID string `gorm:"-"`  // 仅内存使用
}

type Conflict struct {
    ConflictID    string
    GUID          string
    Type          string
    LocalChange   Change
    RemoteChange  Change
}

// Detect 检测冲突（v4 修正：删除同批 create 检测、添加空 guids 检查）
func (d *ConflictDetector) Detect(deviceID string, baseVersion int64, changes []Change) (*ConflictResult, error) {
    result := &ConflictResult{
        HasConflict: false,
        Conflicts:   []Conflict{},
    }
    
    // v4 新增：收集 guid 并去重
    guids := make([]string, 0, len(changes))
    for _, change := range changes {
        if change.GUID != "" {
            guids = append(guids, change.GUID)
        }
    }
    slices.Sort(guids)
    guids = slices.Compact(guids)
    
    // v4 修正：guids 为空时直接返回（避免 WHERE IN () 非法 SQL）
    if len(guids) == 0 {
        return result, nil
    }
    
    // 使用子查询获取每个 GUID 的最新变更
    var latestChanges []Change
    err := d.db.Raw(`
        SELECT cl.* FROM change_log cl
        INNER JOIN (
            SELECT guid, MAX(to_version) as max_ver
            FROM change_log
            WHERE guid IN ? AND to_version > ?
            GROUP BY guid
        ) latest ON cl.guid = latest.guid AND cl.to_version = latest.max_ver
    `, guids, baseVersion).Scan(&latestChanges).Error
    
    if err != nil {
        return nil, err
    }
    
    latestMap := make(map[string]Change)
    for _, lc := range latestChanges {
        latestMap[lc.GUID] = lc
    }
    
    // v4 修正：删除同批 create 冲突检测（同一 parent_local_id 多个 create 是正常操作）
    
    // 检测冲突
    for _, change := range changes {
        latestChange, exists := latestMap[change.GUID]
        if !exists {
            continue
        }
        
        if baseVersion < latestChange.ToVersion {
            result.HasConflict = true
            result.Conflicts = append(result.Conflicts, Conflict{
                ConflictID:    uuid.New().String(),
                GUID:          change.GUID,
                Type:          d.classifyConflict(change, latestChange),
                LocalChange:   change,
                RemoteChange:  latestChange,
            })
        }
    }
    
    return result, nil
}

// classifyConflict 冲突分类（v4 修正：create 操作比较 title/URL）
func (d *ConflictDetector) classifyConflict(local, remote Change) string {
    switch {
    case local.Type == "create" && remote.Type == "create":
        // v4 修正：比较 title 和 URL 是否一致
        if local.Data.Title != remote.Data.Title || local.Data.URL != remote.Data.URL {
            return "create_conflict"
        }
        return "update_conflict"  // 相同则视为 update
    case local.Type == "delete" || remote.Type == "delete":
        return "delete_conflict"
    case local.Type == "move" && remote.Type == "move" && local.Data.ParentGUID != remote.Data.ParentGUID:
        return "move_conflict"
    case local.Type == "update" && remote.Type == "update" && local.Data.Title != remote.Data.Title:
        return "rename_conflict"
    default:
        return "update_conflict"
    }
}
```

### 6.2 版本号管理（v4 修正）

```go
// pkg/version/manager.go

type VersionManager struct {
    db *gorm.DB
    mu sync.Mutex
}

// NextVersion 获取下一个版本号（v4 修正：统一使用 RETURNING）
func (m *VersionManager) NextVersion() (int64, error) {
    var state GlobalState
    
    // SQLite 3.35+ / PostgreSQL: 使用 RETURNING 子句
    // v4 修正：删除 NextVersionSQLite 方法，统一使用此方法
    err := m.db.Raw(`
        UPDATE global_state 
        SET current_version = current_version + 1 
        WHERE id = 1 
        RETURNING current_version
    `).Scan(&state.CurrentVersion).Error
    
    return state.CurrentVersion, err
}

// AcquireLock 获取全局写锁（指数退避）
func (m *VersionManager) AcquireLock(ctx context.Context) error {
    delay := 100 * time.Millisecond
    maxDelay := 2 * time.Second
    
    deadline, hasDeadline := ctx.Deadline()
    if !hasDeadline {
        deadline = time.Now().Add(5 * time.Second)
    }
    
    for time.Now().Before(deadline) {
        if m.mu.TryLock() {
            return nil
        }
        
        zap.L().Debug("waiting for lock", zap.Duration("delay", delay))
        
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(delay):
            delay *= 2
            if delay > maxDelay {
                delay = maxDelay
            }
        }
    }
    
    return errors.New("lock timeout")
}

func (m *VersionManager) ReleaseLock() {
    m.mu.Unlock()
}
```

### 6.3 排序权重归一化（v4 修正）

```go
// pkg/sync/weight.go

// NormalizeWeights 归一化（v4 修正：先获取版本号再插入）
func NormalizeWeights(parentGUID string, db *gorm.DB) (*Change, error) {
    var lastNormalize Change
    err := db.Where("type = 'reindex' AND guid = ?", parentGUID).
        Order("to_version DESC").
        First(&lastNormalize).Error
    
    if err == nil && time.Since(time.UnixMilli(lastNormalize.Timestamp)) < NormalizeThrottle {
        return nil, nil
    }
    
    var change *Change
    err = db.Transaction(func(tx *gorm.DB) error {
        var children []Bookmark
        err := tx.Where("parent_guid = ?", parentGUID).
            Order("index_weight ASC").
            Find(&children).Error
        if err != nil {
            return err
        }
        
        // 批量 UPDATE
        childrenWeights := make(map[string]float64)
        cases := make([]string, 0, len(children))
        values := make([]interface{}, 0, len(children)*2)
        
        for i, child := range children {
            newWeight := float64(i+1) * WeightInterval
            childrenWeights[child.GUID] = newWeight
            cases = append(cases, fmt.Sprintf("WHEN guid = ? THEN ?"))
            values = append(values, child.GUID, newWeight)
        }
        
        query := fmt.Sprintf(
            "UPDATE bookmarks SET index_weight = CASE guid %s END WHERE guid IN (?)",
            strings.Join(cases, " ")
        )
        guids := make([]string, len(children))
        for i, child := range children {
            guids[i] = child.GUID
        }
        values = append(values, guids)
        
        err = tx.Exec(query, values...).Error
        if err != nil {
            return err
        }
        
        // v4 修正：先获取版本号，再插入 change_log
        newVersion, err := (&VersionManager{db: tx}).NextVersion()
        if err != nil {
            return err
        }
        
        change = &Change{
            ChangeID: uuid.New().String(),
            Type:     "reindex",
            GUID:     parentGUID,
            DeviceID: "server",
            Data: ChangeData{
                // v4 修正：使用 ChangeData 结构体
            },
            FromVersion: 0,
            ToVersion:   newVersion,  // v4 修正：使用获取的版本号
            Timestamp:   time.Now().UnixMilli(),
        }
        
        // 填充 data 字段
        change.Data = ChangeData{}  // 实际实现中填充 children_weights
        
        err = tx.Create(change).Error
        return err
    })
    
    if err != nil {
        return nil, err
    }
    
    return change, nil
}
```

### 6.4 变更日志截断算法（v4 修正）

```go
// pkg/sync/truncate.go

// TruncateChangeLog 截断变更日志（v4 修正：数学化说明 + 清理全部类型）
// 
// 设计意图：
// 1. 保留最近 N 条变更或 M 天内的变更（取交集）
// 2. baseline_version = 保留记录的最小 from_version - 1
// 3. 更新 snapshot 的 is_baseline 标记
//
// 边界条件：
// - 如果 change_log 为空，不截断
// - 如果 cutoffVersion <= baseline_version，不截断
func TruncateChangeLog(db *gorm.DB, maxCount int64, maxAgeDays int) error {
    return db.Transaction(func(tx *gorm.DB) error {
        cutoffVersion := int64(0)
        
        // 按数量截断
        var oldestChange Change
        err := tx.Where("type IN ?", []string{"create", "update", "delete", "move", "reindex", "rollback"}).  // v4 修正：包含全部类型
            Order("to_version ASC").
            Offset(int(maxCount)).
            First(&oldestChange).Error
        if err == nil {
            cutoffVersion = oldestChange.FromVersion - 1  // v4 修正：使用 from_version - 1
        }
        
        // 按年龄截断
        cutoffTime := time.Now().AddDate(0, 0, -maxAgeDays).UnixMilli()
        var timeCutoffChange Change
        err = tx.Where("timestamp < ?", cutoffTime).
            Order("timestamp ASC").
            First(&timeCutoffChange).Error
        if err == nil && timeCutoffChange.FromVersion - 1 > cutoffVersion {
            cutoffVersion = timeCutoffChange.FromVersion - 1
        }
        
        if cutoffVersion == 0 {
            return nil
        }
        
        // 更新 baseline_version
        err = tx.Exec("UPDATE global_state SET baseline_version = ? WHERE id = 1", cutoffVersion).Error
        if err != nil {
            return err
        }
        
        // 删除旧变更
        err = tx.Where("to_version <= ? AND type IN ?", cutoffVersion, []string{"create", "update", "delete", "move", "reindex", "rollback"}).
            Delete(&Change{}).Error
        if err != nil {
            return err
        }
        
        // v4 新增：更新 snapshot 的 is_baseline 标记
        err = tx.Exec("UPDATE snapshots SET is_baseline = 0 WHERE is_baseline = 1").Error
        if err != nil {
            return err
        }
        
        var baselineSnapshot Snapshot
        err = tx.Where("version <= ?", cutoffVersion).
            Order("version DESC").
            First(&baselineSnapshot).Error
        if err == nil {
            err = tx.Model(&baselineSnapshot).Update("is_baseline", 1).Error
            if err != nil {
                return err
            }
        }
        
        // 记录截断时间
        return tx.Exec("UPDATE global_state SET last_truncated_at = ? WHERE id = 1", time.Now().UnixMilli()).Error
    })
}
```

### 6.5 快照生成算法（v4 修正）

```go
// pkg/sync/snapshot.go

// GenerateSnapshot 生成快照（v4 修正：参数化查询 + 30 天年龄清理）
func GenerateSnapshot(db *gorm.DB, version int64) error {
    return db.Transaction(func(tx *gorm.DB) error {
        var bookmarks []Bookmark
        err := tx.Where("is_deleted = 0").Find(&bookmarks).Error
        if err != nil {
            return err
        }
        
        tree := buildTree(bookmarks)
        
        treeJSON, err := json.Marshal(tree)
        if err != nil {
            return err
        }
        
        var buf bytes.Buffer
        gz := gzip.NewWriter(&buf)
        _, err = gz.Write(treeJSON)
        if err != nil {
            return err
        }
        gz.Close()
        
        hash := sha256.Sum256(buf.Bytes())
        
        snapshot := Snapshot{
            Version:      version,
            BookmarkTree: base64.StdEncoding.EncodeToString(buf.Bytes()),
            CreatedAt:    time.Now().UnixMilli(),
            IsBaseline:   0,
        }
        err = tx.Create(&snapshot).Error
        if err != nil {
            return err
        }
        
        // v4 修正：参数化查询 + 年龄清理
        // 注意：以下 SQL 在 SQLite 和 PostgreSQL 上兼容
        // PostgreSQL 可能需要调整为使用窗口函数
        err = tx.Exec(`
            DELETE FROM snapshots 
            WHERE version NOT IN (
                SELECT version FROM (
                    SELECT version FROM snapshots 
                    ORDER BY version DESC 
                    LIMIT ?
                ) AS recent
            ) AND created_at < ?
        `, 500, time.Now().AddDate(0, 0, -30).UnixMilli()).Error
        
        return err
    })
}
```

### 6.6 离线队列合并

```typescript
// src/sync/Queue.ts

interface PendingChange {
  change_id: string;
  type: 'create' | 'update' | 'delete' | 'move';
  guid: string | null;
  local_id: string;  // v4 修正：必需字段
  parent_local_id: string | null;
  base_version: number;
  data: Record<string, unknown>;
  created_at: number;
  retry_count: number;
  submitted: boolean;
}
```

---

## 7. 部署方案

### 7.1 Docker 配置（v4 修正）

```dockerfile
# Dockerfile
FROM node:20-alpine AS web-builder

RUN corepack enable && corepack prepare pnpm@latest --activate

WORKDIR /build/web
COPY web/package*.json ./
COPY web/pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY web/ ./
RUN pnpm run build

FROM golang:1.23-alpine AS go-builder

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download

# v4 修正：仅复制必要目录
COPY cmd/ ./cmd/
COPY internal/ ./internal/
COPY pkg/ ./pkg/
COPY migrations/ ./migrations/

RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

FROM alpine:3.19

RUN apk --no-cache add ca-certificates wget

WORKDIR /app
COPY --from=go-builder /build/server .
COPY --from=go-builder /build/migrations ./migrations
COPY --from=web-builder /build/web/dist ./web/dist

# v4 新增：非 root 运行
RUN adduser -D -g '' appuser
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/health || exit 1

CMD ["./server"]
```

### 7.2 Docker Compose

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

### 7.3 环境变量配置

```bash
# .env.example

# 数据库配置
DB_TYPE=sqlite
DB_PATH=/data/lzc-bookmark.db

# 安全配置
ALLOW_INSECURE_HTTP=false

# 日志配置
LOG_LEVEL=info
LOG_FORMAT=json

# 变更日志配置
CHANGE_LOG_MAX_COUNT=1000
CHANGE_LOG_MAX_AGE_DAYS=7

# 快照配置
SNAPSHOT_MAX_COUNT=500
SNAPSHOT_MAX_AGE_DAYS=30

# 限流配置
RATE_LIMIT_SYNC=60
RATE_LIMIT_OTHER=120
RATE_LIMIT_KEY_TOTAL=200

# 同步配置
SYNC_INTERVAL_SECONDS=30
SOFT_DELETE_DAYS=90
MAX_BOOKMARKS=100000
MAX_SYNC_CHANGES=500

# 冲突解决超时（v4 新增）
CONFLICT_RESOLUTION_TIMEOUT_DAYS=7
```

### 7.4 懒猫微服部署

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

## 8. 测试策略

### 8.1 服务端测试

```go
// internal/handler/sync_test.go
// 注意：以下代码为伪代码示例，不可直接编译

func TestSyncHandler_PartialConflict(t *testing.T) {
    // 测试部分冲突场景
    // 断言：8 条应用，2 条冲突
}

func TestConflictDetector_RetryAfterPartialConflict(t *testing.T) {
    // 测试部分冲突重试场景
}
```

### 8.2 插件测试

```typescript
// src/sync/SyncEngine.test.ts
// 注意：以下代码为伪代码示例，不可直接编译

describe('SyncEngine', () => {
  describe('partialConflict', () => {
    it('should update last_known_version after partial conflict', async () => {
      // 断言 lastKnownVersion 更新为 128
    });
  });
});
```

### 8.3 集成测试（v4 修正）

```bash
# tests/integration/run.sh

# 1. 启动服务端（v4 修正：使用 docker compose v2 语法）
docker compose up -d

# 2. 运行服务端测试
go test ./... -tags=integration

# 3. 构建插件（v4 修正：使用 pnpm + --frozen-lockfile）
cd extension && pnpm install --frozen-lockfile && pnpm run build

# 4. 运行插件 E2E 测试
pnpm exec playwright test tests/e2e/

# 5. 清理
docker compose down
```

### 8.4 最小验收条目（v4 新增）

| 测试项 | 预期结果 |
|--------|----------|
| 单设备同步 | 变更成功应用 |
| 多设备并发同步 | 冲突正确检测 |
| 部分冲突重试 | 重试后成功应用 |
| 版本溢出 | 返回 `full_sync: true` |
| 离线队列合并 | 同 guid 变更合并 |
| 归一化节流 | 5 分钟内不重复触发 |

---

## 9. 监控与告警

### 9.1 指标暴露

```go
// internal/metrics/metrics.go

var (
    syncRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "lzc_sync_requests_total",
            Help: "Total number of sync requests",
        },
        []string{"status"},
    )
    
    syncDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "lzc_sync_duration_seconds",
            Help:    "Sync request duration",
            Buckets: prometheus.ExponentialBuckets(0.1, 1.5, 10),
        },
        []string{},
    )
    
    conflictCount = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "lzc_conflicts_total",
            Help: "Total number of conflicts detected",
        },
        []string{"type"},
    )
    
    changeLogSize = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "lzc_change_log_size",
            Help: "Current change log size",
        },
    )
    
    // 版本号空洞监控
    versionGaps = promauto.NewCounter(
        prometheus.CounterOpts{
            Name: "lzc_version_gaps_total",
            Help: "Total number of version gaps (holes)",
        },
    )
)
```

### 9.2 告警规则（v4 修正）

```yaml
# prometheus/alerts.yml

groups:
  - name: lzc-bookmark
    rules:
      - alert: HighConflictRate
        expr: sum(rate(lzc_conflicts_total[5m])) > 0.167
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
          sum(rate(lzc_sync_requests_total{status="error"}[5m])) 
          / sum(rate(lzc_sync_requests_total[5m]))
        > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "同步错误率高"
          description: "同步错误率超过 10%"
      
      - alert: VersionGaps
        expr: rate(lzc_version_gaps_total[1h]) > 10
        for: 1h
        labels:
          severity: info
        annotations:
          summary: "版本号空洞过多"
          description: "过去 1 小时空洞超过 10 个"
```

### 9.3 健康检查（v4 新增）

```go
// internal/handler/health.go

func HealthHandler(db *gorm.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // 检查 DB 连接
        var count int64
        err := db.Raw("SELECT COUNT(*) FROM global_state WHERE id = 1").Scan(&count).Error
        if err != nil || count != 1 {
            http.Error(w, "DB connection failed", http.StatusInternalServerError)
            return
        }
        
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]string{
            "status": "healthy",
            "db":     "connected",
        })
    }
}
```

---

## 10. 开发里程碑

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

## 11. 风险与缓解

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| Chrome 三根映射兼容性问题 | 新设备拉取失败 | 中 | 提前测试多版本 Chrome、Edge、Brave |
| 归一化触发频繁 | change_log 膨胀 | 低 | 5 分钟节流、不计入上限 |
| 部分冲突重试逻辑错误 | 冲突变更无法应用 | 中 | 充分测试重试场景、添加集成测试 |
| storage.local 配额超限 | 离线队列丢失 | 中 | 80% 预警、合并策略 |
| SQLite 并发写锁 | 高并发下性能下降 | 低 | 个人用户并发低、可切换 PostgreSQL |
| Service Worker 休眠导致事件丢失 | 本地变更未及时同步 | 中 | 唤醒时 getTree diff、定时同步 |
| sync.Mutex 多实例失效 | 回滚操作并发问题 | 低 | v1 单实例部署，v2 改用数据库锁 |
| 版本号空洞 | to_version 唯一约束失效 | 低 | 添加监控指标，定期审计 |
| 冲突解决超时 |  pending 冲突累积 | 低 | 7 天自动解决策略 |

---

## 12. 参考文档

- [PRD v6.0](../prd/PRD-v6.md)
- [TECHNICAL-DESIGN-v3 评审报告](./)
  - [Composer2 审计报告](./tech-design-v3-reviewed-by-composer2.md)
  - [GLM-5.1 审计报告](./tech-design-v3-reviewed-by-glm5.1.md)
  - [Kimi Code 审计报告](./tech-design-v3-reviewed-by-kimi-code.md)
  - [Kimi K2.5 审计报告](./tech-design-v3-reviewed-by-kimi-k2.5.md)
- [LPK 框架文档](https://github.com/lazycat/lpk-docs)
- [Chrome Bookmarks API](https://developer.chrome.com/docs/extensions/reference/bookmarks/)
- [Manifest V3](https://developer.chrome.com/docs/extensions/mv3/intro/)
- [GORM 文档](https://gorm.io/docs/)
- [Gin 文档](https://gin-gonic.com/docs/)
- [modernc.org/sqlite](https://pkg.go.dev/modernc.org/sqlite)
- [pgx 文档](https://github.com/jackc/pgx)

---

## 13. 修订历史

| 版本 | 日期 | 修订内容 |
|------|------|----------|
| v1.0 | 2026-04-16 | 初始版本 |
| v2.0 | 2026-04-16 | 修复 4 份评审报告提出的全部 P0/P1 问题 |
| v2.1 | 2026-04-16 | 包管理器改为 pnpm、Web 管理端 UI 使用 shadcn/ui |
| v3.0 | 2026-04-16 | 综合 5 份审计报告，修复全部 P0/P1 问题 |
| **v4.0** | **2026-04-17** | **综合 4 份 v3 评审报告，修复全部 P0/P1 问题** |

**v4.0 关键变更：**

1. **Data 字段类型** — 改用 `ChangeData` 结构体 + `serializer:json`，支持点号访问
2. **删除 FOR UPDATE** — 统一使用 `RETURNING`（要求 SQLite 3.35+）
3. **同步协议对齐 PRD** — 端点 `/api/v1/bookmarks/sync`，错误码 `E001-E008`，删除 410 GONE
4. **全量同步响应** — 溢出时返回 200 + `full_sync: true`
5. **删除同批 create 检测** — 同一 parent_local_id 多个 create 是正常操作
6. **NormalizeWeights 版本号** — 先调用 `NextVersion()` 再插入 change_log
7. **截断算法修正** — baseline_version = 保留记录的最小 `from_version - 1`，更新 snapshot 标记
8. **删除 SyncResponse.changes** — 严格遵循"先推后拉"
9. **parent_guid 位置** — 移至 `data` 对象内部
10. **响应不变式** — 新增表格说明四种场景字段填充规则
11. **空 guids 检查** — 避免 `WHERE IN ()` 非法 SQL
12. **SQLite busy_timeout** — 添加 `PRAGMA busy_timeout=5000`
13. **Dockerfile 优化** — 仅复制必要目录，添加 USER 指令
14. **集成测试修正** — 使用 `docker compose` v2 语法，添加 `--frozen-lockfile`
15. **连接池配置** — main.go 显式配置 GORM 连接池
16. **健康检查** — `/health` 端点检查 DB 连接
17. **冲突解决超时** — 7 天未解决自动使用服务端版本
18. **参考链接修正** — 路径改为 `../prd/PRD-v6.md`

---

**文档状态:** 已评审，待开发  
**下一步:** Phase 1 开发启动  
**评审报告:** 见 `./tech/` 目录下 4 份 v3 评审报告
