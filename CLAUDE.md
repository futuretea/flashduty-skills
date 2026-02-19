# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 提供处理此代码库的指导。

## 项目概述

这是一个 Claude Code 插件，提供 FlashDuty 集成技能，包括事件管理、诊断、团队协作和事件分析。插件采用 **Sub-Agent + Skill** 架构，其中 Skill 作为轻量级触发器，将复杂操作委托给具有独立上下文的专业 Sub-Agent。

## 项目结构

```
flashduty-assistant/
├── .claude/
│   └── settings.local.json              # 工具权限配置
├── .claude-plugin/
│   ├── marketplace.json                 # 插件市场元数据
│   └── plugin.json                      # 插件元数据
├── agents/                              # Sub-Agent 定义
│   ├── stats-collector.md               # 聚合统计 Agent
│   ├── diagnosis-engine.md              # 事件诊断 Agent
│   ├── team-resolver.md                 # 团队/值班解析 Agent
│   ├── incident-analyzer.md             # 复杂查询 Agent
│   ├── postmortem-generator.md          # 无责备事后分析 Agent (SRE)
│   ├── error-budget-tracker.md          # SLO/错误预算 Agent (SRE)
│   └── toil-analyzer.md                 # 琐事分析 Agent (SRE)
├── skills/                              # Skill 触发器
│   ├── incident-management/SKILL.md     # 事件生命周期操作
│   ├── incident-diagnosis/SKILL.md      # 事件诊断
│   ├── incident-analytics/SKILL.md      # 统计分析
│   ├── team-collaboration/SKILL.md      # 团队协调
│   └── sre-practices/SKILL.md           # SRE 最佳实践
├── .gitignore
├── CLAUDE.md                            # 本文件
├── LICENSE                              # MIT 许可证
└── README.md
```

## 架构：Sub-Agent + Skill

### Skill 层
- **目的**：意图识别和触发逻辑
- **职责**：决定调用哪个 Sub-Agent
- **特点**：轻量、快速、最小逻辑

### Sub-Agent 层
- **目的**：在隔离上下文中执行复杂操作
- **职责**：获取数据、执行分析、返回结构化结果
- **特点**：独立上下文、可并行、范围聚焦

### 为什么这样设计？

每个 Sub-Agent 有独立上下文，不会互相干扰；多个 Agent 可以并行跑，比如同时分析三个事件。Skill 只管「什么时候行动」，Agent 只管「怎么干」，职责清晰。

## Agent 定义 (AGENT.md)

每个 Agent 定义包括：

```yaml
---
name: flashduty-agent-name
description: 何时使用此 Agent
tools: ["mcp__flashduty__xxx"]  # 可用 MCP 工具
parallel: true                  # 可与其他 Agent 并行运行
---

# Agent 职责
# 输入/输出格式
# 执行策略
```

## Skill 定义 (SKILL.md)

Skill 委托给 Agent：

```yaml
---
name: skill-name
description: 触发条件...
version: 3.0.0
---

## 可用 Sub-Agent
- `agent-name` - 何时使用

## 并行模式
并行 Agent 执行示例

## 工作流
1. 解析用户请求
2. 确定 Agent 策略（单个/并行）
3. 启动 Sub-Agent
4. 汇总结果
```

## 并行执行模式

### 多频道对比
```javascript
// 为每个频道并行启动 Agent
const channels = ["基础架构", "数据库", "应用层"];
const tasks = channels.map(ch => Task({
  subagent_type: "general-purpose",
  description: `${ch} 的统计`,
  prompt: `你是 flashduty-stats-collector。使用 time_range: "7d" 分析频道：${ch}`
}));
const results = await Promise.all(tasks);
```

### 多事件诊断
```javascript
// 并行诊断多个事件
const incidents = ["FD123", "FD124", "FD125"];
const tasks = incidents.map(id => Task({
  subagent_type: "general-purpose",
  description: `诊断 ${id}`,
  prompt: `你是 flashduty-diagnosis-engine。诊断事件 ${id}`
}));
```

### 多维度分析
```javascript
// 同时分析不同维度
const dimensions = ["severity", "channel", "trend"];
const tasks = dimensions.map(dim => Task({
  subagent_type: "general-purpose",
  description: `按 ${dim} 分析`,
  prompt: `你是 flashduty-stats-collector。使用 time_range: "7d" 按维度分析：${dim}`
}));
```

## 可用 MCP 工具

**事件管理：**
- `mcp__flashduty__create_incident`
- `mcp__flashduty__ack_incidents`
- `mcp__flashduty__resolve_incidents`
- `mcp__flashduty__reopen_incidents`
- `mcp__flashduty__snooze_incidents`
- `mcp__flashduty__comment_incidents`
- `mcp__flashduty__update_incident` -- 更新事件标题、描述、严重级别、影响、根因、解决方案
- `mcp__flashduty__assign_incident` -- 分配事件给指定人员或升级规则
- `mcp__flashduty__get_incident` -- 返回结果自动包含人员/协作空间名称
- `mcp__flashduty__list_incidents` -- 支持 `time_range`、`brief` 参数，返回结果自动包含人员/协作空间名称
- `mcp__flashduty__list_similar_incidents` -- 查找相似历史事件，用于模式分析和诊断
- `mcp__flashduty__get_incident_timeline` -- 支持 `types` 过滤参数，返回结果自动包含操作人名称
- `mcp__flashduty__list_incident_alerts` -- 支持 `is_active` 过滤参数

**告警：**
- `mcp__flashduty__list_alerts` -- 支持 `time_range`、`brief`、`is_active` 参数
- `mcp__flashduty__get_alert`
- `mcp__flashduty__close_alerts`

**团队/协作：**
- `mcp__flashduty__list_members`
- `mcp__flashduty__list_teams`
- `mcp__flashduty__list_channels`
- `mcp__flashduty__get_channel`
- `mcp__flashduty__list_schedules`
- `mcp__flashduty__query_escalation_rules` -- 查询协作空间的升级规则配置

**变更管理：**
- `mcp__flashduty__query_changes` -- 查询部署/变更事件，支持 `time_range`，用于关联事件与近期部署

**自定义字段：**
- `mcp__flashduty__query_fields` -- 查询自定义字段定义

**分析：**
- `mcp__flashduty__get_incident_stats` -- 支持 `time_range`、`aggregate_unit`、`team_ids`、`severities`、`labels` 参数

### `time_range` 参数

`list_incidents`、`list_alerts`、`get_incident_stats`、`query_changes` 支持 `time_range` 参数，作为 `start_time`+`end_time` 的替代：
- **时长格式**：`'1h'`、`'24h'`、`'7d'`、`'30d'`、`'1w'`、`'6M'`（结束时间 = 当前时间）
- **命名范围**：`'last_day'`（昨天 00:00:00 到 23:59:59）、`'last_week'`（上周一到周日）
- 如果不设置 `time_range`，则 `start_time` 和 `end_time` 为必填

### `brief` 参数

`list_incidents` 和 `list_alerts` 支持 `brief: true` 参数：
- 仅返回关键字段，大幅减少响应大小
- **推荐在初始查询时使用**，然后对重点项获取完整详情
- 事件 brief 字段：`incident_id`, `title`, `incident_severity`, `progress`, `created_at`, `channel_name`, `channel_id`, `num`
- 告警 brief 字段：`alert_id`, `title`, `alert_severity`, `is_active`, `created_at`, `channel_name`, `channel_id`

### `aggregate_unit` 参数

`get_incident_stats` 支持 `aggregate_unit` 参数进行时间序列分解：
- `day`：按天分解
- `week`：按周分解
- `month`：按月分解
- 省略：整个周期的汇总统计

### 数据富化（Enrichment）

以下工具的返回结果会自动将 ID 解析为人类可读的名称：
- `list_incidents`、`get_incident`：`creator_id` → `creator_name`、`closer_id` → `closer_name`、`channel_id` → `channel_name`、`responders[].person_id` → `person_name`、`assigned_to.person_ids` → `person_names`
- `get_incident_timeline`：`operator_id` → `operator_name`、`person_id` → `person_name`
- `list_similar_incidents`：同 `list_incidents`
- 使用 `brief: true` 时不触发富化（减少 API 调用）

## 何时使用 Sub-Agent

### 始终使用 Sub-Agent 的场景
- 需要多步分析或多源数据获取（时间线 + 告警 + 关联事件）
- 可并行化的调用
- SRE 实践：事后分析、错误预算、琐事分析

### 直接调用 MCP 工具的场景
- 已知 ID 的简单 CRUD（ack / resolve / comment）
- 参数明确的单工具调用

## SRE Agent 使用

### 事后分析生成
```javascript
// 事件解决后
Task({
  subagent_type: "general-purpose",
  description: "为事件 " + incident_id + " 生成事后分析",
  prompt: `你是 flashduty-postmortem-generator。为事件 ${incident_id} 创建无责备事后分析。包括：时间线、5 Whys 分析、行动项和经验教训。`
})
```

### 错误预算追踪
```javascript
// 多服务预算追踪
const services = [
  { name: "api-gateway", slo: 99.99 },
  { name: "payment", slo: 99.95 },
  { name: "user-service", slo: 99.9 }
];

const tasks = services.map(svc => Task({
  subagent_type: "general-purpose",
  description: `追踪 ${svc.name} 的错误预算`,
  prompt: `你是 flashduty-error-budget-tracker。分析 ${svc.name}，SLO ${svc.slo}%，过去 30 天。返回预算消耗、燃烧率和建议。`
}));
```

### 琐事分析
```javascript
// 月度琐事审查
Task({
  subagent_type: "general-purpose",
  description: "分析运营琐事",
  prompt: `你是 flashduty-toil-analyzer。分析过去 30 天的事件。识别：告警噪音、重复模式、人工干预和自动化机会。返回琐事评分和路线图。`
})
```

## 添加新组件

### 添加 Sub-Agent

1. 创建 `agents/<agent-name>.md`
2. 定义 Agent 名称、描述、工具、并行能力
3. 记录输入/输出格式
4. 从相关 Skill 引用

### 添加 Skill

1. 创建 `skills/<skill-name>/SKILL.md`
2. 在前言中定义触发条件
3. 记录调用哪个 Sub-Agent
4. 提供并行执行模式

## 测试

1. 安装插件：`/plugin install flashduty-assistant@flashduty-assistant`
2. 使用用户查询测试 Skill 触发
3. 验证 Sub-Agent 接收正确参数
4. 检查并行执行是否按预期工作

## 重要说明

- Sub-Agent 使用 `subagent_type: "general-purpose"` 运行
- 长时间运行的并行任务使用 `run_in_background: true`
- 向 Sub-Agent 提供清晰的结构化提示，在 Skill 层处理 Agent 失败
- 时间参数支持两种方式：`time_range`（如 `'7d'`、`'last_week'`）或 `start_time`+`end_time`（Unix 秒级时间戳）
- 优先使用 `time_range` 简化查询：`'1h'`、`'24h'`、`'7d'`、`'30d'`、`'1w'`、`'6M'`、`'last_day'`、`'last_week'`
- **始终使用 `brief: true`** 查询事件/告警列表，减少响应大小，然后对关键项获取完整详情
- 趋势分析使用 `aggregate_unit`（`day`、`week`、`month`）分解时间序列数据
- Agent 名称统一使用 `flashduty-` 前缀
- 事件 ID 格式："FD123456" 或类似
- 严重级别：`Critical`、`Warning`、`Info`
- 进度：`Triggered`、`Processing`、`Closed`
