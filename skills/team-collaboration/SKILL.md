---
name: flashduty-team-collaboration
description: This skill should be used when the user asks to "assign incident", "find team member", "query teams", "who is on call", "oncall schedule", "escalation rules", "团队成员", "协作空间", "值班", "指派", "升级规则", "channels", "teams", "members", "escalation", or discusses incident assignment and team coordination.
version: 3.0.0
---

# FlashDuty 团队协作

本技能帮助查找团队、成员和值班信息，并为事件指派提供建议。

## SRE 背景

事件响应需要：
- 清晰的升级路径：当 SLO 受威胁时联系谁
- 值班轮换健康度：均衡负载，避免倦怠
- 响应时间目标：按严重级别的 MTTA 目标

## 主要 Sub-Agent

### `flashduty-team-resolver`
**用于**：所有团队/成员/指派查询

**SRE 特定用例**：
- 查找负责影响 SLO 事件的值班工程师
- 当错误预算快速消耗时进行升级
- 识别重大故障的事件指挥官

**查询类型**：
```json
{
  "find_member": { "name": "John" | "email": "..." | "phone": "..." },
  "find_team": { "name": "SRE" },
  "get_oncall": { "team_id": "..." | "team_name": "..." },
  "resolve_channel": { "channel_id": "..." | "channel_name": "..." },
  "suggest_assignee": { "incident_id": "...", "channel_id": "..." },
  "get_escalation_rules": { "channel_id": "..." }
}
```

## 并行执行模式

### 模式 1：多团队值班检查
```
用户：哪些团队有人在值班？
→ 获取所有团队
→ 为每个团队并行启动 Agent
→ 汇总值班状态
```

### 模式 2：指派建议
```
用户：这个事件应该指派给谁？
→ 并行执行：
  Agent 1：获取事件详情
  Agent 2：获取频道值班表
→ 确定最佳指派人
```

## 工作流

### 步骤 1：识别查询类型
解析用户请求以确定：
- 查找人？ → `find_member`
- 查找团队？ → `find_team`
- 查看值班？ → `get_oncall`
- 查询升级规则？ → `get_escalation_rules`
- 需要帮助指派？ → `suggest_assignee`
- 直接分配事件？ → 直接使用 `mcp__flashduty__assign_incident`

### 步骤 2：启动团队解析器
```
Task({
  subagent_type: "general-purpose",
  description: "解析团队查询",
  prompt: `你是 flashduty-team-resolver。处理此查询：${query}`
})
```

### 步骤 3：展示解析结果

## 使用模式

### 查找团队成员
```
用户：在 SRE 团队中查找 John
→ 启动 resolver：{ find_member: { name: "John" } }
→ 如果有多个匹配，请用户确认
```

### 查看值班
```
用户：平台团队谁在值班？
→ 启动 resolver：{ get_oncall: { team_name: "platform" } }
→ 展示值班表
```

### 建议指派
```
用户：谁应该处理事件 FD123456？
→ 启动 resolver：{ suggest_assignee: { incident_id: "FD123456" } }
→ 基于频道 + 值班推荐
```

### 解析协作空间配置
```
用户：展示基础架构频道详情
→ 启动 resolver：{ resolve_channel: { channel_name: "基础架构" } }
→ 展示频道配置 + 升级规则
```

### 查询升级规则
```
用户：查看这个协作空间的升级策略
→ 启动 resolver：{ get_escalation_rules: { channel_id: "..." } }
→ 展示升级规则配置（升级层级、通知方式、超时时间）
```

### 直接分配事件
```
用户：把事件 FD123 分配给 John
→ 查找 John 的 person_id
→ 直接调用 mcp__flashduty__assign_incident
   { incident_ids: ["FD123"], type: "assign", person_ids: [john_id] }
→ 确认分配成功
```

## 多查询处理

当用户一次提出多个问题时：

```
用户：查找 SRE 团队值班，同时查找 John
→ 并行启动：
  Agent 1：{ get_oncall: { team_name: "SRE" } }
  Agent 2：{ find_member: { name: "John" } }
→ 展示两个结果
```

## 响应格式

```
## 团队解析结果

### 查询：[查询描述]

**主要联系人**：
- 姓名：[name]
- 邮箱：[email]
- 人员 ID：[id]

**当前值班**（如适用）：
- [name] ([email])
- 轮换：[schedule]

**升级路径**：
1. 主要联系人：[contact]
2. X 分钟后：[escalation_target]
3. Y 分钟后：[final_target]

**备选**（如果有多个匹配）：
- [选项 1]
- [选项 2]
```
