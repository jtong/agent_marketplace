

## Response 数据结构

首先，让我们看一下 Response 的基本结构：

```javascript
class Response {
    constructor(message = '', isHtml = false) {
        this.message = message;
        this.isHtml = isHtml;
        this.stream = null;
        this.meta = {};
        this.availableTasks = [];
        this.updateLastMessage = false;
        this.nextTask = null;
    }
}
```

## Response 属性详解

1. message（响应消息）
   - 用途：存储 AI 的响应文本。
   - UI 显示：作为机器人消息的主体内容显示在聊天界面。
   - 不显示情况：当使用流式响应时，这个属性可能不会被直接使用。

2. isHtml（是否为 HTML）
   - 用途：指示消息内容是否应该被解析为 HTML。
   - UI 显示：影响消息的渲染方式。如果为 true，消息会被解析为 HTML；否则作为纯文本或 Markdown 渲染。
   - 显示逻辑：
     ```javascript
     if (message.isHtml) {
         textContainer.innerHTML = message.text;
     } else {
         textContainer.innerHTML = renderMarkdown(message.text);
     }
     ```

3. stream（流式响应）
   - 用途：用于流式传输大量文本或长时间运行的响应。
   - UI 显示：不直接显示，但用于逐步更新聊天界面的内容。
   - 显示逻辑：
     ```javascript
     if (response.isStream()) {
         for await (const chunk of response.getStream()) {
             botMessage.text += chunk;
             panel.webview.postMessage({
                 type: 'updateBotMessage',
                 messageId: botMessage.id,
                 text: chunk
             });
         }
     }
     ```

4. meta（元数据）
   - 用途：存储额外的响应相关信息。
   - UI 显示：通常不直接显示，但可能用于控制 UI 行为或存储上下文信息。
   - 不显示情况：对用户隐藏，主要用于内部逻辑。

5. availableTasks（可用任务）
   - 用途：存储与此响应相关的可执行任务列表。
   - UI 显示：通常显示为消息下方的一系列按钮。
   - 显示逻辑：
     ```javascript
     if (response.hasAvailableTasks()) {
         addTaskButtons(messageElement, response.getAvailableTasks());
     }
     ```

6. updateLastMessage（更新最后一条消息）
   - 用途：指示是否应该更新聊天历史中的最后一条消息，而不是添加新消息。
   - UI 显示：影响消息的添加或更新方式。
   - 显示逻辑：
     ```javascript
     if (response.shouldUpdateLastMessage()) {
         // 更新最后一条消息的逻辑
     } else {
         // 添加新消息的逻辑
     }
     ```
7. nextTask（下一个任务）
   - 用途：指定在当前响应后应自动执行的下一个任务。
   - UI 显示：不直接显示，但影响后续交互流程。
   - 不显示情况：对用户隐藏，主要用于内部逻辑控制。

## Response 使用场景示例

1. 简单文本响应
   ```javascript
   new Response("这是一个简单的文本响应。")
   ```
   - UI 表现：在聊天界面显示纯文本消息。

2. HTML 响应
   ```javascript
   new Response("<p>这是一个 <strong>HTML</strong> 响应。</p>", true)
   ```
   - UI 表现：在聊天界面显示格式化的 HTML 内容。

3. 流式响应
   ```javascript
   const response = new Response();
   response.setStream(asyncGenerator);
   ```
   - UI 表现：消息内容会逐步显示，给用户实时反馈的感觉。

4. 给最后一条消息添加任务按钮的响应
   ```javascript
   const response = new Response("这是一个带任务的响应。");
   response.setUpdateLastMessage(true);
   response.addAvailableTask(new AvailableTask("执行任务", new Task({/*...*/})));
   ```
   - UI 表现：显示消息文本，下方有一个"执行任务"按钮。

5. 计划响应（Plan Response）
   ```javascript
   const response = new Response();
   response.setPlanTasks([
     new Task({
       name: "分析代码",
       type: Task.TYPE_ACTION,
       message: "正在分析代码复杂度...",
       skipUserMessage: true
     }),
     new Task({
       name: "生成报告",
       type: Task.TYPE_ACTION,
       message: "正在生成分析报告...",
       skipUserMessage: true
     })
   ]);
   return response;
   ```
   - UI 表现：系统会忽略Response里的FullMessage，立刻依次执行计划中的任务，每个任务可能会生成独立的消息或更新现有消息，具体取决于任务的配置。

6. 带有后续任务的响应
   ```javascript
   const response = new Response("这是当前任务的响应。");
   response.setNextTask(new Task({
       name: "FollowUpAction",
       type: Task.TYPE_ACTION,
       message: "执行后续操作",
       skipUserMessage: true
   }));
   return response;
   ```
   - UI 表现：首先显示当前响应的消息，然后自动执行后续任务，可能会生成额外的消息或更新现有消息。

7. 带有后续任务的响应
   ```javascript
   const response = new Response("这是当前任务的响应。");
   response.setNextTask(new Task({
       name: "FollowUpAction",
       type: Task.TYPE_ACTION,
       message: "执行后续操作",
       skipUserMessage: true
   }));   

 

## AvailableTask 数据结构

AvailableTask 代表一个可供用户选择的任务，通常与 UI 中的按钮相关联。现在，让我们看看 AvailableTask 的结构：

```javascript
class AvailableTask {
    constructor(name, task) {
        this.name = name;
        this.task = task;
    }
}
```

## AvailableTask 属性详解

1. name（任务名称）
   - 用途：为任务提供一个用户友好的名称。
   - UI 显示：通常显示为任务按钮的文本。
   - 不显示情况：几乎总是显示，除非特意隐藏。

2. task（任务对象）
   - 用途：包含任务的详细信息和执行逻辑。
   - UI 显示：不直接显示，但其属性可能影响按钮的行为和外观。
   - 不显示情况：任务详情对用户隐藏，仅在内部使用。

## AvailableTask 在 UI 中的应用

AvailableTask 主要用于在聊天界面中创建可交互的任务按钮。

显示逻辑：
```javascript
function addTaskButtons(container, availableTasks) {
    const taskContainer = document.createElement('div');
    taskContainer.className = 'task-buttons-container';

    availableTasks.forEach(availableTask => {
        const button = document.createElement('button');
        button.textContent = availableTask.name;
        button.className = 'task-button';
        button.addEventListener('click', () => executeTask(availableTask.task));
        taskContainer.appendChild(button);
    });

    container.appendChild(taskContainer);
}
```

## AvailableTask 使用场景示例

1. 简单操作任务
   ```javascript
   new AvailableTask("刷新数据", new Task({
       name: "RefreshData",
       type: Task.TYPE_ACTION,
       message: "正在刷新数据...",
       skipUserMessage: true
   }))
   ```
   - UI 表现：显示一个"刷新数据"按钮，点击后不显示用户消息，但会执行刷新操作。

2. 带用户输入的任务
   ```javascript
   new AvailableTask("提问", new Task({
       name: "AskQuestion",
       type: Task.TYPE_MESSAGE,
       message: "请输入您的问题"
   }))
   ```
   - UI 表现：显示一个"提问"按钮，点击后会提示用户输入问题，并显示用户的输入作为新的消息。

3. 复杂交互任务
   ```javascript
   new AvailableTask("生成报告", new Task({
       name: "GenerateReport",
       type: Task.TYPE_ACTION,
       message: "正在生成报告...",
       skipUserMessage: true,
       skipBotMessage: false,
       meta: { reportType: "monthly" }
   }))
   ```
   - UI 表现：显示一个"生成报告"按钮，点击后不显示用户消息，但会显示生成报告的过程和结果。

通过这些示例，我们可以看到 Response 和 AvailableTask 如何协同工作，以创建丰富的交互体验。Response 对象控制整体的回复结构和内容，而 AvailableTask 则提供了额外的交互点，允许用户执行特定的操作或启动新的对话流程。这种灵活性使得开发者可以创建复杂的对话系统，同时保持用户界面的简洁和直观。

## Task 数据结构

Task 代表一个可执行的任务，它与 UI 中的按钮无直接关系。Task 通常是由 Agent 内部逻辑或系统流程调用执行的。

```javascript
class Task {
    static TYPE_MESSAGE = 'message';
    static TYPE_ACTION = 'action';

    constructor({
        name,
        type,
        message,
        meta = {},
        host_utils = null,
        skipUserMessage = false,
        skipBotMessage = false
    }) {
        this.name = name;
        this.type = type;
        this.message = message;
        this.meta = meta;
        this.host_utils = host_utils;
        this.skipUserMessage = skipUserMessage;
        this.skipBotMessage = skipBotMessage;
    }

    isMessageTask() {
        return this.type === Task.TYPE_MESSAGE;
    }
}
```

### 属性说明

- `name`: 任务的唯一标识符。
- `type`: 任务类型，可以是 `TYPE_MESSAGE` 或 `TYPE_ACTION`。
- `message`: 与任务相关的消息内容。
- `meta`: 存储任务相关的额外元数据。
- `host_utils`: 提供宿主环境（VSCode 插件）的辅助功能。
- `skipUserMessage`: 是否在执行任务时跳过显示用户消息。
- `skipBotMessage`: 是否在执行任务时跳过显示机器人消息。

通过灵活使用这些属性，特别是 `skipUserMessage` 和 `skipBotMessage`，我们可以创建各种类型的任务，从完全可见的交互式任务到完全在后台运行的静默任务。这种灵活性允许开发者根据具体需求定制任务的 UI 行为，提升用户体验和界面的清晰度。