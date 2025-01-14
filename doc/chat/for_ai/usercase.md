## 向消息添加按钮

用于在某条消息下方添加可点击按钮，让用户能对该消息进行特定操作。

核心代码示例:
```javascript
const { Response } = require('ai-agent-response');

class ButtonDemoAgent {
    async generateReply(thread) {
        const response = new Response();
        response.setStream(this.fakeStream());
        // 流结束后添加按钮
        response.addNextTask(new Task({
            name: "AddButtons",
            type: Task.TYPE_ACTION,
            skipUserMessage: true,
            skipBotMessage: true
        }));
        return response;
    }

    async executeTask(task) {
        if(task.name === "AddButtons") {
            const response = new Response();
            response.setUpdateLastMessage(true); // 更新最后消息而不是新增
            response.addAvailableTask(new AvailableTask("查看分析", new Task({/*...*/})));
            return response;
        }
    }
}
```

关键点:
- setUpdateLastMessage(true): 表示更新已有消息
- skipBotMessage: true 表示不新增机器人消息
- addAvailableTask: 添加按钮配置

## 添加标记消息

用于在对话中插入分隔线，将对话分成不同的段落或上下文。

核心代码示例:
```javascript
class MarkerDemoAgent {
    async executeTask(task, thread) {
        if(task.name === "AddMarker") {
            const { threadRepository, postMessage } = task.host_utils;
            
            // 添加标记
            const markerId = threadRepository.addMarker(thread);
            
            // 通知前端更新UI
            postMessage({
                type: 'markerAdded',
                thread: thread,
                markerId  
            });

            return new Response();
        }
    }
}
```

关键点:
- threadRepository.addMarker: 添加标记到对话
- postMessage: 通知前端显示标记线

## 引导消息(Boot Message)

用于在用户开始新对话或重置上下文时显示的引导性消息，帮助用户了解当前Agent的功能。

核心代码示例:
```javascript
class BootMessageDemoAgent {
    async executeTask(task, thread) {
        if(task.name === "ResetContext") {
            const { threadRepository } = task.host_utils;
            
            // 添加标记
            threadRepository.addMarker(thread);
            
            // 返回引导消息
            if(this.metadata?.bootMessage) {
                return Response.fromJSON(this.metadata.bootMessage);
            }
            
            return new Response("上下文已重置");
        }
    }
}

// agents.json配置
{
    "metadata": {
        "bootMessage": {
            "message": "欢迎使用...",
            "isHtml": false,
            "meta": {
                "isBootMessage": true,
                "isVirtual": true
            },
            "availableTasks": [/*...*/]
        }
    }
}
```

关键点:
- bootMessage定义在agent配置的metadata中
- isVirtual: true 表示虚拟消息不计入上下文
- Response.fromJSON: 从配置创建response

## 添加操作按钮 (Operations)

用于在对话界面底部添加常驻的操作控件(如按钮、下拉框等)，这些控件在仅对当前对话线程(Thread)有效。


核心代码示例:
```javascript
// agents.json
{
  "agents": [
    {
      "name": "OperationButtonAgent",
      "operations": [
        {
          "type": "setting",  // type主要分为 setting 和 task 两类
          "control": "select", // UI控件类型
          "name": "AI Platform",
          "settingKey": "currentModel",
          "options": [
            {
              "label": "openai_option", 
              "value": {
                "apiKey": "openai",
                "model": "model_name" 
              }
            }
          ]
        },
        {
          "type": "task",
          "control": "button", // button是最常用的UI控件类型
          "name": "重置上下文",
          "task": {
            "name": "ResetContext",
            "type": "action",
            "skipUserMessage": true
          }
        }
      ]
    }
  ]
}
```

关键点:
- type表示操作类型: setting(更新配置)或task(执行任务)
- control表示UI控件类型: select、button等
- 虽然setting通常用select, task通常用button,但它们是独立的概念
- operations在agents.json中配置,使UI操作更容易管理和修改