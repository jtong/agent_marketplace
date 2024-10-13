# agents.json 概述

## agents.json 基本数据结构

`agents.json` 文件位于 `.ai_helper/agent/chat/` 目录下，用于配置各种聊天 agent。它的基本结构是一个 JSON 对象，包含一个名为 "agents" 的数组，数组中的每个元素代表一个 agent 的配置。

基本结构如下：

```json
{
  "agents": [
    {
      "name": "AgentName",
      "path": "./path/to/AgentFile.js",
      "metadata": {},
      "settings": {},
      "operations": []
    },
    // 更多 agent 配置...
  ]
}
```

每个 agent 的配置对象包含以下字段：

1. `name`（必需）：agent 的唯一标识符，用于在界面上显示和选择 agent。

2. `path`（必需）：agent 的 JavaScript 文件相对路径，相对于 `.ai_helper/agent/chat/` 目录。

3. `metadata`（可选）：一个对象，包含 agent 的元数据，可以用于存储额外的信息。

4. `settings`（可选）：一个对象，包含 agent 的初始设置。这些设置会被复制到新创建的聊天 chat thread 中。

5. `operations`（可选）：一个数组，定义了 agent 支持的操作或用户可以进行的交互。

下面是一个更详细的示例，展示了这些字段的使用：

```json
{
  "agents": [
    {
      "name": "MultiModelAgent",
      "path": "./multi_llm_agent/MultiModelAgent.js",
      "metadata": {
        "description": "An agent that can switch between different AI models",
        "version": "1.0.0"
      },
      "settings": {
        "currentModel": {
          "apiKey": "openai",
          "model": "gpt-3.5-turbo"
        }
      },
      "operations": [
        {
          "type": "setting",
          "control": "select",
          "name": "AI Platform",
          "settingKey": "currentModel",
          "options": [
            {
              "label": "openai__gpt-3.5",
              "value": {
                "apiKey": "openai",
                "model": "gpt-3.5-turbo"
              }
            },
            {
              "label": "gemini__1.5-flash",
              "value": {
                "apiKey": "gemini",
                "model": "gemini-1.5-flash"
              }
            }
          ],
          "default": {
            "apiKey": "openai",
            "model": "gpt-3.5-turbo"
          }
        },
        {
          "name": "Reset Context",
          "control": "button",
          "type": "task",
          "task": {
            "name": "ResetContext",
            "type": "action",
            "skipUserMessage": true,
            "skipBotMessage": true
          }
        }
      ]
    }
  ]
}
```

在这个示例中，我们定义了一个名为 "MultiModelAgent" 的 agent，它包含了所有可选字段。`metadata` 提供了 agent 的描述和版本信息，`settings` 定义了初始的 AI 模型设置，`operations` 包含了一个设置操作（用于选择 AI 平台）和一个任务操作（用于重置上下文）。

这种结构允许您灵活地定义各种 agent，每个 agent 可以有自己的设置和操作，从而提供丰富的用户交互体验。


## settings 的处理

`agents.json` 中的 `settings` 字段用于定义 agent 的初始设置。这些设置会在以下情况下被使用和复制：

1. 创建新的聊天 chat thread 时：
   - 当用户创建一个新的聊天时，`agentLoader.js` 中的 `loadAgentForThread` 方法会合并全局设置和 agent 特定的设置。

2. 更新 chat thread 设置时：
   - 用户可以通过界面更改设置，这些更改会被保存到特定聊天 chat thread 的设置中。

3. 设置的存储位置：
   - 每个聊天 chat thread 的设置都存储在该 chat thread 的 JSON 文件中，位于 `.ai_helper/agent/memory_repo/chat_threads/<thread_id>/thread.json`。

4. 设置的动态更新：
   - 当用户通过界面更改设置时，`chatViewProvider.js` 中的 `handleUpdateSetting` 方法会更新 chat thread 的设置并通知 `agentLoader` 重新加载 agent。


## operations 配置

`operations` 字段用于定义 agent 可以执行的操作或用户可以进行的交互。它是一个数组，每个元素代表一个操作。主要有两种类型的操作：

1. 设置（setting）操作
2. 任务（task）操作

### 设置（setting）类型操作示例：

在 `agents.json` 中定义的 setting operation 实际上是用来修改特定 chat thread 的设置，而不是 agent 的全局设置。这是一个重要的区别，因为它允许每个聊天会话（chat thread）有自己的独立配置。

让我们通过一个例子来详细说明：

```json
{
  "type": "setting",
  "control": "select",
  "name": "AI Platform",
  "settingKey": "currentModel",
  "options": [
    {
      "label": "openai__gpt-3.5",
      "value": {
        "apiKey": "openai",
        "model": "gpt-3.5-turbo"
      }
    },
    {
      "label": "gemini__1.5-flash",
      "value": {
        "apiKey": "gemini",
        "model": "gemini-1.5-flash"
      }
    }
  ],
  "default": {
    "apiKey": "openai",
    "model": "gpt-3.5-turbo"
  }
}
```

当用户通过界面使用这个 setting operation（例如，从下拉菜单中选择不同的 AI 模型）时，发生以下过程：

1. 用户的选择被捕获。

2. `chatViewProvider.js` 中的 `handleUpdateSetting` 方法被调用。

3. 这个方法会更新当前 chat thread 的设置，而不是 agent 的全局设置。具体来说，它会：
   - 获取当前 chat thread 的设置（如果存在的话）。
   - 更新设置中的 `currentModel` 字段，使用用户选择的新值。
   - 将更新后的设置保存回该特定 chat thread 的配置中。

4. 更新后的设置被保存到该 chat thread 的 JSON 文件中（位于 `.ai_helper/agent/memory_repo/chat_threads/<thread_id>/thread.json`）。

5. `agentLoader` 被通知重新加载该 chat thread 的 agent，使用更新后的设置。

这种机制的好处是：

- 每个 chat thread 可以有自己的独立设置，不会影响其他 chat thread 或 agent 的全局设置。
- 用户可以在同一个 agent 的不同聊天会话中使用不同的配置。
- 设置的更改是持久化的，即使关闭并重新打开 VSCode，每个 chat thread 的设置也会被保留。

重要的是要理解，虽然这些 setting operations 是在 `agents.json` 中定义的，但它们实际上是作用于单个 chat thread 的，而不是修改 agent 的全局配置。这为用户提供了极大的灵活性，允许他们为每次对话定制 agent 的行为，而不会影响其他正在进行的或未来的对话。


### 任务（task）类型操作示例：

```json
{
  "name": "重置上下文",
  "control": "button",
  "type": "task",
  "task": {
    "name": "ResetContext",
    "type": "action",
    "skipUserMessage": true,
    "skipBotMessage": true
  }
}
```

这个操作会在聊天界面上创建一个按钮，点击后执行task标记的该任务。该任务的效果由Agent保证，插件只负责调用将该任务发送给Agent。
