# 高级教程：创建基于LLM的流式响应聊天代理

在完成了[快速开始指南](doc/chat/QuickStart.md)和[流式响应与Meta配置指南](doc/chat/StreamResponse&Meta.md)之后，我们将学习如何创建使用在线AI的 API的流式响应聊天代理。这个教程将展示如何集成外部AI服务，并利用流式响应提供流畅的用户体验。

## 第一部分：创建OpenAI流式响应代理

### 准备工作

1. 确保您已完成之前的两个指南中的步骤。
2. 您需要一个有效的OpenAI API密钥。如果还没有，请在[OpenAI网站](https://openai.com/)上注册并获取API密钥。

### 创建OpenAI流式响应代理

1. 在 `.ai_helper/chat` 目录下创建一个新文件夹:

```shell
cd .ai_helper/chat
mkdir llm_chat_agent
cd llm_chat_agent
```

2. 初始化为npm包并安装必要的依赖:

```shell
npm init -y
npm install -s ai-agent-response openai
```

3. 创建OpenAI流式响应代理文件 `OpenAIStreamAgent.js`:

```javascript
const { Response } = require('ai-agent-response');
const OpenAI = require('openai');

class OpenAIStreamAgent {
    constructor(metadata, settings) {
        this.metadata = metadata;
        this.settings = settings;
        this.openai = new OpenAI({
            apiKey: this.settings[this.metadata.llm.apiKey],
        });
        this.model = this.metadata.llm.model;
    }

    async generateReply(thread) {
        const messages = this.convertToOpenAIMessages(thread.messages);
        const stream = await this.openai.chat.completions.create({
            model: this.model,
            messages: messages,
            stream: true,
        });

        const response = new Response('');
        response.setStream(this.createStream(stream));
        return response;
    }

    convertToOpenAIMessages(messages) {
        return messages.map(message => ({
            role: message.sender === 'user' ? 'user' : 'assistant',
            content: message.text
        }));
    }

    async *createStream(stream) {
        for await (const chunk of stream) {
            const content = chunk.choices[0]?.delta?.content || '';
            if (content) {
                yield content;
            }
        }
    }
}

module.exports = OpenAIStreamAgent;
```

4. 在 `.ai_helper/agent/chat/agents.json` 中添加新的agent配置:

```json
{
  "agents": [
    {
      "name": "OpenAIStreamAgent",
      "path": "./llm_chat_agent/OpenAIStreamAgent.js",
      "metadata": {
        "llm": {
          "apiKey": "openai",
          "model": "gpt-3.5-turbo"
        }
      }
    }
  ]
}
```

### API密钥配置

1. 在VSCode中，使用My Assistant插件打开设置面板。
2. 在 "API Keys" 部分，添加一个新的键值对：
   - 键(Key)：输入 "openai"
   - 值(Value)：输入您的OpenAI API密钥

### 代码解析

- **构造函数**：初始化OpenAI客户端，使用从设置中获取的API密钥。
- **generateReply方法**：创建一个流式聊天完成请求，并设置流式响应。
- **convertToOpenAIMessages方法**：将My Assistant的消息格式转换为OpenAI API所需的格式。
- **createStream生成器方法**：遍历API返回的流，逐步yield内容，实现流式效果。

### 使用Meta配置

在 `agents.json` 中，我们使用元数据来配置OpenAI代理：

- `apiKey`：指定在设置中存储API密钥的键名。
- `model`：指定要使用的OpenAI模型。

### 激活您的Agent

完成上述步骤后，点击VSCode中的刷新按钮或重启VSCode来使新的OpenAI流式代理生效。

## 第二部分：添加Gemini流式响应代理

现在，我们将在同一个文件夹中添加Google Gemini API的支持，展示如何在一个项目中管理多个AI服务的Agent。

### 准备工作

1. 您需要一个有效的Google AI Studio API密钥。如果还没有，请在[Google AI Studio](https://makersuite.google.com/app/apikey)上注册并获取API密钥。

### 添加Gemini流式响应代理

1. 在之前创建的 `llm_chat_agent` 文件夹中，安装Gemini API所需的依赖:

```shell
npm install -s @google/generative-ai
```

2. 在同一文件夹中创建Gemini流式响应代理文件 `GeminiStreamAgent.js`:

```javascript
const { Response } = require('ai-agent-response');
const { GoogleGenerativeAI } = require("@google/generative-ai");

class GeminiStreamAgent {
    constructor(metadata, settings) {
        this.metadata = metadata;
        this.settings = settings;
        const genAI = new GoogleGenerativeAI(this.settings[this.metadata.llm.apiKey]);
        this.model = genAI.getGenerativeModel({ model: this.metadata.llm.model });
    }

    async generateReply(thread) {
        const history = this.convertToGeminiHistory(thread.messages);
        const chat = this.model.startChat({ history });
        const result = await chat.sendMessageStream(thread.messages[thread.messages.length - 1].text);

        const response = new Response('');
        response.setStream(this.createStream(result.stream));
        return response;
    }

    convertToGeminiHistory(messages) {
        return messages.slice(0, -1).map(message => ({
            role: message.sender === 'user' ? 'user' : 'model',
            parts: [{ text: message.text }]
        }));
    }

    async *createStream(stream) {
        for await (const chunk of stream) {
            const chunkText = chunk.text();
            yield chunkText;
        }
    }
}

module.exports = GeminiStreamAgent;
```

3. 更新 `.ai_helper/agent/chat/agents.json` 以包含Gemini代理:

```json
{
  "agents": [
    {
      "name": "OpenAIStreamAgent",
      "path": "./llm_chat_agent/OpenAIStreamAgent.js",
      "metadata": {
        "llm": {
          "apiKey": "openai",
          "model": "gpt-3.5-turbo"
        }
      }
    },
    {
      "name": "GeminiStreamAgent",
      "path": "./llm_chat_agent/GeminiStreamAgent.js",
      "metadata": {
        "llm": {
          "apiKey": "gemini",
          "model": "gemini-1.5-flash"
        }
      }
    }
  ]
}
```

### API密钥配置

在VSCode的My Assistant插件设置中，添加Gemini的API密钥：
- 键: "gemini", 值: 您的Google AI Studio API密钥

### 代码组织和多Agent管理

这个例子展示了如何在同一个文件夹（`llm_chat_agent`）中管理多个不同AI服务的Agent。这种方法有以下优点：

1. **代码组织**：相关的Agent代码都集中在一个地方，便于管理和维护。
2. **灵活性**：可以轻松添加新的AI服务Agent，而不需要创建新的文件夹结构。
3. **配置简化**：在`agents.json`中，我们可以轻松地引用同一文件夹下的不同Agent文件。
4. **共享资源**：如果有共同的辅助函数或配置，可以很方便地在这些Agent之间共享。

### 使用Meta配置

注意在`agents.json`中，我们为每个Agent使用了不同的元数据配置。这种方式允许我们灵活地配置每个Agent，而无需修改Agent的代码。

### 激活您的Agents

完成上述步骤后，再次刷新或重启VSCode来使新添加的Gemini流式代理生效。

### 运行效果

现在，您可以在创建新对话时选择"OpenAIStreamAgent"或"GeminiStreamAgent"。两者都会提供流畅的流式响应体验，但可能会因为底层AI模型的不同而产生不同的回答。

## 下一步

1. 实现一个通用的LLM接口，使得添加新的AI服务变得更加容易。
2. 创建一个Agent选择器，允许用户在对话中动态切换不同的AI服务。
3. 实现性能比较功能，帮助用户了解不同AI服务在特定任务上的表现。
4. 探索如何结合多个AI服务的优势，创建更强大的混合Agent。

通过这种方法，您可以轻松地在My Assistant中集成和比较多个AI服务，为用户提供更丰富的AI交互体验。祝您在My Assistant开发之旅中取得成功！