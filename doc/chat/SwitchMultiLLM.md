# 高级教程：使用动态选择组件切换多个LLM代理

在之前的教程中，我们学习了如何创建基于OpenAI、Gemini和本地LLM的代理。现在，我们将进一步探索如何在一个统一的界面中动态切换这些不同的LLM代理。这种方法不仅能让用户更灵活地选择不同的AI模型，还能方便地比较不同模型的表现。

## 准备工作

1. 确保您已完成之前的OpenAI、Gemini和本地LLM集成教程。
2. 在`.ai_helper/chat`目录下创建`multi_llm_agent`文件夹：

```shell
cd .ai_helper/chat
mkdir multi_llm_agent
cd multi_llm_agent
```

3. 初始化npm包并安装必要的依赖：

```shell
npm init -y
npm install -s ai-agent-response openai @google/generative-ai
```

4. 如果您需要使用代理来访问这些AI服务，请在`MultiModelAgent.js`文件的顶部添加以下代码：

```javascript
const { setGlobalDispatcher, ProxyAgent } = require("undici");
const dispatcher = new ProxyAgent({ uri: new URL("http://127.0.0.1:7890").toString() });
setGlobalDispatcher(dispatcher);
```

请根据您的实际代理设置调整URL。

5. 确保您有可用的OpenAI和Gemini API密钥，并且已经在My Assistant插件的设置中配置好了。可以参考以下步骤进行配置：
   - 打开VSCode，进入My Assistant插件的设置面板。
   - 在"API Keys"部分，添加以下键值对：
     - 键: "openai", 值: 您的OpenAI API密钥
     - 键: "gemini", 值: 您的Google AI Studio API密钥

6. 如果您打算使用本地LLM，确保已安装并配置好LM Studio或其他兼容的本地LLM服务。


## 创建MultiModelAgent

首先，让我们来看看`MultiModelAgent`的核心结构：

1. 在`.ai_helper/chat/multi_llm_agent`目录下创建`MultiModelAgent.js`文件：

```javascript
const { Response } = require('ai-agent-response');
const OpenAI = require('openai');
const { GoogleGenerativeAI } = require("@google/generative-ai");

// 如果需要代理，可以使用下面的代码，使用之前要安装相关依赖： `npm install -s undici` 
//const { setGlobalDispatcher, ProxyAgent } = require("undici");
//const dispatcher = new ProxyAgent({ uri: new URL("http://127.0.0.1:7890").toString() });
//setGlobalDispatcher(dispatcher);

class MultiModelAgent {
    constructor(metadata, settings) {
        this.metadata = metadata;
        this.settings = settings;
        
        // 直接使用 settings.currentModel
        // const currentModel = JSON.parse(this.settings.currentModel);
        const currentModel = this.settings.currentModel;
        this.currentApiKey = currentModel.apiKey;
        this.currentModelName = currentModel.model;

        // 初始化各个模型客户端
        this.openai = new OpenAI({ apiKey: this.settings.openai });
        this.gemini = new GoogleGenerativeAI(this.settings.gemini);
        this.localLLM = new OpenAI({
            apiKey: 'not-needed',
            baseURL: "http://localhost:1234/v1",
        });
    }

    async generateReply(thread) {
        let response;

        switch (this.currentApiKey) {
            case 'openai':
                response = await this.generateOpenAIReply(thread);
                break;
            case 'gemini':
                response = await this.generateGeminiReply(thread);
                break;
            case 'localLLM':
                response = await this.generateLocalLLMReply(thread);
                break;
            default:
                throw new Error('Invalid API key selected');
        }

        return new Response(response);
    }

    convertThreadToOpenAIHistory(thread) {
        return thread.messages.map(message => ({
            role: message.sender === 'user' ? 'user' : 'assistant',
            content: message.text
        }));
    }

    convertThreadToGeminiHistory(thread) {
        return thread.messages.slice(0, -1).map(message => ({
            role: message.sender === 'user' ? 'user' : 'model',
            parts: [{ text: message.text }]
        }));
    }

    async generateOpenAIReply(thread) {
        const history = this.convertThreadToOpenAIHistory(thread);
        const completion = await this.openai.chat.completions.create({
            model: this.currentModelName,
            messages: history,
        });
        return completion.choices[0].message.content;
    }

    async generateGeminiReply(thread) {
        const history = this.convertThreadToGeminiHistory(thread);
        const model = this.gemini.getGenerativeModel({ model: this.currentModelName });
        const chat = model.startChat({ history: history });
        const lastMessage = thread.messages[thread.messages.length - 1].text;
        const result = await chat.sendMessage(lastMessage);
        return result.response.text();
    }

    async generateLocalLLMReply(thread) {
        const history = this.convertThreadToOpenAIHistory(thread);
        const completion = await this.localLLM.chat.completions.create({
            model: this.currentModelName,
            messages: history,
        });
        return completion.choices[0].message.content;
    }
}

module.exports = MultiModelAgent;
```

## 配置动态选择组件

现在，我们需要在`agents.json`中配置动态选择组件。这将允许用户在对话过程中切换不同的LLM模型。

2. 更新`.ai_helper/agent/chat/agents.json`文件：

```json
{
  "agents": [
    {
      "name": "MultiModelAgent",
      "path": "./multi_llm_agent/MultiModelAgent.js",
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
            },
            {
              "label": "localLLM__gemma-2-2b",
              "value": {
                "model": "lmstudio-community/gemma-2-2b-it-GGUF/gemma-2-2b-it-Q4_K_M.gguf"
              }
            }
          ],
          "default": {
            "apiKey": "openai",
            "model": "gpt-3.5-turbo"
          }
        }
      ],
      "metadata": {}
    }
  ]
}
```

## 使用动态选择组件

配置完成后，用户可以在对话过程中动态切换不同的LLM模型。以下是使用步骤：

1. 在VSCode中打开My Assistant插件。
2. 创建一个新的聊天会话，选择"MultiModelAgent"作为代理。
3. 在聊天界面上，你会看到一个下拉菜单，标记为"AI Platform"。
4. 通过这个下拉菜单，你可以随时切换不同的AI模型：
   - OpenAI GPT-3.5
   - Google Gemini 1.5
   - 本地LLM（Gemma 2B）

5. 每次切换模型后，新的对话将使用所选的模型进行回复。

## 代码解析

让我们深入了解`MultiModelAgent`是如何工作的：

1. **构造函数**：
   - 初始化所有可能使用的AI客户端（OpenAI、Gemini、本地LLM）。
   - 从`settings`中读取当前选择的模型信息。

2. **generateReply方法**：
   - 根据当前选择的API键（`this.currentApiKey`）来决定使用哪个AI模型生成回复。
   - 调用相应的方法（如`generateOpenAIReply`、`generateGeminiReply`等）来处理请求。

3. **动态选择组件配置**：
   - 在`agents.json`中，我们定义了一个`operations`数组，包含了一个类型为`setting`的操作。
   - 这个操作使用`select`控件，允许用户选择不同的AI平台。
   - `options`数组定义了可选择的模型，包括它们的标签和对应的配置值。
   - `default`字段设置了默认选择的模型。

## 优势和应用场景

1. **模型比较**：用户可以轻松比较不同AI模型的性能和回答质量。
2. **灵活性**：根据不同的任务需求，快速切换到最合适的模型。
3. **本地与云端结合**：在需要隐私保护时使用本地模型，需要强大性能时切换到云端模型。
4. **开发和测试**：开发者可以方便地测试他们的应用在不同AI模型下的表现。

## 注意事项

1. 确保所有需要的API密钥都已正确配置在My Assistant的设置中。
2. 使用本地LLM时，请确保LM Studio或其他本地LLM服务正在运行。
3. 网络连接对于云端模型（OpenAI和Gemini）是必要的，而本地模型则不受影响。

## 下一步

1. 尝试添加更多的AI模型或服务到选项中。
2. 实现一个功能，允许用户保存和比较不同模型的回答。
3. 添加模型性能指标，帮助用户更好地选择适合的模型。

通过这个高级教程，你现在能够创建一个灵活的多模型AI助手，让用户可以根据需求自由切换不同的AI模型。这不仅提高了用户体验，也为进一步的AI应用开发提供了强大的基础。