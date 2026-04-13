---
name: auto-reply-topic-trigger
overview: 在现有 triggerKeywords 机制基础上，新增「话题分组触发」功能：支持配置多个话题，每个话题包含一组关键词和一段上下文提示，命中对应话题时将提示前置附加到消息中，再发给 AI。
todos:
  - id: add-topic-triggers-config
    content: 在 src/config.ts 中新增 TopicTriggerItem 接口和 getTopicTriggers() 配置读取函数
    status: pending
  - id: implement-topic-trigger-logic
    content: 在 src/handlers/process-inbound.ts 中实现 topicTriggers 匹配逻辑，并将 contextPrefix 注入到发送给 AI 的消息文本
    status: pending
    dependencies:
      - add-topic-triggers-config
---

## 用户需求

在 QQ 群聊中，不需要 @ 机器人，机器人也能根据预定义的「话题规则」自动接话并回复。

## 产品概述

扩展现有 OneBot 插件的群消息触发机制，新增**话题分组触发（Topic Triggers）**功能。用户预先在配置文件中定义多个话题，每个话题包含一组关键词和可选的上下文提示。当群消息命中某话题时，机器人自动触发，并将对应话题的上下文提示附加到消息前发给 AI，让 AI 在该话题背景下自由回复。

## 核心功能

- **话题分组配置**：在 `openclaw.json` 中定义 `topicTriggers` 数组，每条记录含话题名称、关键词组、匹配模式（`prefix` / `contains`）、可选上下文提示
- **多话题并行匹配**：收到群消息时，按配置顺序依次检查所有话题，命中第一个匹配话题即触发响应
- **上下文提示注入**：命中话题后，若该话题配置了 `contextPrefix`，则将其拼接到原始消息文本前面后转发给 AI（AI 收到的是 `[contextPrefix]\n[原始消息]`）
- **与现有机制兼容**：`topicTriggers` 与现有 `triggerKeywords` 并列生效；当 `requireMention: false` 且两者都配置时，两者的关键词都能触发；当两者都未配置时降级为响应所有消息
- **对所有群统一生效**，无需按群号单独配置

## 技术栈

现有项目：TypeScript + Node.js，直接在已有文件上扩展，无需引入新依赖。

## 实现思路

### 整体策略

在 `requireMention: false` 分支的触发检查逻辑中，新增 `topicTriggers` 的匹配流程：

1. 先尝试匹配 `topicTriggers`，命中时记录该话题的 `contextPrefix`
2. 若未命中 `topicTriggers`，再走原有 `triggerKeywords` 逻辑
3. 匹配成功后，将命中话题的 `contextPrefix`（若有）拼接到 `messageText` 前，再进入消息派发流程

### 关键设计决策

- **contextPrefix 仅修改发给 AI 的 `messageText`**：不修改 `formattedBody` 的 `from`/`sender` 等字段，保持用户标识不变；也不写入历史记录（`recordPendingHistoryEntry` 仍使用原始 `messageText`），避免上下文污染
- **匹配函数复用**：复用现有 `checkTriggerKeyword(text, keywords, mode)` 函数，不重复造轮子
- **优先级**：`topicTriggers` 优先于 `triggerKeywords`；命中 `topicTriggers` 后不再检查 `triggerKeywords`

## 实现细节

### 配置格式（openclaw.json）

```
{
  "channels": {
    "onebot": {
      "requireMention": false,
      "topicTriggers": [
        {
          "name": "天气",
          "keywords": ["天气", "气温", "下雨"],
          "mode": "contains",
          "contextPrefix": "用户在询问天气相关信息，请根据你的知识回答。"
        },
        {
          "name": "编程",
          "keywords": ["代码", "bug", "报错"],
          "mode": "contains",
          "contextPrefix": "用户在询问编程技术问题，请给出专业解答。"
        }
      ]
    }
  }
}
```

`contextPrefix` 为可选字段，省略时话题仅作触发条件，不注入额外提示。

### messageText 注入时机

在触发检查通过后、构建 `formattedBody` 之前，修改 `messageText`（用于发给 AI 的版本），保留原始 `messageText` 用于历史记录写入。

```
原始 messageText  →  recordPendingHistoryEntry（历史记录，保持原文）
                 →  aiMessageText = contextPrefix + "\n" + messageText（发给 AI）
```

### 触发逻辑流程

```mermaid
flowchart TD
    A[收到群消息] --> B{requireMention?}
    B -- true --> C{@ 了机器人?}
    C -- 否 --> Z[忽略]
    C -- 是 --> PASS[进入处理流程]
    B -- false --> D{配置了 topicTriggers?}
    D -- 是 --> E{命中某话题?}
    E -- 是 --> F[记录 contextPrefix, 进入处理]
    E -- 否 --> G{配置了 triggerKeywords?}
    D -- 否 --> G
    G -- 是 --> H{命中关键词?}
    H -- 否 --> Z
    H -- 是 --> PASS
    G -- 否 --> PASS
    F --> PASS
```

## 架构设计

修改范围极小，仅涉及以下两个文件，不影响其他模块：

- `src/config.ts`：新增 `getTopicTriggers()` 函数，解析并校验 `topicTriggers` 配置，返回强类型数组
- `src/handlers/process-inbound.ts`：
- 在触发检查块（142-168 行）中插入 `topicTriggers` 匹配逻辑
- 将匹配到的 `contextPrefix` 传递到后续消息构建阶段

## 目录结构

```
src/
├── config.ts                     # [MODIFY] 新增 TopicTriggerItem 类型定义和 getTopicTriggers() 函数
└── handlers/
    └── process-inbound.ts        # [MODIFY] 触发检查块新增 topicTriggers 分支；messageText 注入 contextPrefix
```

## 关键类型定义

```typescript
// src/config.ts 新增
export interface TopicTriggerItem {
  name: string;                      // 话题名称（用于日志）
  keywords: string[];                // 关键词组
  mode?: "prefix" | "contains";     // 匹配模式，默认 "contains"
  contextPrefix?: string;            // 命中时注入到 AI 消息的前缀，可选
}

export function getTopicTriggers(cfg: any): TopicTriggerItem[];
```