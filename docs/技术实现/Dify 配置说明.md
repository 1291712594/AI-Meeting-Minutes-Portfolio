# Dify 配置说明

本文档基于产品设计，描述 AI 会议纪要智能体在 Dify 平台上的工作流配置与提示词设计，用于指导开发实现。实际运行时参数可能微调。

## 1. 工作流概览

Dify 工作流包含以下节点（按顺序）：

1. **输入节点**：接收用户上传的音频文件或粘贴的文本
2. **条件分支**：判断输入类型（音频 → 语音识别；文本 → 直接处理）
3. **语音识别节点**：调用阿里云百炼 API 将音频转为文本
4. **LangChain Agent 调用节点**：将文本发送给 LangChain Agent 进行处理
5. **输出节点**：将 Agent 返回的 JSON 渲染为 Markdown 格式的纪要

## 2. 节点配置详情

### 2.1 输入节点

- **类型**：文本/文件混合输入
- **配置**：支持 `text` 和 `file` 两种输入方式
    - 若用户上传音频文件，`type=audio`，文件格式限制 `mp3, wav, m4a`
    - 若用户粘贴文本，`type=text`，内容直接传入

### 2.2 条件分支

- **条件**：
    - `if input.type == "audio"` → 进入语音识别节点
    - `else` → 直接进入 LangChain Agent 节点

### 2.3 语音识别节点

- **调用 API**：阿里云百炼语音识别服务
- **参数设置**：
    - `format`：根据文件扩展名自动识别（mp3/wav/m4a）
    - `sample_rate`：16000 Hz（标准电话音质）
    - `enable_itn`：true（智能断句）
    - `enable_voice_detection`：true（自动过滤静音）
- **输出**：识别后的纯文本字符串

### 2.4 LangChain Agent 调用节点

- **调用方式**：通过 Dify 的“自定义代码节点”或“HTTP 请求节点”调用部署好的 LangChain Agent API
- **输入**：文本内容（来自语音识别或直接粘贴）
- **Agent 内部逻辑**：
    - 使用 LangChain 的 `create_agent` 创建多智能体
    - 工具集：
        - `SummarizerTool`：生成会议主题、讨论要点、决策事项
        - `ActionItemExtractor`：提取任务、责任人、截止时间
        - `FormatterTool`：将结果按固定 JSON 格式输出
    - 底层 LLM：DeepSeek（通过 API 调用）
- **输出**：JSON 格式的结构化纪要，例如：
    
    ```json
    {
      "title": "Q3 产品规划评审会",
      "discussion_points": [
        "确定了首页改版方向",
        "讨论了预算分配问题"
      ],
      "decisions": [
        "确认启动 X 项目",
        "预算批准 50 万"
      ],
      "action_items": [
        {
          "description": "完成原型设计",
          "owner": "张三",
          "deadline": "2025-10-20"
        },
        {
          "description": "安排下次会议",
          "owner": null,
          "deadline": null
        }
      ]
    }
    ```
    

### 2.5 输出节点

- **功能**：将 JSON 渲染为 Markdown 格式的纪要
- **模板**（示例）：
    
    markdown
    
    ```
    # AI 会议纪要
    
    ## 会议主题
    {{title}}
    
    ## 讨论要点
    {% for point in discussion_points %}
    - {{point}}
    {% endfor %}
    
    ## 决策事项
    {% for decision in decisions %}
    - {{decision}}
    {% endfor %}
    
    ## 行动计划
    {% for item in action_items %}
    - [ ] **{{item.description}}**
      责任人：{{item.owner or '【待补充】'}}
      截止时间：{{item.deadline or '【待补充】'}}
    {% endfor %}
    ```
    

## **3. 提示词设计**

### **3.1 摘要专家（SummarizerTool）**

**系统提示词**：

text

```
你是一位专业的会议记录员。请根据提供的会议文本，提取以下信息：
1. 会议主题（一句话概括）
2. 讨论要点（分点列出，每条不超过20字）
3. 决策事项（明确会议达成的决定，分点列出）

要求：
- 只基于文本内容，不添加任何虚构信息
- 如果某部分信息缺失，输出空列表
- 输出格式为 JSON
```

**用户提示词**：直接传入会议文本。

### **3.2 任务提取专家（ActionItemExtractor）**

**系统提示词**：

text

```
你是一位项目经理。请从会议文本中提取所有需要执行的任务（行动项）。
每条任务必须包含：
- 任务描述（具体做什么）
- 责任人（谁负责，如果无法确定则设为 null）
- 截止时间（具体日期或相对时间，如“下周五”，如果无法确定则设为 null）

输出格式为 JSON 数组，每个元素包含 description, owner, deadline。
如果文本中没有明确任务，输出空数组。
```

**特殊处理**：

- 当 owner 或 deadline 为 null 时，在最终输出中标记【待补充】。
- 如果文本中包含模糊承诺（如“尽量”“可能”），不强行提取。

### **3.3 格式化专家（FormatterTool）**

**系统提示词**：

text

```
你是一个格式整理器。将前面提取的摘要和任务合并为以下 JSON 结构：
{
  "title": "会议主题",
  "discussion_points": ["要点1", "要点2"],
  "decisions": ["决策1", "决策2"],
  "action_items": [
    {"description": "任务描述", "owner": "责任人", "deadline": "截止时间"}
  ]
}
```

## **4. 模型参数**

| **参数** | **值** | **说明** |
| --- | --- | --- |
| model | deepseek-chat | 选用 DeepSeek 大模型，兼顾效果与成本 |
| temperature | 0.3 | 较低随机性，保证纪要输出稳定、准确 |
| max_tokens | 2048 | 足够覆盖大部分会议纪要长度 |
| top_p | 0.9 | 保持一定多样性，但不影响关键信息提取 |

## **5. 语音识别 API 配置**

- **服务商**：阿里云百炼
- **接口**：[https://dashscope.aliyuncs.com/api/v1/services/audio/asr/transcription](https://dashscope.aliyuncs.com/api/v1/services/audio/asr/transcription)
- **认证**：Bearer Token（用户自行申请）
- **请求参数**：
    - `file_url`：音频文件公网可访问地址（或 base64 编码）
    - `format`：音频格式
    - `sample_rate`：16000
- **响应**：返回转写文本及置信度

## **6. 异常处理**

- **语音识别失败**：提示用户“音频质量不佳，建议重新上传或粘贴文本”。
- **LangChain Agent 超时**：设置超时时间 30 秒，超时则返回部分结果并提示用户稍后重试。
- **关键信息缺失**：在输出中明确标注【待补充】，不中断流程。

## **7. 后续优化方向**

- 增加自定义词库，提升专业术语识别率
- 引入流式处理，降低用户等待感知
- 优化提示词，提高责任人识别准确率

---