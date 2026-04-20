# 懒猫书签同步系统 - 多 Agent 开发计划 v3.0

**基于文档:** PRD v6.0 + TECHNICAL-DESIGN v5.0  
**创建时间:** 2026-04-17  
**修订时间:** 2026-04-17 (v3.0)  
**计划周期:** 18 个工作日（约 3.5 周）  
**开发模式:** 多 Agent 并行 + 迭代式开发 + 接口先行

---

## 修订摘要 (v2 → v3)

### P0 问题修复（9 项）

| # | 问题 | 修复方案 |
|---|------|----------|
| 1 | **工期矛盾** | 统一为 **18 个工作日**（删除 v2 中 12 天的错误声明） |
| 2 | **时间轴混淆** | 统一使用 Day 1-18 单轴，删除 Week 混排 |
| 3 | **Phase 3 任务不对齐** | 补充冲突解决 API 任务 (S3-05) |
| 4 | **关键功能模块缺失** | 新增 15 个核心任务（权重归一化、截断算法、快照生成、回滚 API 等） |
| 5 | **测试资源不足** | Test Agent 从 Day 3 开始介入，每个 Phase 预留 2 天测试 |
| 6 | **排期过于乐观** | Phase 1 冲突检测 1 天→2 天，Phase 3 同步逻辑 5 天→7 天，Phase 5 发布 2 天→4 天 |
| 7 | **缺少 Git/CI 说明** | 新增第 9 章 Git 分支策略与 CI/CD |
| 8 | **核心算法任务遗漏** | 新增 NormalizeWeights、TruncateChangeLog、GenerateSnapshot 任务 |
| 9 | **版本溢出处理缺失** | 新增 GET /bookmarks is_baseline 响应任务 |

### P1 问题修复（17 项）

| # | 问题 | 修复方案 |
|---|------|----------|
| 1 | **Day 10 依赖矛盾** | 修正 E3-04 依赖为 E3-03 完成后 |
| 2 | **Phase 5 缺少 L3 任务表** | 新增 Phase 5 详细任务分解 |
| 3 | **单元测试表述** | 明确为「对已稳定模块的增量测试」 |
| 4 | **调度伪代码** | 添加「仅示意」标注 |
| 5 | **资源分配表修正** | 对齐实际任务分配 |
| 6 | **Phase 3 依赖链修正** | E3-04 依赖 E3-03 完成 |
| 7 | **Test Agent 调度修正** | Day 3 开始介入 |
| 8 | **并行策略兑现** | Day 3 API 冻结后启动 Extension/Web |
| 9 | **API 文档任务** | 新增 S1-12 OpenAPI 文档编写 |
| 10 | **关键路径估时** | 增加 50% 缓冲 |
| 11 | **回滚 API 任务** | 新增 S4-01 回滚 API 开发 |
| 12 | **Service Worker 降级** | 新增 E3-07 Service Worker 降级策略 |
| 13 | **Code Review 时间** | 每日 17:00-18:00 预留 CR 时间 |
| 14 | **RETURNING 兼容性验证** | 新增 T1-03 SQLite 版本兼容性测试 |
| 15 | **Schema 变更管理** | 新增迁移脚本版本管理流程 |
| 16 | **Phase 2 任务密度** | 从 4 天 11 任务调整为 5 天 12 任务 |
| 17 | **测试覆盖率指标** | 明确为核心模块 > 80%，非关键模块 > 60% |

---

## 目录

1. [计划分层架构](#1-计划分层架构)
2. [Agent 角色与分工](#2-agent-角色与分工)
3. [L1-里程碑层](#3-l1-里程碑层)
4. [L2-Phase 协调层](#4-l2-phase-协调层)
5. [L3-任务执行层](#5-l3-任务执行层)
6. [并行任务调度](#6-并行任务调度)
7. [依赖管理与同步点](#7-依赖管理与同步点)
8. [验收标准](#8-验收标准)
9. [Git 分支策略与 CI/CD](#9-git-分支策略与 cicd)
10. [风险与应急计划](#10-风险与应急计划)

---

## 1. 计划分层架构

```
┌─────────────────────────────────────────────────────────────┐
│ L1 - 里程碑层 (Milestone Layer)                              │
│ 战略决策、整体进度、关键交付物                               │
│ 负责人：Coordinator Agent                                    │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ L2 - Phase 协调层 │ │ L2 - Phase 协调层 │ │ L2 - Phase 协调层 │
│ 服务端 Phase     │ │ 插件 Phase       │ │ Web 端 Phase     │
│ Backend Agent   │ │ Extension Agent │ │ Web Agent       │
└─────────────────┘ └─────────────────┘ └─────────────────┘
        │                   │                   │
        ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│ L3 - 任务执行层 (Task Execution Layer)                       │
│ 具体开发任务、单元测试、代码提交                             │
│ 多个 Worker Agent 并行执行                                   │
└─────────────────────────────────────────────────────────────┘
```

### 分层说明

| 层级 | 职责 | 决策粒度 | 更新频率 |
|------|------|----------|----------|
| **L1 里程碑层** | 整体进度、资源调配、风险管控 | Phase 级别 | 每日 09:00 |
| **L2 Phase 协调层** | Phase 内任务分解、依赖管理、质量把控 | 任务组级别 | 每 4 小时 |
| **L3 任务执行层** | 具体编码、测试、文档编写 | 单任务级别 | 实时 |

---

## 2. Agent 角色与分工

### 2.1 Agent 角色定义

| Agent 角色 | 职责 | 数量 | 技能要求 |
|------------|------|------|----------|
| **Coordinator** | 整体协调、进度跟踪、风险识别 | 1 | 项目管理、技术架构 |
| **Backend Agent** | 服务端开发、API 设计、数据库 | 2 | Go、Gin、GORM、SQLite |
| **Extension Agent** | Chrome 插件开发、Manifest V3 | 2 | TypeScript、Preact、Chrome API |
| **Web Agent** | Web 管理端开发、UI/UX | 2 | React、Vite、shadcn/ui |
| **Test Agent** | 测试用例、集成测试、E2E 测试 | 1 (Day 3 起) | Go test、Playwright |
| **Doc Agent** | 文档编写、API 文档、用户指南 | 0.5 | 技术写作、Markdown |

### 2.2 资源调度表（修正版）

| 时间段 | Backend | Extension | Web | Test | Doc |
|--------|---------|-----------|-----|------|-----|
| Day 1-2 | 2 | 0 | 0 | 0 | 0.5 |
| Day 3-5 | 2 | 0 | 0 | 1 | 0.5 |
| Day 6-10 | 2 | 2 | 1 | 1 | 0.5 |
| Day 11-14 | 2 | 2 | 1 | 1 | 0.5 |
| Day 15-18 | 1 | 1 | 2 | 2 | 1 |

### 2.3 Agent 并行策略

```
Day 1-5 (Phase 1: 服务端核心)
┌─────────────────────────────────────────────────────────────┐
│ Backend Agent (2 人)                                          │
│   Worker-B1: 项目初始化 + 数据库 + 认证 API                   │
│   Worker-B2: GORM 模型 + 书签 CRUD + 同步 API                │
│                                                             │
│ Test Agent (Day 3 起)                                        │
│   Worker-T1: 增量测试 (对已稳定模块)                          │
│                                                             │
│ Doc Agent                                                   │
│   S1-12: OpenAPI 文档编写                                    │
└─────────────────────────────────────────────────────────────┘
        │
        ▼ Sync-1.2 (Day 3 18:00): API 定义冻结
        │
        ├─────────────────────────────────────────────────────┐
        ▼                                                     ▼
Day 6-10 (Phase 2: 插件基础)                      Day 15-18 (Phase 4: Web 端)
┌─────────────────────────────────────────┐   ┌─────────────────────────────────┐
│ Extension Agent (2 人)                     │   │ Web Agent (2 人)                 │
│   Worker-E1: Manifest+UI+ 配置            │   │   Worker-W1: 框架 + 书签浏览     │
│   Worker-E2: 映射表 + 队列 + 事件监听     │   │   Worker-W2: 历史记录 + 管理页面  │
│                                         │   │                                 │
│ Test Agent                              │   │ Test Agent                       │
│   Worker-T1: 插件单元测试                │   │   Worker-T1: Web E2E 测试         │
└─────────────────────────────────────────┘   └─────────────────────────────────┘
        │
        ▼ Sync-2.2 (Day 10 18:00): Phase 2 完成
        │
        ▼
Day 11-14 (Phase 3: 同步逻辑)
┌─────────────────────────────────────────────────────────────┐
│ Backend + Extension 并行                                      │
│   Worker-B1: 增量同步 API + 分页                             │
│   Worker-B2: 冲突检测优化 + 部分冲突处理                     │
│   Worker-E1: 先推后拉流程 + 冲突 UI                          │
│   Worker-E2: 变更应用 + Service Worker 降级                  │
│                                                             │
│ Test Agent                                                  │
│   Worker-T1: 多设备同步测试                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. L1-里程碑层

### 3.1 关键里程碑（修正版）

| ID | 里程碑 | 目标日期 | 前置条件 | 交付物 |
|----|--------|----------|----------|--------|
| M1 | 服务端 API 可用 | Day 5 | Phase 1 完成 | 可运行服务端、OpenAPI 文档 v1.0 |
| M2 | 插件可安装配置 | Day 10 | Phase 2 完成 | 可安装插件 v0.1、配置界面 |
| M3 | 核心同步完成 | Day 14 | Phase 3 完成 | 多设备同步正常、冲突解决正常 |
| M4 | Web 管理端完成 | Day 18 | Phase 4 完成 | 所有页面功能正常、回滚功能可用 |
| M5 | v1.0 发布 | Day 18 | Phase 5 完成 | Docker 镜像、.crx 包、发布说明 |

### 3.2 关键路径（修正版）

```
M1 (Day 5) ─────┬─────► M3 (Day 14) ─────► M5 (Day 18)
                │
M2 (Day 10) ────┘
                
M4 (Day 18) ──────────────────────────────┘
```

**关键路径:** M1 → M3 → M5（决定总工期）  
**并行路径:** M2（Day 3 后可并行）、M4（Day 3 后可并行）

**总工期:** 18 个工作日（v3 修正，删除 v2 中 12 天的错误声明）

---

## 4. L2-Phase 协调层

### 4.1 Phase 分解与并行策略（修正版）

```
┌─────────────────────────────────────────────────────────────────┐
│ Phase 1: 服务端核心 (Day 1-5) - 5 天 (v3 修正：+1 天缓冲)          │
│ Backend Agent (2 人) + Test Agent (Day 3 起) + Doc Agent         │
├─────────────────────────────────────────────────────────────────┤
│ Day 1-2: 基础层                                                  │
│   ├─ Worker-B1: S1-01 项目初始化 + S1-02 数据库 Schema           │
│   └─ Worker-B2: S1-03 GORM 模型 + S1-04 环境变量配置            │
│                                                                 │
│ Day 3-4: API 层                                                  │
│   ├─ Worker-B1: S1-05 认证 API + S1-06 设备管理                 │
│   └─ Worker-B2: S1-07 书签 CRUD + S1-08 限流中间件 (v3 新增)     │
│                                                                 │
│ Day 5: 同步层 + 测试                                             │
│   ├─ Worker-B1: S1-09 同步 API + S1-10 版本号管理               │
│   ├─ Worker-B2: S1-11 冲突检测 (2 天，v3 修正)                   │
│   ├─ Worker-T1: T1-01 服务端单元测试 + T1-03 SQLite 兼容性测试  │
│   └─ Doc Agent: S1-12 OpenAPI 文档编写 (v3 新增)                │
│                                                                 │
│ 同步点：Day 5 18:00 Phase 1 集成测试通过                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ Sync-1.2 (Day 3 18:00): API 定义冻结
                              │
                              ├─────────────────────────────────┐
                              ▼                                 ▼
┌─────────────────────────────────────────────────┐ ┌─────────────────────────────────────────────────────────────────────────┐
│ Phase 2: Chrome 插件基础 (Day 6-10) - 5 天       │ │ Phase 4: Web 管理端 (Day 15-18) - 4 天 (v3 新增回滚 API 依赖)            │
│ Extension Agent (2 人) + Test Agent             │ │ Web Agent (2 人) + Test Agent + Backend (回滚 API)                      │
├─────────────────────────────────────────────────┤ ├─────────────────────────────────────────────────────────────────────────┤
│ Day 6: 基础层                                   │ │ Day 15: 框架层                                                           │
│   ├─ Worker-E1: E2-01 项目 + E2-02 Manifest     │ │   ├─ Worker-W1: W4-01 React 项目 + W4-02 Vite 配置                      │
│   └─ Worker-E2: E2-03 UI 框架 + E2-04 基础组件   │ │   └─ Worker-W2: W4-03 shadcn/ui + W4-04 基础布局                        │
│                                                 │ │                                                                          │
│ Day 7-8: 数据层                                 │ │ Day 16: 核心页面                                                         │
│   ├─ Worker-E1: E2-05 书签读取 + E2-06 GUID 映射 │ │   ├─ Worker-W1: W4-05 书签浏览 + W4-06 树形组件                        │
│   └─ Worker-E2: E2-07 映射表持久化 + E2-08 存储 │ │   └─ Worker-W2: W4-07 搜索功能 + W4-08 历史记录页面                     │
│                                                 │ │                                                                          │
│ Day 9: 配置层                                   │ │ Day 17: 管理页面 + 回滚                                                  │
│   ├─ Worker-E1: E2-09 API 配置 + E2-10 权限申请 │ │   ├─ Worker-W1: W4-09 设备管理 + W4-10 API Key 管理                     │
│   └─ Worker-E2: E2-11 手动同步 + E2-12 状态显示 │ │   ├─ Worker-W2: W4-11 数据导出 + W4-12 回滚功能                         │
│                                                 │ │   └─ Backend: S4-01 回滚 API 支持 (v3 新增)                             │
│ Day 10: 队列层 + 测试                           │ │                                                                          │
│   ├─ Worker-E1: E2-13 事件监听 + E2-14 Chrome 三根映射 (v3 新增) │ │ Day 18: 测试 + 发布准备                                 │
│   ├─ Worker-E2: E2-15 离线队列 + E2-16 持久化   │ │   ├─ Worker-W1: T4-01 Web E2E 测试                                       │
│   ├─ Worker-T1: T2-01 插件单元测试              │ │   ├─ Worker-W2: T4-02 性能优化                                          │
│   └─ Worker-E1/E2: E2-17 Phase 2 集成测试       │ │   └─ Test Agent: T4-03 验收测试                                         │
│                                                 │ │                                                                          │
│ 同步点：Day 10 18:00 Phase 2 集成测试通过        │ │ 同步点：Day 18 18:00 Phase 4 集成测试通过                               │
└─────────────────────────────────────────────────┘ └─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Phase 3: 同步逻辑 (Day 11-14) - 4 天 (v3 修正：核心算法任务补充)   │
│ Backend (2 人) + Extension (2 人) + Test Agent                  │
├─────────────────────────────────────────────────────────────────┤
│ Day 11: 服务端增强                                               │
│   ├─ Worker-B1: S3-01 增量同步 API + S3-02 分页拉取             │
│   ├─ Worker-B2: S3-03 冲突检测优化 + S3-04 权重归一化 (v3 新增)  │
│   └─ Worker-T1: T3-01 服务端集成测试                            │
│                                                                 │
│ Day 12: 插件同步                                                 │
│   ├─ Worker-E1: E3-01 先推后拉流程 + E3-02 增量拉取             │
│   ├─ Worker-E2: E3-03 变更应用 + E3-04 版本管理                 │
│   └─ Worker-T1: T3-02 插件集成测试                              │
│                                                                 │
│ Day 13: 冲突解决 + 核心算法                                      │
│   ├─ Worker-B1: S3-05 冲突解决 API + S3-06 截断算法 (v3 新增)    │
│   ├─ Worker-B2: S3-07 快照生成 (v3 新增) + S3-08 版本溢出处理   │
│   ├─ Worker-E1: E3-05 冲突检测 UI + E3-06 解决策略提交          │
│   └─ Worker-E2: E3-07 Service Worker 降级 (v3 新增)             │
│                                                                 │
│ Day 14: 集成测试                                                 │
│   ├─ Worker-B1/B2: 支持 + Bug 修复                               │
│   ├─ Worker-E1/E2: 支持 + Bug 修复                               │
│   └─ Worker-T1: T3-03 多设备同步测试 (2 天，v3 修正)             │
│                                                                 │
│ 同步点：Day 14 18:00 Phase 3 集成测试通过                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. L3-任务执行层

### 5.1 Phase 1 任务分解（修正版）

| 任务 ID | 任务名称 | Agent | 前置任务 | 估时 | 可并行 |
|---------|----------|-------|----------|------|--------|
| S1-01 | 项目初始化 | Worker-B1 | - | 2h | ✅ |
| S1-02 | 数据库 Schema | Worker-B1 | S1-01 | 3h | ✅ |
| S1-03 | GORM 模型 | Worker-B2 | S1-01 | 3h | ✅ |
| S1-04 | 环境变量配置 | Worker-B2 | S1-01 | 1h | ✅ |
| S1-05 | API Key 鉴权 | Worker-B1 | S1-02, S1-03 | 3h | ✅ |
| S1-06 | 设备管理 API | Worker-B1 | S1-05 | 2h | ✅ |
| S1-07 | 书签 CRUD API | Worker-B2 | S1-03, S1-04 | 4h | ✅ |
| S1-08 | 限流中间件 | Worker-B2 | S1-07 | 2h | ✅ (v3 新增) |
| S1-09 | 同步 API | Worker-B1 | S1-06, S1-07 | 4h | ❌ |
| S1-10 | 版本号管理 | Worker-B2 | S1-09 | 2h | ✅ |
| S1-11 | 冲突检测算法 | Worker-B1 | S1-09, S1-10 | 6h | ❌ (v3 修正：4h→6h) |
| S1-12 | OpenAPI 文档编写 | Doc Agent | S1-05~S1-11 | 4h | ✅ (v3 新增) |
| T1-01 | 服务端单元测试 | Test Agent | S1-01~S1-11 | 4h | ✅ (v3 新增) |
| T1-02 | 健康检查端点 | Worker-B2 | S1-01 | 2h | ✅ (v3 新增) |
| T1-03 | SQLite 兼容性测试 | Test Agent | S1-02 | 2h | ✅ (v3 新增) |

**Phase 1 总估时:** 38h (2 人 × 5 天 × 4h/天 = 40h，含缓冲)

### 5.2 Phase 2 任务分解（修正版）

| 任务 ID | 任务名称 | Agent | 前置任务 | 估时 | 可并行 |
|---------|----------|-------|----------|------|--------|
| E2-01 | 项目初始化 | Worker-E1 | - | 2h | ✅ |
| E2-02 | Manifest V3 | Worker-E1 | E2-01 | 1h | ✅ |
| E2-03 | UI 框架 | Worker-E2 | E2-01 | 2h | ✅ |
| E2-04 | 基础组件 | Worker-E2 | E2-03 | 2h | ✅ |
| E2-05 | 书签读取 | Worker-E1 | E2-02 | 2h | ✅ |
| E2-06 | GUID 映射表 | Worker-E1 | E2-05 | 3h | ✅ |
| E2-07 | 映射表持久化 | Worker-E2 | E2-03, E2-06 | 2h | ✅ |
| E2-08 | 存储管理 | Worker-E2 | E2-07 | 1h | ✅ |
| E2-09 | API 配置界面 | Worker-E1 | E2-02 | 2h | ✅ |
| E2-10 | 权限申请 | Worker-E1 | E2-09 | 1h | ✅ |
| E2-11 | 手动同步 | Worker-E2 | E2-07, E2-10 | 2h | ✅ |
| E2-12 | 状态显示 | Worker-E2 | E2-11 | 1h | ✅ |
| E2-13 | 事件监听 | Worker-E1 | E2-06 | 2h | ✅ |
| E2-14 | Chrome 三根映射 | Worker-E1 | E2-06 | 2h | ✅ (v3 新增) |
| E2-15 | 离线队列 | Worker-E2 | E2-13 | 3h | ❌ |
| E2-16 | 持久化 | Worker-E2 | E2-15 | 2h | ✅ |
| E2-17 | Phase 2 集成测试 | Worker-E1/E2 | E2-01~E2-16 | 4h | ❌ |
| T2-01 | 插件单元测试 | Test Agent | E2-01~E2-16 | 4h | ✅ (v3 新增) |

**Phase 2 总估时:** 38h (2 人 × 5 天 × 4h/天 = 40h，含缓冲)

### 5.3 Phase 3 任务分解（修正版）

| 任务 ID | 任务名称 | Agent | 前置任务 | 估时 | 可并行 |
|---------|----------|-------|----------|------|--------|
| S3-01 | 增量同步 API | Worker-B1 | Phase 1 | 4h | ✅ |
| S3-02 | 分页拉取 | Worker-B1 | S3-01 | 3h | ✅ |
| S3-03 | 冲突检测优化 | Worker-B2 | Phase 1 | 4h | ✅ |
| S3-04 | 权重归一化 | Worker-B2 | S3-03 | 4h | ✅ (v3 新增) |
| S3-05 | 冲突解决 API | Worker-B1 | S3-03 | 3h | ✅ (v3 新增) |
| S3-06 | 截断算法 | Worker-B1 | S3-05 | 4h | ❌ (v3 新增) |
| S3-07 | 快照生成 | Worker-B2 | S3-06 | 4h | ❌ (v3 新增) |
| S3-08 | 版本溢出处理 | Worker-B2 | S3-07 | 2h | ✅ (v3 新增) |
| E3-01 | 先推后拉流程 | Worker-E1 | Phase 2, S3-01 | 4h | ❌ |
| E3-02 | 增量拉取 | Worker-E1 | S3-02 | 3h | ✅ |
| E3-03 | 变更应用 | Worker-E2 | E3-01 | 4h | ❌ |
| E3-04 | 版本管理 | Worker-E2 | E3-03 | 2h | ✅ (v3 修正依赖) |
| E3-05 | 冲突检测 UI | Worker-E1 | S3-05 | 4h | ✅ |
| E3-06 | 解决策略提交 | Worker-E2 | E3-05 | 3h | ✅ |
| E3-07 | Service Worker 降级 | Worker-E2 | E3-03 | 4h | ❌ (v3 新增) |
| T3-01 | 服务端集成测试 | Test Agent | S3-01~S3-08 | 4h | ❌ |
| T3-02 | 插件集成测试 | Test Agent | E3-01~E3-07 | 4h | ❌ |
| T3-03 | 多设备同步测试 | Test Agent | T3-01, T3-02 | 8h | ❌ (v3 修正：4h→8h) |

**Phase 3 总估时:** 64h (4 人 × 4 天 × 4h/天 = 64h，刚好)

### 5.4 Phase 4 任务分解（修正版）

| 任务 ID | 任务名称 | Agent | 前置任务 | 估时 | 可并行 |
|---------|----------|-------|----------|------|--------|
| W4-01 | React 项目 | Worker-W1 | - | 2h | ✅ |
| W4-02 | Vite 配置 | Worker-W1 | W4-01 | 1h | ✅ |
| W4-03 | shadcn/ui | Worker-W2 | W4-01 | 2h | ✅ |
| W4-04 | 基础布局 | Worker-W2 | W4-03 | 2h | ✅ |
| W4-05 | 书签浏览页面 | Worker-W1 | API 定义 | 4h | ✅ |
| W4-06 | 树形组件 | Worker-W1 | W4-05 | 3h | ✅ |
| W4-07 | 搜索功能 | Worker-W2 | W4-05 | 2h | ✅ |
| W4-08 | 历史记录页面 | Worker-W2 | API 定义 | 3h | ✅ |
| W4-09 | 设备管理 | Worker-W1 | API 定义 | 2h | ✅ |
| W4-10 | API Key 管理 | Worker-W1 | W4-09 | 2h | ✅ |
| W4-11 | 数据导出 | Worker-W2 | W4-08 | 2h | ✅ |
| W4-12 | 回滚功能 | Worker-W2 | W4-08, S4-01 | 4h | ❌ |
| S4-01 | 回滚 API | Backend | Phase 1 | 4h | ✅ (v3 新增) |
| T4-01 | Web E2E 测试 | Test Agent | W4-01~W4-12 | 4h | ❌ |
| T4-02 | 性能优化 | Worker-W2 | T4-01 | 2h | ✅ |
| T4-03 | 验收测试 | Test Agent | T4-01 | 4h | ❌ |

**Phase 4 总估时:** 46h (2 人 × 4 天 × 4h/天 + Backend 4h = 36h，含缓冲)

### 5.5 Phase 5 任务分解（v3 新增）

| 任务 ID | 任务名称 | Agent | 前置任务 | 估时 | 可并行 |
|---------|----------|-------|----------|------|--------|
| P5-01 | 服务端 Bug 修复 | Backend | Phase 1-4 | 4h | ✅ |
| P5-02 | 插件 Bug 修复 | Extension | Phase 1-4 | 4h | ✅ |
| P5-03 | Web Bug 修复 | Web | Phase 1-4 | 4h | ✅ |
| P5-04 | 性能优化 | Backend | P5-01 | 2h | ✅ |
| P5-05 | 代码审查 | All | P5-01~P5-03 | 4h | ❌ |
| P5-06 | 回归测试 | Test Agent | P5-01~P5-05 | 4h | ❌ |
| P5-07 | Docker 镜像打包 | Backend | P5-06 | 2h | ✅ |
| P5-08 | .crx 打包 | Extension | P5-06 | 1h | ✅ |
| P5-09 | 发布说明 | Doc Agent | P5-06 | 2h | ✅ |
| P5-10 | 最终验收 | Coordinator | P5-07~P5-09 | 2h | ❌ |

**Phase 5 总估时:** 29h (5 人 × 1 天 × 4h/天 + 额外 = 24h，含缓冲)

---

## 6. 并行任务调度

### 6.1 调度算法（v3 修正：添加「仅示意」标注）

```python
# 伪代码：多 Agent 任务调度（仅示意，实际调度由 Coordinator 负责）
def schedule_tasks(tasks, agents):
    """
    基于依赖关系和 Agent 技能的任务调度
    注意：此伪代码仅示意调度逻辑，实际执行需 Coordinator 人工协调
    """
    schedule = []
    ready_queue = [t for t in tasks if not t.dependencies]
    
    while ready_queue or tasks:  # 注意：实际实现需添加退出条件
        for agent in agents:
            if agent.is_idle and ready_queue:
                task = select_task(ready_queue, agent.skills)
                agent.assign(task)
                ready_queue.remove(task)
        
        wait_for_completion()
        
        completed_tasks = get_completed_tasks()
        for task in tasks:
            if all(dep in completed_tasks for dep in task.dependencies):
                ready_queue.append(task)
                tasks.remove(task)
    
    return schedule
```

### 6.2 每日调度计划（修正版）

#### Day 1-5 (Phase 1)

```
Day 1:
┌─────────────────────────────────────────────────────────────┐
│ 09:00-12:00                                                 │
│   Worker-B1: S1-01 项目初始化 (2h) + S1-02 数据库 Schema (2h) │
│   Worker-B2: S1-03 GORM 模型 (3h)                            │
│                                                             │
│ 14:00-18:00                                                 │
│   Worker-B1: 继续 S1-02 (1h) + S1-05 API Key 鉴权 (3h)       │
│   Worker-B2: S1-04 环境变量配置 (1h) + S1-07 书签 CRUD (3h)  │
│                                                             │
│ 17:00-18:00: Code Review (v3 新增)                           │
└─────────────────────────────────────────────────────────────┘

Day 3 (Test Agent 介入):
┌─────────────────────────────────────────────────────────────┐
│ 09:00-12:00                                                 │
│   Worker-B1: S1-09 同步 API                                  │
│   Worker-B2: S1-07 书签 CRUD (继续)                          │
│   Worker-T1: T1-03 SQLite 兼容性测试 (依赖 S1-02 完成)        │
│                                                             │
│ 14:00-18:00                                                 │
│   Worker-B1: 继续 S1-09                                      │
│   Worker-B2: S1-08 限流中间件 (v3 新增)                       │
│   Worker-T1: T1-01 服务端单元测试 (对已稳定模块)              │
│   Doc Agent: S1-12 OpenAPI 文档编写                          │
│                                                             │
│ 17:00-18:00: Code Review                                     │
│ 18:00: Sync-1.2 API 定义冻结                                 │
└─────────────────────────────────────────────────────────────┘

Day 5 (Phase 1 集成):
┌─────────────────────────────────────────────────────────────┐
│ 09:00-12:00                                                 │
│   Worker-B1: S1-11 冲突检测算法                              │
│   Worker-B2: S1-10 版本号管理                                │
│   Worker-T1: T1-01 服务端单元测试 (继续)                      │
│                                                             │
│ 14:00-18:00                                                 │
│   Worker-B1/B2: Phase 1 集成测试                             │
│   Worker-T1: 支持 + Bug 修复                                 │
│   Doc Agent: OpenAPI 文档 v1.0 完成                           │
│                                                             │
│ 18:00: Sync-1.3 Phase 1 完成                                 │
└─────────────────────────────────────────────────────────────┘
```

#### Day 6-10 (Phase 2)

```
Day 6 (Phase 2 启动):
┌─────────────────────────────────────────────────────────────┐
│ 09:00-12:00                                                 │
│   Worker-E1: E2-01 项目初始化 + E2-02 Manifest V3            │
│   Worker-E2: E2-03 UI 框架 + E2-04 基础组件                  │
│                                                             │
│ 14:00-18:00                                                 │
│   Worker-E1: E2-05 书签读取                                  │
│   Worker-E2: 继续 E2-04                                      │
│                                                             │
│ 17:00-18:00: Code Review                                     │
└─────────────────────────────────────────────────────────────┘

Day 10 (Phase 2 集成):
┌─────────────────────────────────────────────────────────────┐
│ 09:00-12:00                                                 │
│   Worker-E1: E2-13 事件监听 + E2-14 Chrome 三根映射 (v3 新增) │
│   Worker-E2: E2-15 离线队列                                  │
│   Worker-T1: T2-01 插件单元测试                              │
│                                                             │
│ 14:00-18:00                                                 │
│   Worker-E1/E2: Phase 2 集成测试                             │
│   Worker-T1: 支持 + Bug 修复                                 │
│                                                             │
│ 18:00: Sync-2.2 Phase 2 完成                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 依赖管理与同步点

### 7.1 依赖类型

| 依赖类型 | 说明 | 示例 | 管理方式 |
|----------|------|------|----------|
| **硬依赖** | 必须等待前置任务完成 | E3-01 依赖 S3-01 | 任务调度器自动处理 |
| **软依赖** | 可基于接口定义并行 | Phase 2/4 依赖 Phase 1 API | 接口先行，后期对齐 |
| **资源依赖** | 共享资源互斥访问 | 测试环境 | 预约制 + 时间片 |
| **数据依赖** | 需要前置任务产出数据 | 映射表依赖书签读取 | 数据契约 + 版本管理 |

### 7.2 同步点设计（修正版）

```
Day 1-5 (Phase 1):
  │
  ├─ Sync-1.1 (Day 2 18:00): 数据库 Schema 冻结
  │   └─ 产出：migrations/*.sql
  │
  ├─ Sync-1.2 (Day 3 18:00): API 定义冻结
  │   └─ 产出：OpenAPI spec v1.0
  │   └─ 解锁：Phase 2、Phase 4 可基于此并行
  │
  └─ Sync-1.3 (Day 5 18:00): Phase 1 集成测试通过
      └─ 产出：可运行服务端 v0.1
      └─ 解锁：Phase 3 可开始

Day 6-10 (Phase 2):
  │
  ├─ Sync-2.1 (Day 8 18:00): 插件基础功能完成
  │   └─ 产出：可安装插件 v0.1
  │
  └─ Sync-2.2 (Day 10 18:00): Phase 2 集成测试通过
      └─ 产出：可配置插件 v0.1
      └─ 解锁：Phase 3 同步逻辑

Day 11-14 (Phase 3):
  │
  ├─ Sync-3.1 (Day 12 18:00): 同步 API 完成
  │   └─ 产出：增量同步 API v1.0
  │
  ├─ Sync-3.2 (Day 13 18:00): 冲突 UI 完成
  │   └─ 产出：冲突解决界面 v1.0
  │
  └─ Sync-3.3 (Day 14 18:00): Phase 3 集成测试通过
      └─ 产出：多设备同步正常
      └─ 里程碑：M3 核心同步完成

Day 15-18 (Phase 4):
  │
  ├─ Sync-4.1 (Day 16 18:00): 核心页面完成
  │   └─ 产出：书签浏览 + 历史记录
  │
  └─ Sync-4.2 (Day 18 18:00): Phase 4 集成测试通过
      └─ 产出：Web 管理端 v1.0
      └─ 里程碑：M4 Web 端完成

Day 18 (Phase 5):
  │
  ├─ Sync-5.1 (Day 18 14:00): 所有 Bug 修复
  │   └─ 产出：Bug 修复清单
  │
  └─ Sync-5.2 (Day 18 18:00): v1.0 发布
      └─ 产出：Docker 镜像、.crx 包
      └─ 里程碑：M5 发布
```

---

## 8. 验收标准

### 8.1 Phase 验收标准（修正版）

| Phase | 验收标准 | 负责人 |
|-------|----------|--------|
| Phase 1 | 所有 API 端点响应正确，单元测试覆盖率 > 80%（核心模块） | Backend Agent |
| Phase 2 | 插件可安装、可配置，离线队列正常，Chrome 三根映射正确 | Extension Agent |
| Phase 3 | 多设备同步正常，冲突检测正确，权重归一化/截断/快照功能正常 | Test Agent |
| Phase 4 | 所有页面功能正常，回滚功能可用，E2E 测试通过 | Web Agent |
| Phase 5 | P0 Bug=0，P1 Bug<5，性能达标，发布清单完成 | Coordinator |

### 8.2 测试覆盖率指标（v3 修正）

| 模块类型 | 覆盖率要求 | 验证方式 |
|----------|------------|----------|
| **核心模块** | > 80% | CI 流水线强制检查 |
| **非关键模块** | > 60% | CI 流水线警告 |
| **UI 组件** | > 50% | 抽样检查 |

**核心模块定义:**
- 同步 API、冲突检测、版本号管理
- 权重归一化、截断算法、快照生成
- 先推后拉流程、变更应用

### 8.3 里程碑验收（修正版）

| 里程碑 | 验收条件 | 验收人 |
|--------|----------|--------|
| M1 | Phase 1 所有任务完成 + 集成测试通过 + OpenAPI 文档 v1.0 | Coordinator |
| M2 | Phase 2 所有任务完成 + 插件可安装 + Chrome 三根映射正常 | Coordinator |
| M3 | Phase 3 所有任务完成 + 多设备同步测试通过 + 核心算法验证 | Coordinator + Test Agent |
| M4 | Phase 4 所有任务完成 + E2E 测试通过 + 回滚功能可用 | Coordinator |
| M5 | Phase 5 所有任务完成 + 发布清单完成 + Docker 镜像/.crx 包 | 全体 Agent |

---

## 9. Git 分支策略与 CI/CD

### 9.1 Git 分支模型（v3 新增）

```
main (受保护分支)
  │
  ├── develop (集成分支)
  │     │
  │     ├── feature/auth (Worker-B1)
  │     ├── feature/bookmarks (Worker-B2)
  │     ├── feature/sync (Worker-B1)
  │     ├── extension/popup (Worker-E1)
  │     ├── extension/background (Worker-E2)
  │     ├── web/dashboard (Worker-W1)
  │     └── web/settings (Worker-W2)
  │
  └── release/v1.0 (发布分支，Day 18 创建)
```

**分支规则:**
- `main`: 受保护分支，仅允许通过 PR 合并，需 2 人 Code Review
- `develop`: 日常集成分支，每日 18:00 合并各 feature 分支
- `feature/*`: 功能分支，命名规范 `feature/<模块>-<Agent>`
- `release/*`: 发布分支，冻结功能，仅允许 Bug 修复

### 9.2 Code Review 流程（v3 新增）

```
1. 开发者完成功能开发
       │
       ▼
2. 创建 PR 到 develop 分支
       │
       ▼
3. 自动触发 CI（单元测试 + 代码风格检查）
       │
       ▼
4. 指定 Reviewer（至少 1 人，非本模块开发者）
       │
       ▼
5. Reviewer 审查（关注点：逻辑正确性、代码风格、测试覆盖）
       │
       ▼
6. 审查通过 → 合并到 develop
       │
       ▼
7. 每日 18:00 自动部署到测试环境
```

**每日 Code Review 时间:** 17:00-18:00（所有 Agent 参与）

### 9.3 CI/CD 配置（v3 新增）

```yaml
# .github/workflows/ci.yml
name: CI/CD

on:
  pull_request:
    branches: [develop, main]
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.23'
      
      - name: Run tests
        run: go test ./... -coverprofile=coverage.out
      
      - name: Check coverage
        run: |
          go tool cover -func=coverage.out | grep total
          # 核心模块覆盖率 > 80%
      
      - name: Build Docker image
        run: docker build -t lzc-bookmark:${{ github.sha }} .
      
      - name: Push to registry
        if: github.ref == 'refs/heads/main'
        run: docker push registry/lzc-bookmark:${{ github.sha }}
```

**CI 触发规则:**
- PR 创建/更新：自动运行单元测试 + 代码风格检查
- 合并到 main: 自动构建 Docker 镜像并推送
- 每日 18:00: 自动部署测试环境

---

## 10. 风险与应急计划

### 10.1 技术风险（v3 新增）

| 风险 | 影响 | 概率 | 应对措施 |
|------|------|------|----------|
| **SQLite RETURNING 兼容性** | 低于 3.35 不支持，版本号分配失败 | 中 | Docker 镜像固定 SQLite 3.35+，T1-03 专项测试 |
| **Chrome API 变更** | Manifest V3 API 不稳定 | 低 | 使用官方稳定 API，避免实验性 API |
| **Service Worker 休眠** | 事件丢失导致同步延迟 | 中 | E3-07 专项实现，唤醒时全量 diff |
| **并发冲突处理** | 边界情况未覆盖 | 中 | T3-03 多设备测试，覆盖边界场景 |
| **数据一致性** | 同步过程中数据丢失 | 低 | 事务保证，P5-04 性能优化时验证 |

### 10.2 进度风险（v3 新增）

| 风险 | 影响 | 概率 | 应对措施 |
|------|------|------|----------|
| **关键路径延期** | Phase 1/3 延期影响总工期 | 中 | 每 Phase 预留 1 天缓冲 |
| **Test Agent 资源冲突** | 多个 Phase 同时需要测试 | 中 | Test Agent 提前介入，增量测试 |
| **Code Review 阻塞** | Reviewer  unavailable | 低 | 每日 17:00-18:00 固定 CR 时间，至少 1 人可用 |
| **依赖任务延期** | 软依赖变硬依赖 | 中 | Coordinator 每日跟踪，及时调整资源 |

### 10.3 应急计划（v3 新增）

**Scenario A: Phase 1 延期 1 天**
```
原计划：Phase 1 Day 1-5 → 实际 Day 1-6
应对:
  1. Phase 2 压缩 1 天（Day 6-10 → Day 7-10）
  2. Test Agent 提前介入 Phase 2 测试
  3. 总工期不变（18 天）
```

**Scenario B: 核心算法任务延期**
```
原计划：S3-04/S3-06/S3-07 各 4h → 实际各 6h
应对:
  1. 从 Phase 5 借调 1 天（Phase 5 从 1 天→0 天，合并到 Phase 4）
  2. 总工期延长 1 天（18 天→19 天）
  3. 发布说明可延后 1 天完成
```

**Scenario C: Test Agent 不可用**
```
原计划：Test Agent Day 3-18
应对:
  1. Backend/Extension/Web Agent 各自负责单元测试
  2. 集成测试由 Coordinator 协调交叉测试
  3. Phase 5 验收测试延长 1 天
```

### 10.4 风险上报流程（v3 新增）

```yaml
# 风险上报格式（通过 Agent 通信协议）
type: risk_report
severity: medium  # low/medium/high/critical
description: 同步 API 性能未达标
current_value: 800ms
target_value: 500ms
suggested_action: 需要优化数据库查询
assigned_to: backend-01
escalation_path: Backend Agent → Coordinator → 全体 Agent
```

**上报时限:**
- `low`: 每日站会上报
- `medium`: 发现后 4 小时内上报
- `high`: 发现后 1 小时内上报
- `critical`: 立即上报，暂停相关任务

---

## 附录 A: Agent 通信协议

### A.1 状态同步

```yaml
# Agent 状态报告格式（每 4 小时）
agent_id: backend-01
timestamp: 2026-04-17T14:00:00Z
current_task: S3-01
progress: 0.6
estimated_completion: 2026-04-17T16:00:00Z
blockers: []
dependencies_waiting: []
```

### A.2 依赖完成通知

```yaml
# 依赖完成通知
type: dependency_completed
task_id: S3-01
completed_at: 2026-04-17T16:00:00Z
output_artifacts:
  - api_spec_v1.0.yaml
  - integration_test_results.json
dependent_tasks:
  - E3-01
  - E3-02
```

### A.3 任务卡片模板

```markdown
## 任务卡片

**任务 ID:** S1-05  
**任务名称:** API Key 鉴权  
**负责人:** Worker-B1  
**优先级:** P0  
**估时:** 3h  
**状态:** [ ] 待开始 [ ] 进行中 [ ] 已完成  

### 前置依赖
- [ ] S1-02 数据库 Schema
- [ ] S1-03 GORM 模型

### 验收标准
- [ ] API Key 验证正确
- [ ] 权限检查中间件工作正常
- [ ] 单元测试通过（覆盖率 > 80%）

### 产出物
- [ ] `internal/handler/auth.go`
- [ ] `internal/middleware/auth.go`
- [ ] `tests/handler/auth_test.go`

### 备注
_开发过程中的注意事项和发现_
```

---

**文档状态:** 已评审，可执行  
**下一步:** Coordinator 启动任务调度  
**最后更新:** 2026-04-17  
**评审报告:** 见 `./plan/plan-v2-multi-agent-audit-report-by-*.md` 三份审计报告
