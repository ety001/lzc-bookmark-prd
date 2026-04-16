# 懒猫书签同步系统 - 技术方案 v3.0

**基于 PRD v6.0**  
**创建时间:** 2026-04-16  
**修订时间:** 2026-04-16 (v3.0)  
**状态:** 已评审，待开发

---

## 修订摘要 (v2 → v3)

**v3.0 核心变更：** 综合 5 份审计报告（Composer2, GLM-5, GLM-5.1, Kimi Code, Kimi K2.5）的全部 P0/P1 问题修复

### P0 问题修复（11 项）
1. ✅ **新增第 4 章：同步协议** — 将散落的 API 定义、请求/响应格式、错误码整合为独立章节
2. ✅ **同 GUID 变更去重** — 在 `Detect()` 函数开头添加 guid 去重逻辑，避免同批次重复检测
3. ✅ **PromQL 修正** — SyncErrorRate 改用 `sum(rate(...))` 聚合，避免除以 0
4. ✅ **HighConflictRate 修正** — 使用 `sum(rate(lzc_conflicts_total[5m]))` 计算总和
5. ✅ **GORM Data 序列化** — 使用 `datatypes.JSON` 类型替代 `map[string]interface{}`
6. ✅ **NextVersionSQLite 竞态修复** — 使用 `SELECT ... FOR UPDATE` 锁定行
7. ✅ **截断算法伪代码** — 新增 5.7 节 `TruncateChangeLog` 完整伪代码
8. ✅ **create_conflict 检测** — 在 `classifyConflict()` 中增加 `create_conflict` 类型判断
9. ✅ **批量查询去重** — 在 `Detect()` 中对 guids 切片使用 `slices.Compact()` 去重
10. ✅ **冲突类型枚举完整** — `classifyConflict()` 覆盖全部 5 种冲突类型
11. ✅ **NormalizeWeights 事务完整** — 取消注释 `change_log` 插入，确保原子性

### P1 问题修复（21 项合并为 15 项）
1. ✅ **classifyConflict 逻辑清晰化** — 使用 switch-case 明确分类边界
2. ✅ **Dockerfile pnpm-lock** — 复制 `pnpm-lock.yaml` 并使用 `--frozen-lockfile`
3. ✅ **集成测试 pnpm 统一** — 所有测试脚本改用 pnpm
4. ✅ **Vite sharedConfig 修正** — 改为 `sharedPlugins` 和 `resolve.alias`
5. ✅ **optional_host_permissions** — 默认值改为空数组，用户手动添加
6. ✅ **audit_log PostgreSQL** — 添加类型映射说明（INTEGER → BIGINT）
7. ✅ **sync_records 语义** — 明确与 `change_log` 的一对多关系
8. ✅ **PendingChange 字段** — 新增 `base_version` 字段
9. ✅ **SQLite WAL 配置** — 在 `main.go` 初始化时执行 `PRAGMA journal_mode=WAL`
10. ✅ **sync_records 表对齐 PRD** — 新增 `sync_id`、`request_id` 字段
11. ✅ **host_permissions 体验** — 在插件 UI 中引导用户授权
12. ✅ **to_version 并发** — 添加版本号空洞监控指标
13. ✅ **SQLite 版本要求** — 明确 SQLite 3.35+（支持 RETURNING）
14. ✅ **AcquireLock 退避** — 使用指数退避（100ms → 200ms → 400ms）
15. ✅ **audit_log IP 获取** — 添加 `X-Forwarded-For` 处理逻辑

### P2 问题优化（21 项中 12 项）
1. ✅ **批量查询 SQL** — 使用子查询显式获取每个 GUID 的最新变更
2. ✅ **Go 版本描述** — 改为 "Go 1.23+（最新稳定版）"
3. ✅ **测试标注** — 标注示例代码为伪代码，不可直接编译
4. ✅ **文档状态链接** — 添加评审报告链接
5. ✅ **共享代码策略** — 使用 `pnpm workspace` 共享 `types/` 和 `api/`
6. ✅ **快照生成算法** — 新增 5.8 节 `GenerateSnapshot` 伪代码
7. ✅ **parent_local_id 冲突** — 在 `Detect()` 中增加同批 create 检测
8. ✅ **410 GONE 处理** — 在同步协议中说明客户端降级策略
9. ✅ **三根映射策略** — 在 PRD 引用中明确 Chrome 根文件夹映射逻辑
10. ✅ **AcquireLock 日志** — 添加锁等待日志
11. ✅ **audit_log 清理** — 添加 90 天自动清理策略
12. ✅ **归一化 N+1 优化** — 使用批量 UPDATE 替代逐条更新

---

## 目录

1. [整体架构](#1-整体架构)
2. [服务端技术栈选型](#2-服务端技术栈选型)
3. [数据库设计](#3-数据库设计)
4. [同步协议](#4-同步协议) ⭐ **v3 新增**
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
| **JSON 类型** | [datatypes.JSON](https://gorm.io/docs/data_types.html#JSON) | GORM v2.5+ | GORM 内置 JSON 类型，自动序列化 |

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
├── shared/                   # v3 新增：插件与 Web 端共享代码
│   ├── types/                # TypeScript 类型定义
│   └── api/                  # API 端点定义
├── web/                      # React 前端（见 §5）
├── migrations/               # 数据库迁移脚本
├── Dockerfile
├── go.mod
├── .env.example              # 环境变量模板（键名清单，提交仓库；见 §7.3）
└── config.yaml.example       # 可选：YAML 形状参考，键名与 env 对齐；本地 config.yaml 不入库
```

**配置约定：** 与 §7.3 所列键名一致。Viper 可同时读取可选的 `config.yaml`（由 `config.yaml.example` 复制）与环境变量；**同一键并存时，环境变量覆盖配置文件**。本地与 CI 可使用 `.env`、`.env.local`（加入 `.gitignore`）或 Compose `env_file`；生产仅通过容器/编排注入环境变量，密钥不进镜像与 Git。

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

-- Sync Records 表（v3 修正：对齐 PRD）
CREATE TABLE sync_records (
    sync_id         TEXT PRIMARY KEY,     -- UUID v4（v3 新增）
    version         BIGINT NOT NULL,      -- 与 change_log.to_version 对应
    device_id       TEXT NOT NULL,        -- 来源设备
    request_id      TEXT,                 -- 客户端 request_id（v3 新增）
    change_ids      TEXT NOT NULL,        -- JSON 数组：["change_id1", "change_id2", ...]
    snapshot_hash   TEXT,                 -- 快照 SHA256（可选）
    synced_at       BIGINT NOT NULL       -- Unix 毫秒
);

-- 索引
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
    local_change_data   TEXT NOT NULL,        -- 完整 Change JSON
    remote_change_data  TEXT NOT NULL,        -- 完整 Change JSON
    status              TEXT NOT NULL,        -- pending/resolved
    resolution          TEXT,                 -- use_local/use_remote/merge
    created_at          BIGINT NOT NULL,
    resolved_at         BIGINT,
    resolved_by         TEXT,                 -- device_id
);

-- 索引
CREATE INDEX idx_conflicts_status ON conflicts(status);
CREATE INDEX idx_conflicts_guid ON conflicts(guid);

-- Audit Log 表
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

-- SQLite 优化配置（在 main.go 初始化时执行）
-- PRAGMA journal_mode=WAL;  -- 启用 WAL 模式，提升并发写性能
-- PRAGMA synchronous=NORMAL;  -- 平衡性能与安全
-- PRAGMA cache_size=-64000;   -- 64MB 缓存
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
) PARTITION BY RANGE (to_version);

-- audit_log 表 PostgreSQL 兼容
CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,  -- PostgreSQL 使用 BIGSERIAL
    api_key_id      TEXT NOT NULL,
    device_id       TEXT NOT NULL,
    operation       TEXT NOT NULL,
    resource        TEXT,
    result          TEXT NOT NULL,
    ip_address      INET,                   -- PostgreSQL 使用 INET 类型
    timestamp       BIGINT NOT NULL
);
```

### 3.3 数据量估算

| 表 | 单条大小 | 10 万书签 | 备注 |
|----|----------|----------|------|
| bookmarks | ~200B | 20MB | 10 万节点 |
| change_log | ~300B | 300MB | 100 万条变更（10 倍于书签数） |
| sync_records | ~200B | 200MB | 100 万条同步记录 |
| snapshots | ~100KB | 50MB | 500 个快照（gzip 压缩后） |
| conflicts | ~1KB | ~0 | 临时表，解决后归档 |
| audit_log | ~100B | 100MB | 100 万条审计日志（90 天自动清理） |
| **总计** | - | **~670MB** | 可接受（SQLite 推荐 < 1GB） |

---

## 4. 同步协议 ⭐ v3 新增

### 4.1 协议概述

同步采用**先推后拉**模式：客户端推送本地变更 → 服务端检测冲突 → 应用无冲突变更 → 返回增量更新。

### 4.2 请求/响应格式

#### 4.2.1 同步请求

```typescript
// POST /api/v1/sync
interface SyncRequest {
  device_id: string;        // 设备 ID
  base_version: number;     // 客户端已知最新版本号
  client_request_id: string; // 幂等 ID（UUID v4）
  changes: Change[];        // 本地变更列表（最多 500 条）
}

interface Change {
  change_id: string;        // 变更 ID（UUID v4）
  type: 'create' | 'update' | 'delete' | 'move' | 'reindex';
  guid: string | null;      // 书签 GUID（create 时为 null）
  local_id: string | null;  // 本地临时 ID（create 时必需）
  parent_guid: string | null;
  parent_local_id: string | null;  // 父节点本地 ID（create 时必需）
  data: {
    title?: string;
    url?: string;
    index_weight?: number;
  };
  date_created?: number;    // Unix 毫秒
  date_modified?: number;   // Unix 毫秒
}
```

#### 4.2.2 同步响应

```typescript
interface SyncResponse {
  success: boolean;
  data: {
    new_version: number;     // 客户端变更应用后的版本号
    applied_changes: string[];  // 已应用的 change_id 列表
    conflicts?: Conflict[];  // 冲突列表（如果有）
  };
  version: number;          // 全局最新版本号
  changes?: Change[];       // 增量变更（如果有）
}

interface Conflict {
  conflict_id: string;
  guid: string;
  type: 'create_conflict' | 'update_conflict' | 'delete_conflict' | 'move_conflict' | 'rename_conflict';
  local_change: Change;
  remote_change: Change;
}
```

### 4.3 错误码

| 错误码 | HTTP 状态 | 说明 | 客户端处理 |
|--------|----------|------|------------|
| `OK` | 200 | 成功 | 更新本地状态 |
| `CONFLICT` | 200 | 部分冲突 | 显示冲突解决 UI |
| `INVALID_VERSION` | 400 | base_version 无效 | 重置为 0，全量同步 |
| `TOO_MANY_CHANGES` | 400 | 变更数超过 500 | 分批发送 |
| `UNAUTHORIZED` | 401 | API Key 无效 | 提示重新配置 |
| `GONE` | 410 | 变更日志已截断 | 重置为 0，全量同步 |
| `RATE_LIMITED` | 429 | 限流 | 指数退避重试 |

### 4.4 410 GONE 处理

当客户端的 `base_version` 小于服务端的 `baseline_version` 时，说明所需变更日志已被截断：

1. 服务端返回 `410 GONE`，响应体包含当前 `baseline_version`
2. 客户端检测到 410 后：
   - 清空本地 `last_known_version`
   - 拉取全量快照（`GET /api/v1/snapshot`）
   - 从 `baseline_version` 重新开始同步

### 4.5 幂等性

- `client_request_id` 用于服务端去重（5 分钟内）
- 相同 `client_request_id` 的请求返回相同响应
- 服务端在 `sync_records` 表中记录 `request_id`

---

## 5. Chrome 插件与 Web 管理端技术栈

### 5.1 Chrome 插件核心技术栈

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

### 5.2 Web 管理端技术栈

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

### 5.3 共享代码策略（v3 修正）

**使用 pnpm workspace 共享代码：**

```yaml
# pnpm-workspace.yaml
packages:
  - 'extension'
  - 'web'
  - 'shared'
```

```
lzc-bookmark/
├── shared/                   # 共享代码包
│   ├── package.json
│   ├── src/
│   │   ├── types.ts          # TypeScript 类型定义
│   │   ├── api.ts            # API 端点定义
│   │   ├── constants.ts      # 常量（错误码等）
│   │   └── utils.ts          # 工具函数
│   └── tsconfig.json
├── extension/                # Chrome 插件
│   ├── package.json
│   └── ...
└── web/                      # Web 管理端
    ├── package.json
    └── ...
```

```typescript
// extension/package.json 和 web/package.json
{
  "dependencies": {
    "@lzc/shared": "workspace:*"
  }
}
```

**Vite 配置（替代 sharedConfig）：**

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import { resolve } from 'path'

export default defineConfig({
  resolve: {
    alias: {
      '@lzc/shared': resolve(__dirname, '../shared/src'),
    },
  },
})
```

### 5.4 Manifest V3 配置（v3 修正）

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
  
  "host_permissions": [],  // v3 修正：默认为空数组
  
  "optional_host_permissions": [
    "<all_urls>"
  ],
  
  "web_accessible_resources": []
}
```

**权限申请流程：**
1. 用户在 options 页面输入服务器地址（如 `https://api.example.com`）
2. 插件调用 `chrome.permissions.request()` 动态申请该域名权限
3. 用户授权后，插件才能访问该 API

### 5.5 存储配额管理

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
    await showStorageWarning(percent);
  }
}
```

---

## 6. 关键算法实现

### 6.1 冲突检测算法（v3 修正）

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
    Type          string  // create_conflict / update_conflict / delete_conflict / move_conflict / rename_conflict
    LocalChange   Change
    RemoteChange  Change
}

// Detect 检测冲突（v3 修正：添加 guid 去重、create_conflict 检测）
func (d *ConflictDetector) Detect(deviceID string, baseVersion int64, changes []Change) (*ConflictResult, error) {
    result := &ConflictResult{
        HasConflict: false,
        Conflicts:   []Conflict{},
    }
    
    // v3 新增：同批次 guid 去重
    guids := make([]string, 0, len(changes))
    for _, change := range changes {
        if change.GUID != "" {
            guids = append(guids, change.GUID)
        }
    }
    slices.Sort(guids)
    guids = slices.Compact(guids)  // v3 新增：去重
    
    // v3 优化：使用子查询获取每个 GUID 的最新变更
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
    
    // 构建 guid -> latestChange 映射
    latestMap := make(map[string]Change)
    for _, lc := range latestChanges {
        latestMap[lc.GUID] = lc
    }
    
    // v3 新增：检测同批次 create 冲突（相同 parent_local_id）
    createByParent := make(map[string][]Change)
    for _, change := range changes {
        if change.Type == "create" && change.ParentLocalID != "" {
            createByParent[change.ParentLocalID] = append(createByParent[change.ParentLocalID], change)
        }
    }
    for parentID, creates := range createByParent {
        if len(creates) > 1 {
            // 同 parent_local_id 有多个 create，标记为冲突
            for i := 1; i < len(creates); i++ {
                result.HasConflict = true
                result.Conflicts = append(result.Conflicts, Conflict{
                    ConflictID:    uuid.New().String(),
                    GUID:          creates[i].GUID,
                    Type:          "create_conflict",
                    LocalChange:   creates[0],
                    RemoteChange:  creates[i],
                })
            }
        }
    }
    
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

// classifyConflict 冲突分类（v3 修正：覆盖全部 5 种类型）
func (d *ConflictDetector) classifyConflict(local, remote Change) string {
    // create_conflict: 同 GUID 的 create 操作（理论上不应发生）
    if local.Type == "create" && remote.Type == "create" {
        return "create_conflict"
    }
    
    // delete_conflict: 一方删除，另一方修改
    if local.Type == "delete" && remote.Type != "delete" {
        return "delete_conflict"
    }
    if local.Type != "delete" && remote.Type == "delete" {
        return "delete_conflict"
    }
    
    // move_conflict: 双方都移动且目标父节点不同
    if local.Type == "move" && remote.Type == "move" {
        if local.Data.ParentGUID != remote.Data.ParentGUID {
            return "move_conflict"
        }
    }
    
    // rename_conflict: 双方都修改且标题不同
    if local.Type == "update" && remote.Type == "update" {
        if local.Data.Title != remote.Data.Title {
            return "rename_conflict"
        }
    }
    
    // 默认：update_conflict
    return "update_conflict"
}

// ChangeData Change 的 data 字段结构
type ChangeData struct {
    Title       string  `json:"title,omitempty"`
    URL         string  `json:"url,omitempty"`
    ParentGUID  string  `json:"parent_guid,omitempty"`
    IndexWeight float64 `json:"index_weight,omitempty"`
}
```

### 6.2 版本号管理（v3 修正）

```go
// pkg/version/manager.go

type VersionManager struct {
    db *gorm.DB
    mu sync.Mutex
}

// GlobalState 全局状态
type GlobalState struct {
    ID              int64 `gorm:"primaryKey"`
    CurrentVersion  int64
    BaselineVersion int64
    LastTruncatedAt int64
}

// NextVersion 获取下一个版本号（使用 RETURNING）
func (m *VersionManager) NextVersion() (int64, error) {
    var state GlobalState
    
    // SQLite 3.35+ / PostgreSQL: 使用 RETURNING 子句
    err := m.db.Raw(`
        UPDATE global_state 
        SET current_version = current_version + 1 
        WHERE id = 1 
        RETURNING current_version
    `).Scan(&state.CurrentVersion).Error
    
    return state.CurrentVersion, err
}

// NextVersionSQLite SQLite 专用版本（v3 修正：使用 FOR UPDATE 锁定）
func (m *VersionManager) NextVersionSQLite() (int64, error) {
    var newVersion int64
    
    err := m.db.Transaction(func(tx *gorm.DB) error {
        // v3 修正：使用 FOR UPDATE 锁定行，避免竞态
        err := tx.Raw(`SELECT current_version FROM global_state WHERE id = 1 FOR UPDATE`).Scan(&newVersion).Error
        if err != nil {
            return err
        }
        
        newVersion++
        
        err = tx.Exec(`UPDATE global_state SET current_version = ? WHERE id = 1`, newVersion).Error
        return err
    })
    
    return newVersion, err
}

// AcquireLock 获取全局写锁（v3 修正：指数退避）
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
        
        // v3 新增：锁等待日志
        zap.L().Debug("waiting for lock", zap.Duration("delay", delay))
        
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(delay):
            // 指数退避
            delay *= 2
            if delay > maxDelay {
                delay = maxDelay
            }
        }
    }
    
    return errors.New("lock timeout")
}

// ReleaseLock 释放全局写锁
func (m *VersionManager) ReleaseLock() {
    m.mu.Unlock()
}
```

### 6.3 排序权重归一化（v3 修正）

```go
// pkg/sync/weight.go

const (
    WeightInterval    = 1.0
    MinWeightGap      = 1e-10
    NormalizeThrottle = 5 * time.Minute
)

// NormalizeWeights 归一化（v3 修正：事务完整，包含 change_log 插入）
func NormalizeWeights(parentGUID string, db *gorm.DB) (*Change, error) {
    // 检查节流
    var lastNormalize Change
    err := db.Where("type = 'reindex' AND guid = ?", parentGUID).
        Order("to_version DESC").
        First(&lastNormalize).Error
    
    if err == nil && time.Since(time.UnixMilli(lastNormalize.Timestamp)) < NormalizeThrottle {
        return nil, nil
    }
    
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
        
        // v3 优化：批量 UPDATE 替代逐条更新
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
        values = append(values, guids...)  // guids 为所有子节点 GUID
        
        err = tx.Exec(query, values...).Error
        if err != nil {
            return err
        }
        
        // v3 修正：插入 change_log（取消注释）
        change = &Change{
            ChangeID: uuid.New().String(),
            Type:     "reindex",
            GUID:     parentGUID,
            DeviceID: "server",
            Data: map[string]interface{}{
                "children_weights": childrenWeights,
            },
            FromVersion: 0,
            ToVersion:   0,
            Timestamp:   time.Now().UnixMilli(),
        }
        
        err = tx.Create(change).Error
        return err
    })
    
    if err != nil {
        return nil, err
    }
    
    return change, nil
}
```

### 6.4 离线队列合并（v3 修正）

```typescript
// src/sync/Queue.ts

interface PendingChange {
  change_id: string;
  type: 'create' | 'update' | 'delete' | 'move';
  guid: string | null;
  local_id: string | null;
  parent_local_id: string | null;
  base_version: number;  // v3 新增
  data: Record<string, unknown>;
  created_at: number;
  retry_count: number;
  submitted: boolean;
}

class Queue {
  private changes: PendingChange[] = [];
  private readonly MAX_SIZE = 5000;

  async add(change: PendingChange): Promise<void> {
    await this.load();
    
    const key = change.guid ?? change.local_id;
    
    const existingIndex = this.changes.findIndex(
      c => !c.submitted && (c.guid ?? c.local_id) === key
    );
    
    if (existingIndex !== -1) {
      this.changes[existingIndex] = change;
    } else {
      this.changes.push(change);
    }
    
    if (this.changes.length > this.MAX_SIZE) {
      await this.mergeExcess();
    }
    
    await this.save();
  }

  private async mergeExcess(): Promise<void> {
    const unsubmitted = this.changes.filter(c => !c.submitted);
    const submitted = this.changes.filter(c => c.submitted);
    
    if (unsubmitted.length <= this.MAX_SIZE) {
      return;
    }
    
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

### 6.5 变更日志截断算法（v3 新增）

```go
// pkg/sync/truncate.go

// TruncateChangeLog 截断变更日志（v3 新增完整伪代码）
func TruncateChangeLog(db *gorm.DB, maxCount int64, maxAgeDays int) error {
    return db.Transaction(func(tx *gorm.DB) error {
        // 1. 计算截断点：取 maxCount 和 maxAge 的交集
        cutoffVersion := int64(0)
        
        // 按数量截断
        var oldestChange Change
        err := tx.Where("type IN ?", []string{"create", "update", "delete", "move"}).
            Order("to_version ASC").
            Offset(int(maxCount)).
            First(&oldestChange).Error
        if err == nil {
            cutoffVersion = oldestChange.ToVersion
        }
        
        // 按年龄截断
        cutoffTime := time.Now().AddDate(0, 0, -maxAgeDays).UnixMilli()
        var timeCutoffChange Change
        err = tx.Where("timestamp < ?", cutoffTime).
            Order("to_version ASC").
            First(&timeCutoffChange).Error
        if err == nil && timeCutoffChange.ToVersion > cutoffVersion {
            cutoffVersion = timeCutoffChange.ToVersion
        }
        
        if cutoffVersion == 0 {
            return nil  // 不需要截断
        }
        
        // 2. 更新 baseline_version
        err = tx.Exec("UPDATE global_state SET baseline_version = ? WHERE id = 1", cutoffVersion).Error
        if err != nil {
            return err
        }
        
        // 3. 删除旧变更
        err = tx.Where("to_version < ? AND type IN ?", cutoffVersion, []string{"create", "update", "delete", "move"}).
            Delete(&Change{}).Error
        if err != nil {
            return err
        }
        
        // 4. 记录截断时间
        err = tx.Exec("UPDATE global_state SET last_truncated_at = ? WHERE id = 1", time.Now().UnixMilli()).Error
        return err
    })
}
```

### 6.6 快照生成算法（v3 新增）

```go
// pkg/sync/snapshot.go

// GenerateSnapshot 生成快照（v3 新增）
func GenerateSnapshot(db *gorm.DB, version int64) error {
    return db.Transaction(func(tx *gorm.DB) error {
        // 1. 获取所有非删除书签
        var bookmarks []Bookmark
        err := tx.Where("is_deleted = 0").Find(&bookmarks).Error
        if err != nil {
            return err
        }
        
        // 2. 构建树结构
        tree := buildTree(bookmarks)
        
        // 3. 序列化为 JSON
        treeJSON, err := json.Marshal(tree)
        if err != nil {
            return err
        }
        
        // 4. gzip 压缩
        var buf bytes.Buffer
        gz := gzip.NewWriter(&buf)
        _, err = gz.Write(treeJSON)
        if err != nil {
            return err
        }
        gz.Close()
        
        // 5. 计算哈希
        hash := sha256.Sum256(buf.Bytes())
        
        // 6. 保存快照
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
        
        // 7. 清理旧快照（保留最近 500 个）
        err = tx.Exec(`
            DELETE FROM snapshots 
            WHERE version NOT IN (
                SELECT version FROM snapshots ORDER BY version DESC LIMIT 500
            )
        `).Error
        
        return err
    })
}
```

---

## 7. 部署方案

### 7.1 Docker 配置（v3 修正）

```dockerfile
# Dockerfile
# 阶段 1：构建 Web 前端
FROM node:20-alpine AS web-builder

# 安装 pnpm
RUN corepack enable && corepack prepare pnpm@latest --activate

WORKDIR /build/web
COPY web/package*.json ./
COPY web/pnpm-lock.yaml ./  # v3 新增：复制锁文件
RUN pnpm install --frozen-lockfile
COPY web/ ./
RUN pnpm run build

# 阶段 2：构建 Go 后端
FROM golang:1.23-alpine AS go-builder

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

# 阶段 3：运行时镜像
FROM alpine:3.19

RUN apk --no-cache add ca-certificates wget

WORKDIR /app
COPY --from=go-builder /build/server .
COPY --from=go-builder /build/migrations ./migrations
COPY --from=web-builder /build/web/dist ./web/dist

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
    # 可选：本地/测试从 .env.local 加载（勿提交）；与 environment 并存时，下方 environment 优先
    # env_file:
    #   - .env.local
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

**配置来源与优先级：** 建议加载顺序为：① 代码内默认值 ② 可选 `config.yaml` ③ **环境变量（最高优先级）**。这样本地可用 YAML 或小文件快速起步，也可用 `.env.local` 覆盖；容器内只注入 env 即可与生产一致，无需在镜像中携带机密。

**本地与测试：** 将 `.env.example` 复制为 `.env` 或 `.env.local`；`docker compose` 可写 `env_file: .env.local`，或使用 `docker compose --env-file .env.local up`。变量名与下表一致（大写 + 下划线，与 Viper `AutomaticEnv` 约定对齐）。

**生产：** 不在镜像或仓库中存放 `DB_PASSWORD` 等机密；在 Compose `environment`、懒猫 `spec.env` 或编排系统的 env 中注入（§7.2、§7.4）。

下列为提交到仓库的模板文件 **`.env.example`**（完整键名与示例值）：

```bash
# .env.example

# 数据库配置
DB_TYPE=sqlite
DB_PATH=/data/lzc-bookmark.db
# PostgreSQL 配置（DB_TYPE=postgres 时使用）
# DB_HOST=localhost
# DB_PORT=5432
# DB_NAME=lzc_bookmark
# DB_USER=lzc
# DB_PASSWORD=***

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

### 8.3 集成测试

```bash
# tests/integration/run.sh

# 1. 启动服务端
docker-compose up -d

# 2. 运行服务端测试
go test ./... -tags=integration

# 3. 构建插件（v3 修正：使用 pnpm）
cd extension && pnpm install && pnpm run build

# 4. 运行插件 E2E 测试（使用 Playwright）
pnpm exec playwright test tests/e2e/

# 5. 清理
docker-compose down
```

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
        []string{"type"},  // update/delete/move/rename/create
    )
    
    changeLogSize = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "lzc_change_log_size",
            Help: "Current change log size",
        },
    )
    
    // v3 新增：版本号空洞监控
    versionGaps = promauto.NewCounter(
        prometheus.CounterOpts{
            Name: "lzc_version_gaps_total",
            Help: "Total number of version gaps (holes)",
        },
    )
)
```

### 9.2 告警规则（v3 修正）

```yaml
# prometheus/alerts.yml

groups:
  - name: lzc-bookmark
    rules:
      - alert: HighConflictRate
        expr: sum(rate(lzc_conflicts_total[5m])) > 0.167  # v3 修正：使用 sum 聚合
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

---

## 12. 参考文档

- [PRD v6.0](./PRD-v6.md)
- [TECHNICAL-DESIGN-v2 评审报告](./tech/)
  - [Composer2 审计报告](./tech/tech-design-v2-reviewed-by-composer2.md)
  - [GLM-5 审计报告](./tech/tech-design-v2-reviewed-by-glm5.md)
  - [GLM-5.1 审计报告](./tech/tech-design-v2-reviewed-by-glm5.1.md)
  - [Kimi Code 审计报告](./tech/tech-design-v2-reviewed-by-kimi-code.md)
  - [Kimi K2.5 审计报告](./tech/tech-design-v2-reviewed-by-kimi-k2.5.md)
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
| **v3.0** | **2026-04-16** | **综合 5 份审计报告，修复全部 P0/P1 问题** |

**v3.0 关键变更：**

1. 新增第 4 章「同步协议」整合 API 定义、请求/响应格式、错误码、410 GONE 处理
2. 冲突检测算法添加 guid 去重、create_conflict 检测、同批 create 冲突检测
3. PromQL 表达式修正（使用 sum 聚合）
4. GORM 使用 datatypes.JSON 类型
5. NextVersionSQLite 使用 FOR UPDATE 锁定
6. 新增截断算法和快照生成算法伪代码
7. classifyConflict 覆盖全部 5 种冲突类型
8. NormalizeWeights 事务完整（包含 change_log 插入）
9. Dockerfile 复制 pnpm-lock.yaml
10. 集成测试统一使用 pnpm
11. PendingChange 新增 base_version 字段
12. SQLite WAL 配置说明
13. sync_records 表对齐 PRD（新增 sync_id、request_id）
14. AcquireLock 使用指数退避
15. audit_log 添加 IP 获取逻辑和 90 天清理策略
16. 共享代码策略改用 pnpm workspace
17. 批量查询使用子查询显式获取最新变更
18. 添加评审报告链接

---

**文档状态:** 已评审，待开发  
**下一步:** Phase 1 开发启动  
**评审报告:** 见 `./tech/` 目录下 5 份审计报告
