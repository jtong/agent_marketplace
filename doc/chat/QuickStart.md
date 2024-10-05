# 快速开始：开发您的第一个 My Assistant 插件

## 准备工作

1. 安装 My Assistant 插件到您的 VSCode。
2. 打开一个新的工作文件夹。

## 自动初始化

安装插件后,打开一个新文件夹,插件会自动为您创建以下目录结构:

```
.
└── .ai_helper
    └── agent
        ├── app
        │   └── apps.json
        ├── chat
        │   └── agents.json
        ├── job
        │   └── agents.json
        └── memory_repo
            ├── chat_threads
            │   └── threads.json
            └── job_threads
                └── threads.json
```

## 创建您的第一个 Agent

1. 在 `.ai_helper/chat` 目录下创建一个新文件夹:

```shell
cd .ai_helper/chat
mkdir hello_world_agent
cd hello_world_agent
```

2. 初始化为 npm 包并安装依赖:

```shell
npm init -y
npm install -s ai-agent-response
```

3. 创建入口文件 `entryAgent.js`:

```javascript
const { Response } = require('ai-agent-response');

class EntryAgent {
    constructor(metadata) {
        this.metadata = metadata;
    }

    generateReply(thread, host_utils) {
        const lastMessage = thread.messages[thread.messages.length - 1];
        const replyText = `回复: ${lastMessage.text}`;
        return new Response(replyText);
    }
}

module.exports = EntryAgent;
```

4. 在 `.ai_helper/agent/chat/agents.json` 中添加新的 agent 配置:

```json
{
  "agents": [
    {
      "name": "HelloWorldAgent",
      "path": "./hello_world_agent/entryAgent.js",
      "metadata": {}
    }
  ]
}
```

## 激活您的 Agent

完成上述步骤后,您有两种方式使您的新 Agent 生效:

1. 点击 VSCode 中的刷新按钮
2. 重启 VSCode

之后,您就可以在创建新对话时选择您的 "HelloWorldAgent" 了。

## 运行效果

当您成功创建并激活 HelloWorldAgent 后,它的运行效果如下:

1. 在 VSCode 中,使用 My Assistant 插件创建一个新的对话。
2. 在 Agent 选择列表中,您会看到并可以选择 "HelloWorldAgent"。
3. 选择 HelloWorldAgent 后,开始与之对话。
4. 无论您输入什么消息,Agent 都会以 "回复: " 加上您的原始消息作为回应。

例如:

```
用户: "你好,世界!"
HelloWorldAgent: "回复: 你好,世界!"
```

```
用户: "今天天气真不错"
HelloWorldAgent: "回复: 今天天气真不错"
```
这个简单的 HelloWorldAgent 演示了 My Assistant 插件的基本工作原理。它接收用户的输入,并通过预定义的逻辑生成回复。虽然这个例子很基础,但它为您开发更复杂、更智能的 Agent 奠定了基础。

## 查看完整代码

您可以在以下链接查看完整的代码示例：

[HelloWorldAgent 完整代码](https://github.com/jtong/my_assistant_agent_examples/tree/main/chat/hello_world_agent)

这个文件夹里包含了 `entryAgent.js` 和 `package.json` 文件,您可以参考这些文件来确保您的本地实现是正确的,或者探索更多的实现细节。


## 下一步

现在您已经成功创建并了解了您的第一个 My Assistant 插件,您可以:

- 尝试修改 `generateReply` 方法,实现更复杂的回复逻辑
- 探索如何使用 `metadata` 来配置您的 Agent
- 学习如何处理多轮对话,记住上下文信息
- 集成外部 API 或数据源,使您的 Agent 能够提供实时信息或执行特定任务
- 开发针对特定领域或用例的智能助手,如代码审查助手、写作辅助工具等

记住,这只是开始!随着您对 My Assistant 开发的深入,您将能够创建出更加强大和有用的 Agent。

祝您在 My Assistant 开发之旅中获得乐趣和成功!