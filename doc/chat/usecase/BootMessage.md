# 引导消息（Boot Message）

引导消息是一种特殊的虚拟消息，通常在创建新的对话线程时自动添加。它的目的是为用户提供初始指导或欢迎信息。引导消息通过 Agent 的配置文件中的 metadata 来定义，并使用 Response 对象的结构进行配置。

## Boot Message 配置

在 `agents.json` 配置文件中，每个 Agent 可以定义自己的 bootMessage。这个配置位于 Agent 的 metadata 字段中，并使用与 Response 对象相同的结构。例如：

```json
{
  "agents": [
    {
      "name": "ExampleAgent",
      "path": "./path/to/ExampleAgent.js",
      "metadata": {
        "bootMessage": {
          "message": "欢迎使用ExampleAgent！我可以帮助您完成各种任务。",
          "isHtml": false,
          "meta": {
            "isBootMessage": true,
            "isVirtual": true
          },
          "availableTasks": [
            {
              "name": "开始新任务",
              "task": {
                "name": "StartNewTask",
                "type": "ACTION",
                "message": "请选择您想开始的任务类型"
              }
            },
            {
              "name": "查看帮助",
              "task": {
                "name": "ShowHelp",
                "type": "ACTION",
                "message": "显示帮助信息"
              }
            }
          ]
        }
      }
    }
  ]
}
```

## Boot Message 数据结构

bootMessage 对象包含以下字段：

- `message` (字符串，必填): 显示给用户的欢迎或引导文本。
- `isHtml` (布尔值，可选): 指示消息是否包含 HTML 内容。默认为 false。
- `meta` (对象，可选): 包含元数据信息。
  - `isBootMessage` (布尔值): 标识这是一个引导消息。
  - `isVirtual` (布尔值): 标识这是一个虚拟消息。
- `availableTasks` (数组，可选): 一组可供用户选择的初始任务。
- `nextTasks` (数组，可选)：指定在当前响应后应自动执行的一系列任务。每个任务都是一个 Task 配置对象。

每个 availableTask 包含：
- `name` (字符串): 显示给用户的任务名称。
- `task` (对象): 描述任务的详细信息。
  - `name` (字符串): 任务的唯一标识符。
  - `type` (字符串): 任务类型，通常是 "ACTION"。
  - `message` (字符串): 执行任务时显示的消息或提示。

## 实现机制

1. 在打开聊天panel时，系统会检查选定 Agent 的配置中是否存在 bootMessage。

2. 如果存在 bootMessage且在marker之后的消息列表长度为0，系统会使用 `Response.fromJSON()` 方法创建一个 Response 对象：

   ```javascript
   const bootResponse = Response.fromJSON(agentConfig.metadata.bootMessage);
   ```

3. 创建的 Response 对象会被传递给 `chatProvider.handleResponse()` 方法进行处理：

   ```javascript
   await chatProvider.handleResponse(newThread, bootResponse, chatProvider);
   ```

4. `handleResponse()` 方法会将 bootMessage 添加到线程的消息列表中，并发送到 webview 进行显示。

5. 由于 bootMessage 被标记为虚拟消息（isVirtual: true），它会在用户界面上显示，但在构造发送给 AI 的消息历史时会被过滤掉。

## 过滤虚拟消息

在处理消息时，Agent 应该检查 isVirtual 标志。如果为 true，这些消息应该在构造发送给 AI 的消息历史时被过滤掉。例如：

```javascript
convertToAIMessages(messages) {
    return messages.filter(msg => !msg.meta.isVirtual).map(message => ({
        role: message.sender === 'user' ? 'user' : 'assistant',
        content: message.text
    }));
}
```

通过这种方式，bootMessage 可以为用户提供有用的初始信息和交互选项，而不会影响 AI 的上下文理解。这种机制使得我们可以为不同的 Agent 定制独特的欢迎体验，同时保持了消息处理的一致性和灵活性。

## 常见用例：重置上下文并显示引导消息

## 用例描述

当用户触发"重置上下文"操作时，系统应清除当前对话历史，并显示一条预定义的引导消息（bootMessage）。这个引导消息应该为用户提供清晰的指示，表明上下文已被重置，并准备开始新的对话。

## 实现要点

1. 配置文件修改
   - 在 `agents.json` 中，确保 ResetContext 任务的 `skipBotMessage` 设置为 `false`。
   - 在 agent 的 metadata 中定义 bootMessage。

2. 代码实现
   - 在 EntryAgent.js 中，ResetContext 任务应返回 bootMessage 对应的 Response。
   - 在 ChatMessageHandler.js 中，处理 ResetContext 任务时清除线程消息。
   - 在 ChatViewProvider.js 中，确保正确显示 bootMessage。

3. 用户体验
   - 用户触发重置后，应立即看到新的引导消息。
   - 引导消息应包含适当的任务选项，便于用户开始新的对话。

## 示例配置 (agents.json)

```json
{
    "name": "重置上下文",
    "control": "button",
    "type": "task",
    "task": {
        "name": "ResetContext",
        "type": "action",
        "skipUserMessage": true,
        "skipBotMessage": false
    }
}
```

## EntryAgent.js 中的实现

在 EntryAgent.js 文件中，ResetContext 任务的处理应该如下：

```javascript
async executeTask(task, thread) {
    if (task.name === "ResetContext") {
        try {
            // 执行重置上下文的操作...

            // 返回 bootMessage 对应的 Response
            if (this.metadata && this.metadata.bootMessage) {
                return Response.fromJSON(this.metadata.bootMessage);
            }
            
            // 如果没有定义 bootMessage，返回默认消息
            return new Response('上下文已重置。有什么我可以帮您的吗？');
        } catch (error) {
            console.error('Error in ResetContext task:', error);
            return new Response('重置上下文时发生错误。');
        }
    }
    // ... 处理其他任务
}
```