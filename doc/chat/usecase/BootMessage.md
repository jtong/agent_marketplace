# 5.1 引导消息（Boot Message）

引导消息是一种特殊的虚拟消息，通常在创建新的对话线程时自动添加。它的目的是为用户提供初始指导或欢迎信息。引导消息通过Agent的配置文件中的metadata来定义。

##  Boot Message 配置

在 `agents.json` 配置文件中，每个Agent可以定义自己的bootMessage。这个配置位于Agent的metadata字段中。例如：

```json
{
  "agents": [
    {
      "name": "ExampleAgent",
      "path": "./path/to/ExampleAgent.js",
      "metadata": {
        "bootMessage": {
          "text": "欢迎使用ExampleAgent！我可以帮助您完成各种任务。",
          "availableTasks": [
            {
              "name": "开始新任务",
              "task": {
                "name": "StartNewTask",
                "type": "ACTION",
                "message": "请选择您想开始的任务类型"
                /** 其他的task属性 **/
              }
            },
            {
              "name": "查看帮助",
              "task": {
                "name": "ShowHelp",
                "type": "ACTION",
                "message": "显示帮助信息"
                /** 其他的task属性 **/
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

- `text` (字符串，必填): 显示给用户的欢迎或引导文本。
- `availableTasks` (数组，可选): 一组可供用户选择的初始任务。

每个 availableTask 包含：
- `name` (字符串): 显示给用户的任务名称。
- `task` (对象): 描述任务的详细信息。
  - `name` (字符串): 任务的唯一标识符。
  - `type` (字符串): 任务类型，通常是 "ACTION"。
  - `message` (字符串): 执行任务时显示的消息或提示。

## Agent自行过滤虚拟消息

在处理消息时，系统会检查 isVirtual 标志。如果为 true，这些消息会被显示在用户界面上，但在构造发送给AI的消息历史时会被过滤掉。但过滤掉虚拟消息的功能需要Agent在消息处理逻辑中自行实现，例如：

```javascript
convertToAIMessages(messages) {
    return messages.filter(msg => !msg.isVirtual).map(message => ({
        role: message.sender === 'user' ? 'user' : 'assistant',
        content: message.text
    }));
}
```