---
tags: [concept, API, LLM, OpenAI]
type: concept
created: 2026-07-07
updated: 2026-07-08
---

# 大模型 API 请求规范

大模型的API请求规范，目前业界实际上形成了一个**事实标准**——即**OpenAI的Chat Completions API格式**。绝大多数主流模型（如智谱GLM、百度文心、阿里通义、DeepSeek、Groq等）都在兼容这套规范。

为了方便你理解和调用，我把这套规范拆解为**三层结构**和**五大核心参数**，并附上具体的代码示例。

## 相关页面
- [[agent/api/deepseek]] — DeepSeek API 调用示例
- [[langchain/models/模型]] — LangChain 的聊天模型抽象

### 一、请求的三层结构（HTTP层面）

一个标准的API请求包含以下三个部分：

1.  **Endpoint（端点地址）**：通常是 `https://api.xxx.com/v1/chat/completions`（注意统一用 `POST` 方法）。
2.  **Headers（请求头）**：
    - `Content-Type: application/json`（固定写法）。
    - `Authorization: Bearer <你的API_Key>`（身份认证，注意Bearer前缀和空格）。
3.  **Body（请求体）**：JSON格式，这是核心业务参数所在。

---

### 二、Body中的五大核心参数（必知必会）

这是规范中最关键的部分，绝大多数调整都是围绕这几个参数展开：

| 参数名               | 类型      | 是否必填   | 作用与使用技巧                                                                                        |
| :---------------- | :------ | :----- | :--------------------------------------------------------------------------------------------- |
| **`model`**       | string  | **必填** | 指定调用的模型版本（如 `gpt-4o`、`deepseek-v3`、`qwen-max`）。**注意**：不同厂商的模型名不同，需查阅官方文档。                      |
| **`messages`**    | array   | **必填** | 对话上下文列表。这是**最核心**的字段，包含角色（role）和内容（content）。角色通常有 `system`（系统设定）、`user`（用户）、`assistant`（AI回复）。 |
| **`temperature`** | number  | 选填     | 控制**随机性**（创造性）。范围 0~2，**默认 1.0**。值越低（如0.1）回答越确定、严谨；值越高（如1.5）回答越发散、有创意。                         |
| **`max_tokens`**  | integer | 选填     | 限制**输出长度**（不是字数，是Token数）。用于控制成本，防止AI长篇大论。**模型本次生成的所有输出Token总数**                                |
| **`stream`**      | boolean | 选填     | 是否**流式输出**。设为 `true` 时，AI会像打字机一样逐字返回（体验好）；`false` 则一次性返回完整结果（便于程序处理）。                          |

---

### 三、Messages数组的进阶玩法（Function Calling）

在现代API规范中，`messages` 里除了简单的对话，还有一个极重要的扩展——**`tool_calls`（函数调用）**。

当你想让AI帮你查天气、订机票或操作数据库时，需要在 `messages` 同级加上 `tools` 参数。流程是：

1. 用户提问 -> 2. AI返回 `tool_calls` 指令（不直接回答） -> 3. 你程序执行函数 -> 4. 将结果以 `tool` 角色回传给AI -> 5. AI输出最终自然语言。

---

### 四、标准的请求体（JSON）长什么样？

这是一个最精简、规范的请求示例（Python版）：

```python
import requests

url = "https://api.openai.com/v1/chat/completions"
headers = {
    "Content-Type": "application/json",
    "Authorization": "Bearer sk-你的密钥"
}
data = {
    "model": "gpt-4o-mini",
    "messages": [
        {"role": "system", "content": "你是一位精通Python的助手，回答要简洁。"},
        {"role": "user", "content": "如何用一行代码读取文件？"}
    ],
    "temperature": 0.7,
    "max_tokens": 500,
    "stream": False
}

response = requests.post(url, headers=headers, json=data)
print(response.json()['choices'][0]['message']['content'])
```

---

### 五、响应体（Response）的结构解读

成功调用后，返回的JSON结构如下，你需要精准提取其中的内容：

```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "created": 1700000000,
  "model": "gpt-4o-mini",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "使用 with open('file.txt') as f: print(f.read())"
      },
      "finish_reason": "stop"  // stop表示正常结束；length表示超出max_tokens截断
    }
  ],
  "usage": {
    "prompt_tokens": 20,    // 输入消耗
    "completion_tokens": 15, // 输出消耗
    "total_tokens": 35      // 计费依据
  }
}
```

---

### 六、必须注意的3个隐藏规范（避坑指南）

1.  **Token计算误差**：不同模型使用不同的Tokenizer（分词器）。你的 `max_tokens` 限制的是**输出**长度，但计费是按 `total_tokens`（输入+输出）。长上下文场景下，输入Token可能远超输出，请留意官方定价。
2.  **System角色的替代性**：如果模型不支持 `system` 角色（部分老旧模型），可以将系统指令写在第一条 `user` 消息里，并用分隔符（如`###指令###`）隔开。
3.  **重试与超时**：API规范建议设置 `timeout`（如30秒）。遇到 `429`（限流）或 `5xx`（服务器错误）时，建议采用**指数退避**策略（间隔 1s, 2s, 4s...）重试，不要无脑刷。

---

### 七、国内厂商的微小差异

虽然国内大厂兼容OpenAI规范，但有几个细微差别请留意：

- **模型名称**：不能填 `gpt-4`，要填对应的 `qwen-turbo`、`ERNIE-4.0` 等。
- **Base URL**：需要换成厂商专属域名（如阿里是 `dashscope.aliyuncs.com`）。
- **额外字段**：部分厂商会增加 `enable_search`（联网搜索）或 `thought`（思考链）字段，这些属于厂商独有扩展，不影响标准字段。
