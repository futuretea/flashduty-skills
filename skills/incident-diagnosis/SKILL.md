---
name: flashduty-incident-diagnosis
description: This skill should be used when the user asks to "diagnose incident", "investigate incident", "analyze incident", "find root cause", "check alerts", "incident timeline", "what happened", "诊断事件", "调查事件", "分析事件", "查找根因", "检查告警", "事件时间线", "发生了什么", or discusses incident analysis and problem diagnosis (事件分析和问题诊断).
version: 3.0.0
---

# FlashDuty 事件诊断

本技能通过委托给具有独立上下文的专用诊断 Agent 来帮助诊断事件。

## 与 SRE 实践集成

诊断事件时，注意：
- **四大黄金信号**: 延迟、流量、错误、饱和度
- **无责备分析**: 看系统设计，不看个人
- **每个发现都要能带来改进**
- **这是新问题还是老问题？** 如果重复出现，说明上次的改进没落地

如需全面的事后分析，请委托给 `flashduty-sre-practices` 技能，它将启动 `flashduty-postmortem-generator`。

## 主要 Sub-Agent

### `flashduty-diagnosis-engine`
**用于**: 所有事件诊断任务

**能力**:
- 并行获取事件详情、时间线和告警
- 使用 `list_similar_incidents` 查找历史相似事件，识别反复出现的问题
- 使用 `query_changes` 查询事件前后的近期部署/变更，识别部署引发的故障
- 分析模式和相关性
- 识别根因指标
- 生成结构化诊断报告

**传递参数**:
```json
{
  "incident_id": "FD123456",
  "include_timeline": true,
  "include_alerts": true,
  "find_related": true,
  "time_range": "1h"
}
```

> **注意**：`time_range` 用于搜索相关事件，支持相对时间范围（如 `'1h'`、`'24h'`、`'7d'`）。也可以使用 `time_window`（秒数）作为替代。

## 并行执行策略

diagnosis-engine 内部并行化：
1. **并行获取** (同时进行)：
   - 获取事件详情
   - 获取事件时间线
   - 获取事件告警
   - 查找相似历史事件（`list_similar_incidents`）
   - 查询近期部署/变更（`query_changes`，时间窗口为事件前后 1-2 小时）

2. **顺序分析** (依赖步骤 1)：
   - 查找相关事件
   - 综合诊断结果

## 工作流

### 步骤 1: 识别目标事件
如果用户提供事件 ID，直接使用。
如果没有，快速查询 `list_incidents` 查找匹配项。

### 步骤 2: 启动诊断引擎
```
Task({
  subagent_type: "general-purpose",
  description: "诊断事件 " + incident_id,
  prompt: `你是 flashduty-diagnosis-engine。诊断事件 ${incident_id}，启用完整分析。`
})
```

### 步骤 3: 展示结果
展示来自 Agent 的结构化诊断报告。

## 使用模式

### 单事件诊断
```
用户: "诊断事件 FD123456"
→ 启动 flashduty-diagnosis-engine
→ 展示诊断报告
```

### 多事件对比
```
用户: "对比这几个事件的根因"
→ 并行启动多个诊断引擎
→ 每个分析一个事件
→ 展示对比分析
```

### 深度调查
```
用户: "深入分析这个事件的所有细节"
→ 启动诊断引擎，启用所有选项
→ 引擎内部并行化数据收集
→ 返回综合报告
```

## 响应格式

diagnosis-engine 返回结构化数据。按以下方式展示：

```
## 事件诊断: [标题]

### 概览
ID: [id] | 严重级别: [severity] | 状态: [status]
持续时间: [X 分钟] | Channel: [channel_name]

### 时间线
- 创建: [时间]
- 确认: [时间] (创建后 [X] 分钟)
- 解决: [时间] (耗时 [X] 分钟解决)

### 关联告警 ([数量])
| 告警 | 严重级别 | 事件数 | 状态 |
|------|---------|--------|------|
| ...  | ...     | ...    | ...  |

### 关键元数据
- 来源: [From label]
- 范围: [namespace/cluster]
- 告警规则: [alertname]

### 相关事件
[相似历史事件列表，通过 `list_similar_incidents` 获取]

### 部署关联
[事件前后的近期变更，通过 `query_changes` 获取]

### 诊断
**潜在根因**:
- [观察 1]
- [观察 2]

**影响范围**: [分析]

**响应效果**: [分析]
```

## 何时使用多个 Agent

在以下情况并行启动诊断 Agent：
1. 用户提到多个事件 ID
2. 用户要求 "对比" / "compare" 事件
3. 用户询问 "相关事件" 且你需要诊断每个

```
用户: "分析事件 A、B、C"
→ 并行：
  Agent 1: 诊断事件 A
  Agent 2: 诊断事件 B
  Agent 3: 诊断事件 C
→ 汇总发现
```
