---
name: auto-reply-topic-trigger
overview: 扩展 OneBot 插件群消息触发机制，新增「话题分组触发（topicTriggers）」和「全量转发开关（passAllGroupMessages）」，支持混合模式：部分话题关键词触发+附加上下文提示、其余消息全量转给 AI 判断。
todos:
  - id: add-topic-triggers-config
    content: 在 src/config.ts 末尾追加 TopicTriggerItem 接口、getTopicTriggers() 和 getPassAllGroupMessages() 函数
    status: completed
  - id: implement-topic-trigger-logic
    content: 更新 src/handlers/process-inbound.ts：import 新函数、重写触发检查块为三级优先级逻辑、formattedBody 改用 aiMessageText
    status: completed
    dependencies:
      - add-topic-triggers-config
---

## 用户需求

在 QQ 群聊中，不 @ 机器人，机器人也能自动接话并回复。

## 产品概述

扩展现有 OneBot 插件的群消息触发机制，新增以下两个功能，支持混合使用，与现有 `requireMention` / `triggerKeywords` 机制完全兼容：

1. **话题分组触发（topicTriggers）**：预定义多个话题，每个话题含一组关键词 + 可选的上下文提示。命中话题时，将 `contextPrefix` 注入消息前发给 AI。AI 的人设/行为规则（如「顺顺接龙」玩法）写在 OpenClaw System Prompt 里，代码只负责传消息。
2. **全量旁听模式（passAllGroupMessages）**：配置为 `true` 时，所有群消息都转发给 AI，由 AI 自主决定是否回复（AI 回复 `NO_REPLY` 即忽略，现有 deliver 逻辑已支持）。

## 核心功能

- **话题分组配置**：`topicTriggers` 数组，每条含 `name`（话题名）、`keywords`（关键词组）、`mode`（`prefix`/`contains`，默认 `contains`）、可选 `contextPrefix`
- **多话题顺序匹配**：按配置顺序依次检查，命中第一个话题即触发，并注入该话题的 `contextPrefix`
- **全量旁听兜底**：`passAllGroupMessages: true` 时，未命中任何话题/关键词的消息也转发给 AI
- **触发优先级**（`requireMention: false` 时）：`topicTriggers` > `triggerKeywords` > `passAllGroupMessages` > 忽略
- **contextPrefix 隔离**：仅注入发给 AI 的消息文本，历史记录始终保存原始消息，避免上下文污染
- **对所有群统一生效**，无需按群号单独配置

## 技术栈

TypeScript + Node.js，在现有文件上直接扩展，无需引入新依赖。

## 实现思路

在 `requireMention: false` 分支的触发检查逻辑（`process-inbound.ts` 第 142-168 行）中，将现有单级 `triggerKeywords` 检查扩展为三级优先级流程。匹配成功后在 `formattedBody` 构建处（第 239-248 行）用 `aiMessageText` 替换 `messageText`，历史记录写入处（第 269-281 行）仍使用原始 `messageText`。

## 触发逻辑流程

```mermaid
flowchart TD
    A[收到群消息] --> B{requireMention?}
    B -- true --> C{@ 了机器人?}
    C -- 否 --> Z[忽略]
    C -- 是 --> PASS[进入处理流程]
    B -- false --> T{配置了 topicTriggers?}
    T -- 是 --> TM{命中某话题?}
    TM -- 是 --> TP[记录 contextPrefix, 触发]
    TM -- 否 --> K{配置了 triggerKeywords?}
    T -- 否 --> K
    K -- 是 --> KM{命中关键词?}
    KM -- 否 --> P{passAllGroupMessages?}
    KM -- 是 --> PASS
    K -- 否 --> P
    P -- true --> PASS
    P -- false --> Z
    TP --> PASS
```

## 实现细节

### messageText 双轨处理

```
原始 messageText ──→ recordPendingHistoryEntry（历史记录，始终保持原文）
                 ──→ aiMessageText = contextPrefix + "\n" + messageText（发给 AI）
```

`passAllGroupMessages` 触发时 `aiMessageText === messageText`，不附加任何额外内容。

### 配置格式（openclaw.json）

```
{
  "channels": {
    "onebot": {
      "requireMention": false,
      "passAllGroupMessages": true,
      "topicTriggers": [
        {
          "name": "顺顺接龙",
          "keywords": ["接龙", "顺"],
          "mode": "contains",
          "contextPrefix": "群友正在玩顺顺接龙游戏，请按你的人设规则参与。"
        },
        {
          "name": "游戏话题",
          "keywords": ["打游戏", "上分", "开黑"],
          "mode": "contains",
          "contextPrefix": "群友在聊游戏，请用你的风格加入话题。"
        }
      ]
    }
  }
}
```

`contextPrefix` 可选；`passAllGroupMessages` 独立于 `topicTriggers`，两者可单独或组合使用。

## 目录结构

```
src/
├── config.ts                  # [MODIFY] 末尾追加 TopicTriggerItem 接口、getTopicTriggers()、getPassAllGroupMessages()
└── handlers/
    └── process-inbound.ts     # [MODIFY] import 新增两个函数；触发检查块替换为三级优先级逻辑；formattedBody 使用 aiMessageText
```

## 关键类型定义

```typescript
// src/config.ts 新增
export interface TopicTriggerItem {
    name: string;                   // 话题名称（仅用于日志）
    keywords: string[];             // 关键词列表
    mode?: "prefix" | "contains";  // 匹配模式，默认 "contains"
    contextPrefix?: string;         // 命中时注入到 AI 消息的前缀，可选
}

export function getTopicTriggers(cfg: any): TopicTriggerItem[];
export function getPassAllGroupMessages(cfg: any): boolean;
```