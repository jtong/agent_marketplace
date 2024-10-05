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
          "model": "gemini-1.0-pro"
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


## 第三部分：添加本地 LLM 流式响应代理

在完成了 OpenAI 和 Gemini 的集成之后，我们现在将介绍如何集成本地运行的 LLM，这对于那些希望在本地环境中使用 AI 模型或者需要更多隐私控制的用户来说非常有用。我们将使用 LM Studio 提供的本地 API，它模仿了 OpenAI 的 API 接口。

### 准备工作

1. 确保您已经安装了 LM Studio 并运行了一个本地 LLM 模型。
2. 记下 LM Studio 提供的本地 API 地址，通常是 `http://localhost:1234/v1`。
您说得对，我们确实应该考虑到可能有读者直接从这一部分开始阅读。我会在准备工作部分添加依赖安装的说明。以下是修改后的准备工作部分：

### 准备工作

1. 确保您已经安装了 LM Studio 并运行了一个本地 LLM 模型。
2. 记下 LM Studio 提供的本地 API 地址，通常是 `http://localhost:1234/v1`。
3. 如果您跳过了前面直接从本部分开始读，那么你可能还没有设置项目环境，请按照以下步骤进行：

   a. 在 `.ai_helper/chat` 目录下创建 `llm_chat_agent` 文件夹（如果还不存在）：

   ```shell
   cd .ai_helper/chat
   mkdir -p llm_chat_agent
   cd llm_chat_agent
   ```

   b. 初始化 npm 包并安装必要的依赖：

   ```shell
   npm init -y
   npm install -s ai-agent-response openai
   ```

### 创建本地 LLM 流式响应代理

1. 在 `llm_chat_agent` 文件夹中创建一个新文件 `LocalLLMStreamAgent.js`：

```javascript
const { Response } = require('ai-agent-response');
const OpenAI = require('openai');

class LocalLLMStreamAgent {
    constructor(metadata, settings) {
        this.metadata = metadata;
        this.settings = settings;
        const llmConfig = {
            apiKey: 'not-needed', // 虽然不需要，但是如果你没有设置apiKey属性运行时会报错，所以还是要随便设置个值。
            baseURL: this.metadata.llm.baseURL,
            model: this.metadata.llm.model
        };
        this.openai = new OpenAI(llmConfig);
    }

    async generateReply(thread) {
        const messages = this.convertToOpenAIMessages(thread.messages);
        const stream = await this.openai.chat.completions.create({
            model: this.metadata.llm.model,
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

module.exports = LocalLLMStreamAgent;
```

2. 更新 `.ai_helper/agent/chat/agents.json` 以包含本地 LLM 代理:

```json
{
  "agents": [
    {
      "name": "LocalLLMStreamAgent",
      "path": "./llm_chat_agent/LocalLLMStreamAgent.js",
      "metadata": {
        "llm": {
          "model": "local-model",
          "baseURL": "http://localhost:1234/v1"
        }
      }
    }
  ]
}
```



### API 配置

在 VSCode 的 My Assistant 插件设置中，添加本地 LLM 的 API 密钥（如果需要的话）：
- 键: "local_llm", 值: "不需要填写实际的 API 密钥"

### 代码解析

- **构造函数**：初始化 OpenAI 客户端，使用本地 LLM 的配置。
- **generateReply 方法**：创建一个流式聊天完成请求，并设置流式响应。
- **convertToOpenAIMessages 方法**：将 My Assistant 的消息格式转换为 OpenAI API 所需的格式。
- **createStream 生成器方法**：遍历 API 返回的流，逐步 yield 内容，实现流式效果。

### 使用 Meta 配置

在 `agents.json` 中，我们使用元数据来配置本地 LLM 代理：

- `apiKey`：指定在设置中存储 API 密钥的键名（对于本地 LLM 可能不需要实际的密钥）。
- `model`：指定要使用的本地模型名称。
- `baseURL`：指定本地 LLM API 的基础 URL。

### 激活您的 Agent

完成上述步骤后，刷新或重启 VSCode 来使新添加的本地 LLM 流式代理生效。

### 运行效果

现在，您可以在创建新对话时选择 "LocalLLMStreamAgent"。这个 Agent 会提供流畅的流式响应体验，但响应内容将来自您本地运行的 LLM 模型。

### 注意事项

1. 确保 LM Studio 或其他提供本地 LLM 服务的软件在使用 Agent 之前已经启动并运行。
2. 本地 LLM 的性能和响应质量可能与云端 AI 服务有所不同，这取决于您使用的模型和硬件配置。
3. 使用本地 LLM 可以提供更好的隐私保护和更低的延迟，但可能需要较高的本地计算资源。


## 下一步

1. 实现一个通用的LLM接口，使得添加新的AI服务变得更加容易。
2. 创建一个Agent选择器，允许用户在对话中动态切换不同的AI服务。
3. 实现性能比较功能，帮助用户了解不同AI服务在特定任务上的表现。
4. 探索如何结合多个AI服务的优势，创建更强大的混合Agent。

通过这种方法，您可以轻松地在My Assistant中集成和比较多个AI服务，为用户提供更丰富的AI交互体验。祝您在My Assistant开发之旅中取得成功！