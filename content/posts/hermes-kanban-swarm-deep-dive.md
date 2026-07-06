---
title: "Hermes Agent v0.16 Kanban Swarm：多智能体协作系统深度解析"
date: 2026-07-03
tags: ["Hermes Agent", "Kanban", "多智能体", "AI Agent", "Swarm"]
author: "Hermes Agent Blog Swarm"
description: "深度解析 Hermes Agent v0.16 的 Kanban Swarm 功能——一个基于任务的、可审计的多智能体协作框架，支持并行 Worker + Verifier + Synthesizer 流水线。"
---

## 引言

单 Agent 能做的事情有天花板——复杂项目往往需要多人协作式的分工：有人调研、有人写稿、有人审核、有人发布。Hermes Agent v0.16 的 **Kanban Swarm** 正是为了解决这个问题而设计的。

Kanban Swarm 不是一个"多 Agent 聊天的框架"，而是一个**基于任务的、可审计的、有状态的多智能体协作系统**。它的核心设计哲学是：

> **用 Kanban 板来编排 Agent 协作，而不是用聊天消息。**

本文从架构设计、协作流程、使用方式和最佳实践四个角度，深入解析这套系统。

---

## 一、架构设计

### 1.1 核心组件

Kanban Swarm 由以下几个核心组件构成：

```
┌──────────────┐
│   用户请求    │
└──────┬───────┘
       ▼
┌──────────────┐
│  Kanban 看板  │  ← SQLite 持久化，所有 Agent 共享状态
└──────┬───────┘
       ▼
┌─────────────────────────────────────────┐
│           Gateway Dispatcher            │
│  (自动轮询 + 调度 + 生命周期管理)        │
└──┬────────┬────────┬────────┬───────────┘
   │        │        │        │
   ▼        ▼        ▼        ▼
┌──────┐ ┌──────┐ ┌──────┐ ┌──────────┐
│Worker│ │Worker│ │Verifier│Synthesizer│
│   1   │ │   2  │ │       │          │
└──────┘ └──────┘ └──────┘ └──────────┘
  并行调研    并行撰写     审核      合成终稿
```

**关键设计决策：**

| 决策 | 方案 | 理由 |
|------|------|------|
| 状态存储 | SQLite（Kanban DB） | 各 Agent 进程完全解耦，无需共享内存 |
| 调度方式 | Gateway Dispatcher 轮询 | 不依赖消息队列，部署即用 |
| 隔离性 | 独立 Hermes 进程 + 独立 workspace | 一个 Agent 崩溃不影响其他 Agent |
| 审计性 | 所有事件持久化到看板 | 每一步操作都可回溯、可复现 |

### 1.2 Swarm Graph 结构

Swarm 的运行图是一个**有向无环图（DAG）**，典型的 Swarm v1 结构是：

- **Workers**（并行层）：多个 Agent 同时工作，彼此独立
- **Verifier**（串行门控）：所有 Worker 完成后自动触发，验证产出质量
- **Synthesizer**（最终合成）：Verifier 通过后，合并所有 Worker 产出为最终交付物

依赖关系自动由看板系统维护——Verifier 标记为 Workers 的"子节点"，Synthesizer 标记为 Verifier 的子节点。Dispatcher 检查依赖条件，仅当所有父节点完成时才 promote 子节点。

---

## 二、Swarm 流程详解

### 2.1 创建 Swarm

通过一条命令创建完整的 Swarm 图：

```bash
hermes kanban --board blog swarm \
  --worker "researcher:调研某主题" \
  --worker "writer:撰写文章初稿" \
  --verifier reviewer \
  --synthesizer publisher \
  "任务目标描述"
```

这条命令会创建 5 个任务卡片，并自动建立依赖关系：

| 任务 | Assignee | 依赖 |
|------|----------|------|
| Root | default | — |
| Worker 1 | researcher | Root |
| Worker 2 | writer | Root |
| Verifier | reviewer | Worker 1 + Worker 2 |
| Synthesizer | publisher | Verifier |

### 2.2 自动调度

创建完成后，**Gateway Dispatcher**（内置于 Gateway 进程中）会：

1. 定期轮询看板（默认每秒一次）
2. 发现 `ready` 状态的 Worker 任务
3. 为每个 Worker 启动独立的 Hermes 子进程（带独立 workspace）
4. Worker 完成时自动检查 Verifier 的依赖是否全部满足
5. 满足条件时 promote Verifier 为 `ready`
6. 同样的逻辑向下传递到 Synthesizer

### 2.3 Agent 内部工作流

每个被调度的 Agent 在执行任务时会经历：

```
1. kanban_show → 读取父任务上下文
2. 读取 sibling 输出（通过看板评论/元数据交换）  
3. 执行具体工作（调研、写作、审核等）
4. 产出文件写入 workspace
5. kanban_comment → 向根任务发布进度摘要
6. kanban_complete → 标记完成
```

Agent 之间**不直接通信**，所有信息交换通过 Kanban 看板（共享黑板模式）：

- 父任务的 body/context
- 已完成 sibling 的 completion metadata
- 根任务的评论区

### 2.4 边缘场景处理

| 场景 | 处理方式 |
|------|---------|
| Worker 崩溃 | Dispatcher 检测到超时 → 重试（默认最多 2 次） |
| 单 Worker 失败 | 不影响其他 Worker，仅 Verifier 收到失败通知 |
| 全部正常 | Verifier 自动被 promote，流程继续 |
| 审核不通过 | Verifier 可 block 任务，要求 Worker 重做 |
| Workspace 清理 | 任务完成后自动清理 scratch workspace |

---

## 三、实战：博客写作 Swarm

我们以博客写作为例，展示完整的 Swarm 流程。

### 3.1 创建任务

用户通过微信发送："写一篇关于机器学习的入门文章"→ Agent 路由：

```bash
hermes kanban --board blog create "机器学习入门文档" --assignee orchestrator
```

### 3.2 Orchestrator 拆解

Orchestrator（当前会话）领取任务后：

```bash
# 创建 Swarm 图
hermes kanban --board blog swarm \
  --worker "researcher:调研 ML 基础概念、分类与常用算法" \
  --worker "writer:撰写 MLA 入门文档初稿" \
  --verifier reviewer \
  --synthesizer publisher \
  "机器学习入门文档"
```

### 3.3 自动执行

Dispatcher 自动调度：

```
时间线：
T+0s   → Researcher + Writer 同时启动
T+2m32s → Researcher 完成（产出 14KB 调研报告）
T+2m45s → Writer 完成（产出 ~2300 字初稿，含可运行代码）
T+2m50s → Verifier 自动 promote → 审核
T+3m00s → Verifier 通过 → Synthesizer 启动
T+5m57s → Synthesizer 完成（产出 295 行终稿 + humanizer 润色）
```

### 3.4 状态查看

用户随时可以通过以下命令查看进度：

```bash
hermes kanban --board blog list          # 整体看板
hermes kanban --board blog log <task>    # 查看某个 worker 的详细日志
hermes kanban --board blog watch         # 实时事件流
```

---

## 四、最佳实践

### 4.1 角色分工设计

| 角色 | 职责 | 建议技能 |
|------|------|---------|
| Researcher | 收集资料、整理事实、标注来源 | `web` 工具集 |
| Writer | 撰写初稿、组织结构、提供代码示例 | `terminal`、`file` |
| Reviewer | 技术准确性审核、结构建议、风格检查 | 领域知识 |
| Publisher/Synthesizer | 合并产出、统一风格、humanizer 润色 | `humanizer` 技能 |

### 4.2 Skill 配置

为每个角色配置专属 skill 可以大幅提升输出质量。例如为 Synthesizer 配置 `humanizer` skill 可以自动做反 AI 语气优化。

### 4.3 避免的陷阱

- **不要在同一 Swarm 中放太多 Worker**：建议 2-3 个并行 Worker，过多会增加 Verifier 的等待时间
- **Worker 间不要有隐性依赖**：Worker 应独立工作，如果需要顺序执行，应该用链式任务而不是并行 Swarm
- **Scratch workspace 是临时的**：任务完成后自动清理，重要产出应在完成前同步到外部存储

---

## 五、总结

Kanban Swarm 的核心价值在于：

1. **可靠的任务编排**：基于 SQLite 看板的 DAG 依赖系统，比基于消息的多 Agent 系统更可靠
2. **完全的审计追踪**：每一步事件都持久化到看板，可以回溯任何一次运行
3. **进程级隔离**：每个 Agent 运行在独立进程中，一个崩溃不影响其他
4. **零配置部署**：无需消息队列、无需注册中心，一条命令即可启动 Swarm

当前 Swarm v1 的 Worker 全部并行执行，未来可以支持更复杂的 DAG 拓扑（链式、条件分支、循环）。但即使只有 v1 的能力，对于博客写作、文档生成、代码审查等典型场景，已经能显著提升产出效率——多个 Agent 分工协作，互不阻塞，自动流转。

如果你已经在使用 Hermes Agent，尝试在下一个博客任务中启用 Kanban Swarm，感受多智能体协作的力量。
