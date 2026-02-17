---
name: flashduty-incident-management
description: This skill should be used when the user asks to "create incident", "acknowledge incident", "close incident", "resolve incident", "reopen incident", "snooze incident", "comment on incident", "query incidents", "list incidents", "创建事件", "确认事件", "关闭事件", "重新打开事件", "延迟事件", "评论事件", "查询事件", or discusses incident lifecycle operations (事件生命周期管理).
version: 3.0.0
---

# FlashDuty 事件管理

本技能管理事件的生命周期。简单操作直接执行；复杂查询委托给 Sub-Agent。

## SRE 集成

本技能遵循 SRE 最佳实践：

### 事件严重级别与 SLO 影响
创建或更新事件时，考虑 SLO 影响：
- **Critical (严重)**: 可能违反 SLO，需要立即响应
- **Warning (警告)**: 如果不处理可能会降低 SLO
- **Info (信息)**: 监控但通常不影响 SLO

### 事后工作流程
解决事件后：
```
用户: "事件已解决，生成事后分析"
→ 步骤 1: 直接解决事件
→ 步骤 2: 委托给 sre-practices 技能
   启动: postmortem-generator
→ 步骤 3: 安排行动项
```

### 错误预算影响
关闭带有根因的事件时：
- 如果影响 SLO，则跟踪到错误预算
- 使用 error-budget-tracker 进行多事件分析

## 直接操作（无需 Sub-Agent）

这些简单操作立即执行：

| 操作 | 工具 | 何时直接使用 |
|------|------|-------------|
| 创建事件 | `mcp__flashduty__create_incident` | 用户提供所有必需信息 |
| 确认事件 | `mcp__flashduty__ack_incidents` | 提供单个或列表 ID |
| 解决事件 | `mcp__flashduty__resolve_incidents` | 提供 ID，可选根因 |
| 重新打开 | `mcp__flashduty__reopen_incidents` | 提供 ID |
| 延迟事件 | `mcp__flashduty__snooze_incidents` | 提供 ID 和时长 |
| 添加评论 | `mcp__flashduty__comment_incidents` | 提供 ID 和评论内容 |
| 更新事件 | `mcp__flashduty__update_incident` | 修改标题/描述/严重级别/影响/根因/解决方案 |
| 分配事件 | `mcp__flashduty__assign_incident` | 指派给特定人员或升级规则 |

## Sub-Agent 委托

### `flashduty-incident-analyzer`
**用于**: 复杂事件查询、过滤、搜索

**何时委托**:
- 用户要求 "find incidents" 但没有具体 ID
- 需要复杂过滤（多个条件）
- 需要聚合或分组结果

**参数**:
```json
{
  "filters": { "severity": "...", "progress": "...", "title": "..." },
  "time_range": "7d",
  "group_by": "severity" | "channel" | "labels.xxx",
  "brief": true
}
```

> **注意**：优先使用 `time_range`（如 `'7d'`、`'24h'`、`'last_week'`）而非手动计算 `start_time`/`end_time`。使用 `brief: true` 减少数据量。

## 决策树

```
用户请求：
├─ 直接操作 (ack/resolve/snooze/comment/update/assign) + 提供 ID
│  └─ 直接使用 MCP 工具执行
│
├─ 创建事件
│  └─ 收集必需字段 → 直接创建
│
├─ 更新事件字段（标题/描述/严重级别/影响/根因/解决方案）
│  └─ 直接使用 update_incident 执行
│
├─ 分配/指派事件
│  └─ 直接使用 assign_incident（type: "assign" + person_ids 或 type: "escalateRule" + escalate_rule_id）
│
├─ 查询/搜索/列出事件
│  └─ 委托给 flashduty-incident-analyzer
│
└─ 混合 (例如 "ack all critical incidents")
   └─ 步骤 1: 委托 analyzer 查找匹配项
   └─ 步骤 2: 与用户确认
   └─ 步骤 3: 直接执行 ack
```

## 工作流模式

### 模式 1: 直接操作
```
用户: "确认事件 FD123456"
→ 直接调用 mcp__flashduty__ack_incidents
→ 确认完成
```

### 模式 2: 先查询后操作
```
用户: "确认今天所有 triggered 状态的事件"
→ 步骤 1: 启动 analyzer 查找匹配事件
     filters: { progress: "Triggered" }, time_range: "24h"
→ 步骤 2: 向用户展示列表，确认
→ 步骤 3: 使用列表调用 ack_incidents
```

### 模式 3: 创建事件
```
用户: "为数据库故障创建一个严重级别的事件"
→ 收集必需信息（标题、严重级别、描述）
→ 直接调用 create_incident
→ 返回事件 ID
```

### 模式 4: 复杂搜索
```
用户: "查找本周基础架构 channel 的所有 Critical 事件"
→ 启动 incident-analyzer
   filters: { severity: "Critical", channel_name: "基础架构" }
   time_range: "7d"
   brief: true
→ 展示结果
```

## 并行执行

### 批量操作
当用户提到多个不相关的事件时：

```
用户: "展示事件 A、B 和 C"
→ 并行：
  Agent 1: 获取事件 A 详情
  Agent 2: 获取事件 B 详情
  Agent 3: 获取事件 C 详情
→ 展示合并结果
```

### 多 Channel 查询
```
用户: "列出基础架构和数据库两个 channel 的 triggered 事件"
→ 并行：
  Agent 1: 查询基础架构 channel（brief: true）
  Agent 2: 查询数据库 channel（brief: true）
→ 合并并展示
```

## 响应模式

### 直接操作成功
```
[操作] 成功完成。
- 事件: [ID]
- 状态: [新状态]
- [附加详情]
```

### 批量操作成功
```
[操作] 已完成 [N] 个事件：
- [ID 1]: 成功
- [ID 2]: 成功
...
```

### 查询结果
```
找到 [N] 个匹配条件的事件：

| ID | 标题 | 严重级别 | 状态 | Channel |
|----|------|---------|------|---------|
|... |...  |...      |...   |...      |

[操作按钮或建议]
```

## 错误处理

- **缺少必需字段**: 请用户提供
- **无效的事件 ID**: 搜索类似事件
- **权限被拒绝**: 告知用户并建议替代方案
- **结果集过大**: 分页或建议过滤条件
- **时间范围超出保留期限**: FlashDuty 有数据保留期限，只能查询最近 N 天的数据。遇到此错误时，自动缩短查询范围为最近可查询的时间段
- **时间顺序错误**: 确保 `start_time` < `end_time`。使用 Unix 秒级时间戳

### 时间范围处理最佳实践

**推荐使用 `time_range` 参数**：
```
# 简单直观，无需计算时间戳
time_range: "24h"    # 最近 24 小时
time_range: "7d"     # 最近 7 天
time_range: "30d"    # 最近 30 天
time_range: "last_day"   # 昨天整天
time_range: "last_week"  # 上周整周
```

**当需要精确时间范围时，使用 `start_time`+`end_time`**：
```javascript
// 如果查询失败（超出保留期限），逐步缩短时间范围
// 1. 尝试最近7天
// 2. 如果失败，尝试最近3天
// 3. 如果失败，尝试最近1天
```
