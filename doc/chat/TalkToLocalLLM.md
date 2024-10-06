# 高级教程：集成本地 LLM 流式响应代理

在完成了 OpenAI 和 Gemini 的集成之后，我们将介绍如何将本地运行的大语言模型（LLM）整合到我们的系统中。这对于那些希望在本地环境中使用 AI 模型或需要更高隐私控制的用户来说尤为重要。我们将使用 LM Studio 提供的本地 API，该 API 模仿了 OpenAI 的接口，使得集成过程变得相对简单。

## 准备工作

1. 确保您已经安装并运行了 LM Studio，且已加载了一个本地 LLM 模型。
2. 记录 LM Studio 提供的本地 API 地址，通常为 `http://localhost:1234/v1`。
3. 如果您是直接从本节开始阅读，那么你可能还没有设置项目环境，请按照以下步骤设置项目环境：

   a. 在 `.ai_helper/chat` 目录下创建 `llm_chat_agent` 文件夹（如果尚不存在）：

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

## 创建本地 LLM 流式响应代理

1. 在 `llm_chat_agent` 文件夹中创建一个新文件 `LocalLLMStreamAgent.js`：

```javascript
const { Response } = require('ai-agent-response');
const OpenAI = require('openai');

class LocalLLMStreamAgent {
    constructor(metadata, settings) {
        this.metadata = metadata;
        this.settings = settings;
        const llmConfig = {
            apiKey: 'not-needed', // 虽然不需要，但是如果没有设置apiKey属性，运行时会报错。所以还是要随便设置个值。
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
          "model": "local-model-name",
          "baseURL": "http://localhost:1234/v1"
        }
      }
    }
  ]
}
```

## API 配置

在 VSCode 的 My Assistant 插件设置中，添加本地 LLM 的模型信息：
- 键: "model"
- 值: 模型名称字符串，通常可在 LM Studio > Developer > Local Server 界面的模型卡片处找到。例如，对于 gemma-2-2b 模型，值可能类似：`lmstudio-community/gemma-2-2b-it-GGUF/gemma-2-2b-it-Q4_K_M.gguf`。

## 代码解析

- **构造函数**：初始化 OpenAI 客户端，使用本地 LLM 的配置。
- **generateReply 方法**：创建流式聊天完成请求，并设置流式响应。
- **convertToOpenAIMessages 方法**：将 My Assistant 的消息格式转换为 OpenAI API 所需的格式。
- **createStream 生成器方法**：遍历 API 返回的流，逐步 yield 内容，实现流式效果。

## 使用元数据配置

在 `agents.json` 中，我们使用元数据来配置本地 LLM 代理：

- `model`：指定要使用的本地模型名称。
- `baseURL`：指定本地 LLM API 的基础 URL。

## 激活您的代理

完成上述步骤后，刷新或重启 VSCode 来使新添加的本地 LLM 流式代理生效。

## 运行效果

现在，您可以在创建新对话时选择 "LocalLLMStreamAgent"。这个代理会提供流畅的流式响应体验，响应内容将来自您本地运行的 LLM 模型。

## 注意事项

1. 确保在使用代理之前，LM Studio 或其他提供本地 LLM 服务的软件已启动并运行。
2. 本地 LLM 的性能和响应质量可能与云端 AI 服务有所不同，这取决于您使用的模型和硬件配置。
3. 使用本地 LLM 可以提供更好的隐私保护和更低的延迟，但可能需要较高的本地计算资源。
4. 定期更新您的本地模型，以确保获得最新的改进和功能。

通过集成本地 LLM，您现在拥有了一个更加灵活和私密的 AI 助手系统。这不仅增强了您的隐私控制，还为进一步自定义和优化 AI 模型开辟了新的可能性。