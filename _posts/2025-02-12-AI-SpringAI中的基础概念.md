---
layout:     post
title:      SpringAI 开启 Java AI 新纪元

subtitle:   SpringAI 
date:       2025-02-12
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringAI
---

作为Java开发者，从SpringAI开始，从0开始认识AI：

- [模型（Model）](#模型model)
- [提示（Prompt）](#提示prompt)
- [提示词模板（Prompt Template）](#提示词模板prompt-template)
- [嵌入（Embedding）](#嵌入embedding)
- [Token](#token)
- [结构化输出（Structured Output）](#结构化输出structured-output)
- [将您的数据和 API 引入 AI 模型](#将您的数据和-api-引入-ai-模型)
- [评估人工智能的回答（Evaluation）](#评估人工智能的回答evaluation)
- [ChatClient](#chatclient)
- [ChatModel](#chatmodel)
- [Image Model](#image-model)
- [Audio Model](#audio-model)
- [Embedding Model 嵌入模型](#embedding-model-嵌入模型)
- [模型上下文协议（Model Context Protocol）](#模型上下文协议model-context-protocol)
- [对话记忆](#对话记忆)
- [提示词 (Prompt)](#提示词-prompt)
- [文档检索 (Document Retriever)](#文档检索-document-retriever)
- [向量存储(Vector Store)](#向量存储vector-store)
- [RAG](#rag)
- [智能机票助手](#智能机票助手)

### 模型（Model）

AI 模型是旨在处理和生成信息的算法，通常模仿人类的认知功能

### 提示（Prompt）

Prompt作为语言基础输入的基础，指导AI模型生成特定的输出
这种互动风格的重要性使得“Prompt工程”这一学科应运而生。现在有越来越多的技术被提出，以提高Prompt的有效性。投入时间去精心设计Prompt可以显著改善生成的输出。

### 提示词模板（Prompt Template）

创建有效的Prompt涉及建立请求的上下文，并用用户输入的特定值替换请求的部分内容

### 嵌入（Embedding）

嵌入（Embedding）是文本、图像或视频的数值表示，能够捕捉输入之间的关系，Embedding通过将文本、图像和视频转换为称为向量（Vector）的浮点数数组来工作
Embedding在实际应用中，特别是在检索增强生成（RAG）模式中，具有重要意义。它们使数据能够在语义空间中表示为点，这类似于欧几里得几何的二维空间，但在更高的维度中。这意味着，就像欧几里得几何中平面上的点可以根据其坐标的远近关系而接近或远离一样，在语义空间中，点的接近程度反映了意义的相似性。关于相似主题的句子在这个多维空间中的位置较近，就像图表上彼此靠近的点。这种接近性有助于文本分类、语义搜索，甚至产品推荐等任务，因为它允许人工智能根据这些点在扩展的语义空间中的“位置”来辨别和分组相关概念。

### Token

token是 AI 模型工作原理的基石。输入时，模型将单词转换为token。输出时，它们将token转换回单词。

### 结构化输出（Structured Output）

即使您要求回复为 JSON ，AI 模型的输出通常也会以 java.lang.String 的形式出现。

```bash
entity(ActorsFilms.class)

entity(new ParameterizedTypeReference<List<ActorsFilms>>() {})

entity(new ParameterizedTypeReference<Map<String, Object>>() {})

entity(new ListOutputConverter(new DefaultConversionService()))

BeanOutputConverter<ActorsFilms> beanOutputConverter =
    new BeanOutputConverter<>(ActorsFilms.class);
ActorsFilms actorsFilms = beanOutputConverter.convert(generation.getOutput().getContent());
```

### 将您的数据和 API 引入 AI 模型

> 请注意，GPT 3.5/4.0 数据集仅支持截止到 2021 年 9 月之前的数据。因此，该模型表示它不知道该日期之后的知识，因此它无法很好的应对需要用最新知识才能回答的问题。一个有趣的小知识是，这个数据集大约有 650GB。

有三种技术可以定制 AI 模型以整合您的数据:

- Fine Tuning 微调

    这种传统的机器学习技术涉及定制模型并更改其内部权重。即使对于机器学习专家来说，这是一个具有挑战性的过程，而且由于 GPT 等模型的大小，它极其耗费资源

- Prompt Stuffing 提示词填充

    一种更实用的替代方案是将您的数据嵌入到提供给模型的提示中。Spring AI 库可帮助您基于“提示词填充” 技术，也称为检索增强生成 (RAG)实现解决方案。

- Function Calling 函数调用

    大型语言模型 (LLM) 在训练后即被冻结，导致知识陈旧，并且无法访问或修改外部数据。

    此技术允许注册自定义的用户函数，将大型语言模型连接到外部系统的 API。Spring AI 大大简化了支持函数调用所需编写的代码。

  - 定义&注册函数
  - 为 Prompt 指定函数
  - 动态注册函数

### 评估人工智能的回答（Evaluation）

有效评估人工智能系统回答的正确性，对于确保最终应用程序的准确性和实用性非常重要，一些新兴技术使得预训练模型本身能够用于此目的。

### ChatClient

ChatClient 提供了与 AI 模型通信的 Fluent API，它支持同步和反应式（Reactive）编程模型。与 ChatModel、Message、ChatMemory 等原子 API 相比，使用 ChatClient 可以将与 LLM 及其他组件交互的复杂性隐藏在背后，因为基于 LLM 的应用程序通常要多个组件协同工作（例如，提示词模板、聊天记忆、LLM Model、输出解析器、RAG 组件：嵌入模型和存储），并且通常涉及多个交互，因此协调它们会让编码变得繁琐。当然使用 ChatModel 等原子 API 可以为应用程序带来更多的灵活性，成本就是您需要编写大量样板代码

- 创建 ChatClient
- 处理 ChatClient 响应
- call() 返回值
- stream() 返回值
- 定制 ChatClient 默认值
- Advisors
    在使用用户输入文本构建 Prompt 调用 AI 模型时，一个常见模式是使用上下文数据附加或扩充 Prompt，最终使用扩充后的 Prompt 与模型交互。

    常见类型包括：
  - 您自己的数据：这是 AI 模型尚未训练过的数据，如特定领域知识、产品文档等，即使模型已经看到过类似的数据，附加的上下文数据也会优先生成响应。
  - 对话历史记录：聊天模型的 API 是无状态的，如果您告诉 AI 模型您的姓名，它不会在后续交互中记住它，每次请求都必须发送对话历史记录，以确保在生成响应时考虑到先前的交互
  - 检索增强生成（RAG）:向量数据库存储的是 AI 模型不知道的数据，当用户问题被发送到 AI 模型时，QuestionAnswerAdvisor 会在向量数据库中查询与用户问题相关的文档
    - 动态过滤表达式:SearchRequest 使用 FILTER_EXPRESSION Advisor 上下文参数在运行时更新过滤表达式
    - 聊天记忆: ChatMemory 接口表示聊天对话历史记录的存储，它提供向对话添加消息、从对话中检索消息以及清除对话历史记录的方法。目前提供两种实现方式 InMemoryChatMemory、CassandraChatMemory，分别为聊天对话历史记录提供内存存储和 time-to-live 类型的持久存储。
      - 日志：SimpleLoggerAdvisor `org.springframework.ai.chat.client.advisor=DEBUG`

### ChatModel

ChatModel API 让应用开发者可以非常方便的与 AI 模型进行文本交互，它抽象了应用与模型交互的过程，包括使用 Prompt 作为输入，使用 ChatResponse 作为输出等。ChatModel 的工作原理是接收 Prompt 或部分对话作为输入，将输入发送给后端大模型，模型根据其训练数据和对自然语言的理解生成对话响应，应用程序可以将响应呈现给用户或用于进一步处理。

### Image Model

mageModel API 抽象了应用程序通过模型调用实现“文生图”的交互过程，即应用程序接收文本，调用模型生成图片。ImageModel 的入参为包装类型 ImagePrompt，输出类型为 ImageResponse。

### Audio Model

1. 文本生成语音 SpeechModel，对应于 OpenAI 的 Text-To-Speech (TTS) API
2. 录音文件生成文字 DashScopeAudioTranscriptionModel，对应于 OpenAI 的 Transcription API

### Embedding Model 嵌入模型

嵌入(Embedding)的工作原理是将文本、图像和视频转换为称为向量（Vectors）的浮点数数组。这些向量旨在捕捉文本、图像和视频的含义。嵌入数组的长度称为向量的维度（Dimensionality）。

嵌入模型（EmbeddingModel）是嵌入过程中采用的模型

### 模型上下文协议（Model Context Protocol）

模型上下文协议（即 Model Context Protocol，MCP）是一个开放协议，它规范了应用程序如何向大型语言模型（LLM）提供上下文

### 对话记忆

自己维护多轮的对话记忆

基于memory的对话记忆

自行实现ChatMemory基于类似于文件、Redis等方式进行上下文内容的存储和记录

### 提示词 (Prompt)

Prompt 中的主要角色（Role）包括：

- 系统角色（System Role）：指导 AI 的行为和响应方式，设置 AI 如何解释和回复输入的参数或规则。这类似于在发起对话之前向 AI 提供说明。
- 用户角色（User Role）：代表用户的输入 - 他们向 AI 提出的问题、命令或陈述。这个角色至关重要，因为它构成了 AI 响应的基础。
- 助手角色（Assistant Role）：AI 对用户输入的响应。这不仅仅是一个答案或反应，它对于保持对话的流畅性至关重要。通过跟踪 AI 之前的响应（其“助手角色”消息），系统可确保连贯且上下文相关的交互。助手消息也可能包含功能工具调用请求信息。它就像 AI 中的一个特殊功能，在需要执行特定功能（例如计算、获取数据或不仅仅是说话）时使用。
- 工具/功能角色（Tool/Function Role）：工具/功能角色专注于响应工具调用助手消息返回附加信息。

### 文档检索 (Document Retriever)

文档检索（DocumentRetriever）是一种信息检索技术，旨在从大量未结构化或半结构化文档中快速找到与特定查询相关的文档或信息。文档检索通常以在线(online)方式运行。

### 向量存储(Vector Store)

向量存储（VectorStore）是一种用于存储和检索高维向量数据的数据库或存储解决方案，它特别适用于处理那些经过嵌入模型转化后的数据。

### RAG

检索增强生成 (RAG) 是一种使用来自私有或专有数据源的信息来辅助文本生成的技术。

- Indexing pipeline阶段

将结构化或者非结构化的数据或文档进行加载和解析、chunk切分、文本向量化并保存到向量数据库

- RAG的阶段

将prompt文本内容转为向量、从向量数据库检索内容、对检索后的文档chunk进行重排和prompt重写、最后调用大模型进行结果的生成。

### 智能机票助手

基于基础订票规则的智能订票、退票、改签小助手，它比你更懂票务：

[直接上代码](https://github.com/springaialibaba/spring-ai-alibaba-examples/blob/main/spring-ai-alibaba-agent-example/playground-flight-booking/src/main/java/ai/spring/demo/ai/playground/services/CustomerSupportAssistant.java)
