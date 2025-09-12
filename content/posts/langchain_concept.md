---
title: "langchain核心概念"
date: 2025-09-12T21:36:29+08:00
draft: false
categories: ["langchain"]
tags: ["langchain"]
author: "Baike"
description: "langchain框架基本概念"
---

# 1	架构
## 1.1	langchain-core
核心组件：
- 向量数据库
- 检索
- 索引
不包括第三方库
## 1.2	langchain
包含chain、agent和检索策略
## 1.3	langchain-community
第三方库
## 1.4	partner packages
大公司的组件库，比如openai、anthropic
## 1.5	langgraph
用来构建基于边和节点组成的图的有状态的、稳定的agent
提供了一些通用的agent模板，比如react agent
# 2	组件
## 2.1	chat models
输入消息输出消息，支持多种角色
一般会将输入的string当做HumanMessage
### 2.1.1	参数
模型名、温度、超时时间、最大tokens，最大轮数、最大重试数、key、地址
>[!note]
>并不是所有模型都支持所有参数
>参数只在模型厂商自己提供的依赖包生效，第三方的不一定有效
>有的三方包还提供其他参数
### 2.1.2	多模态
不是所有的模型都支持多模态输入输出
## 2.2	Message
每个消息都有角色、内容和响应元数据属性
### 2.2.1	内容
- string
- 字典列表：用于多模态消息
有些模型支持多用户角色，用name属性区分
### 2.2.2	响应元数据
记录token出现的对数概率和token消耗量
### 2.2.3	tool_calls
llm会输出，在.tool_calls属性中获取，是一个字典，包含：
- name：工具名
- args：参数
- id：工具id
### 2.2.4	SystemMessage
llm的预设角色，告诉大模型以什么角色工作
### 2.2.5	ToolMessage
调用工具返回的结果：
- tool_call_id：指明是哪个tool产生的结果
- artifact：一些额外信息，比如如果调用查询数据库工具，artifacts保存原始的sql语句
## 2.3	Prompt templates
减少重复写Prompt的工作，会转换成可以直接传递给llm的PromptValue
### 2.3.1	String PromptTemplates
接受一个string，用于简单的输入
### 2.3.2	ChatPromptTemlates
用来格式化消息列表，可以传递多条消息，使用from_messages可以传递角色和消息作为参数构建消息列表
#### 2.3.2.1	MessagesPlaceholder
消息占位符，用来先占住位置，然后在调用的时候用变量填充占位符的内容
```js
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder chat_prompt_template = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    MessagesPlaceholder(variable_name="messages"),
])

chat_prompt = chat_prompt_template.invoke({
    "messages": [
        ("user", "What is the weather in San Francisco?"),
    ]
})

print(chat_prompt)
```
## 2.4	示例选择器
可以给llm一些例子让他参考例子完成任务
## 2.5	输出解析器
将llm的输出格式化为标准格式供下游使用
解析器特性：
- 名称
- 是否支持流式
- 是否调用llm，指解析器自身
- 输入输出类型
- 输出描述
langchain自带了一些解析器，比如json、xml、pydantic、时间格式等
## 2.6	聊天记录
将历史对话记录添加到当前对话中
## 2.7	文档
- page_content：一个str，就是文档内容
- metadata：一个字典，存储文档属性
## 2.8	文档加载器
所有外部文档都可以加载，流程：
- 定义一个加载器
- 使用.load方法从加载器中加载
## 2.9	文本拆分器
可以对文档执行拆分、组合、过滤等操作
拆分文档并保持语义关联
### 2.9.1	流程
- 将文档拆分成chunks，通常是一个句子
- 将句子组合成一个大chunk，这个大chunk是有一定长度限制的
- 开始下一个大chunk，不过下一个大chunk和之前的大chunk有一定重叠，这能保持语义上的关联
**问题：**
- **如何拆分文本**
- **chunk大小如何定**
## 2.10	嵌入
将文本转换成向量
## 2.11	向量数据库
存储向量并检索
## 2.12	检索器
接受一个string类型查询返回文档列表
## 2.13	kv存储
提供BaseStore接口，可以完成序列化和反序列化，接口方法：
- mget：获取多个key的值
- mset
- mdelete
- yield_keys：查看所有key
## 2.14	工具
定义工具：
- 名称
- 描述：工具能做什么
- 输入：json scheme
- 一个函数
用法：
- 定义tools
- bind_tools绑定到llm
- 使用绑定tools的llm完成任务
调用模型有两种方式：
### 2.14.1	使用工具参数调用
手动提取模型生成的ToolCall中的参数传给工具的invoke方法，获取输出之后手动封装成ToolMessage
- 提取ToolCall
- 获取参数，手动invoke
- 构造ToolMessage
### 2.14.2	用完整ToolCall调用
将模型生成的ToolCall传递给invoke方法，依赖模型自动处理参数解析和执行，并返回封装好的ToolMessage
### 2.14.3	最佳实践
- 选择针对工具调用微调过的模型
- 详细描述工具的作用和需要的参数
- 定义简单的工具
## 2.15	工具集
特定领域的工具集合
## 2.16	agent
agent把llm作为推理引擎来决定该调用什么工具，然后将工具调用结果返回给llm驱动下一轮工具调用或者结束
### 2.16.1	react agent
reason and act
流程：
- llm根据输入和当前观测推理下一步该做什么
- llm选择一个工具
- llm生成工具参数
- agent执行工具
- 获取工具执行结果作为观测
- 重复上面的步骤直到agent选择回复
## 2.17	callback
langchain提供了callback机制可以在各个阶段埋下钩子，比如：
- llm开始、结束、出错
- chain开始、结束、出错
- tool开始、结束、出错
- agent开始、结束、出错
- 重试时
### 2.17.1	callback 处理器
可以是同步和异步的
langchain的运行时会启动一个回调管理器来在各种事件发生时调用回到函数
### 2.17.2	传递回调
大部分对象都可以通过参数的形式传递回调，主要有两种：
- 请求时回调：对所有标准Runnable对象都适用，比如invoke，回调会传播给子组件
- 构造时回调：在调用构造函数时传递，只作用在组件自身
# 3	技术点
## 3.1	流式
langchain将流式和回调结合，使得可以实现异步stream，在获取chunk的同时获取事件类型，根据事件类型做出反应
### 3.1.1	流式中的回调
可以通过传递回调在各个阶段做出其他行为
### 3.1.2	function calling
流程：
- 从llm的回复中获取tool和参数
- 调用tool
- 将结果返回给llm生成最终响应
## 3.2	结构化输出
langchain提供了基于pydantic的结构化输出包，用法：
- 从langchain_core.pydantic_v1导入BaseModel、Field
- 继承BaseModel定义自己的响应类
- 使用Field定义字段，Field可以提供描述给llm用于生成结果
- 使用with_structed_output调用
```js
from typing import Optional

from langchain_core.pydantic_v1 import BaseModel, Field


class Joke(BaseModel):
    """Joke to tell user."""

    setup: str = Field(description="The setup of the joke")
    punchline: str = Field(description="The punchline to the joke")
    rating: Optional[int] = Field(description="How funny the joke is, from 1 to 10")

structured_llm = llm.with_structured_output(Joke)

structured_llm.invoke("Tell me a joke about cats")
```
如果不能使用pydantic结构化输出，可以选择以下方法
### 3.2.1	原始文本
自己解析
### 3.2.2	json 模式
有的模型会提供”只输出json“模式
## 3.3	few-shot prompt
给llm一些例子
### 3.3.1	如何生成例子
- 手工生成
- 用更好的模型的生成作为次一些模型的例子
- 用户反馈
- llm的反馈
### 3.3.2	例子数量如何选择
更好的模型需要更少的例子
多做一些测试
### 3.3.3	如何选择例子
可以生成很多例子，根据不同任务的语义来选择对应的例子
langchain提供了ExampleSelectors来选择例子
### 3.3.4	如何格式化例子
例子要指明输入和输出，通常以角色表明
## 3.4	检索
![[Pasted image 20250912152404.png]]
### 3.4.1	查询转换
#### 3.4.1.1	重写
在用户输入进入rag之前先重写用户输入，一些方法：
- 改写为多角度输入，然后分别对每个角度输入进行检索后回答
- 拆解问题：将大问题拆解为可以顺序解决的小问题
- 抽象问题：将用户的问题抽象为一个更高层级的问题，检索之后回答具体的问题
- 假设文档生成：原始问题难以检索到相关文档，使用llm将原始问题转换为假设的中间文档，用这个文档来检索
#### 3.4.1.2	路由
系统存在多个数据源时，将用户的查询路由到最合适的数据源：
- 逻辑路由：可根据规则判断数据源匹配情况时，基于预设的规则推理查询类型
- 语义路由：可通过语义判断数据源匹配情况时，将查询直接嵌入之后通过语义查询
#### 3.4.1.3	查询构建
将自然语言的查询转换成结构化语言：
- 转sql
- 转cypher
- 自查询：用于需要结合文档元数据检索时，将查询转换为两部分，语义检索字符串和元数据过滤，比如要查询某个时间点之前的某个事件
### 3.4.2	索引
有两种方法：
- 多向量检索：建立索引阶段对一个文档用llm生成子内容，比如摘要、假设问题等，为每个子内容embedding
- 父文档检索：先将长文档拆分为多个短片段，对每一个短片段做索引，检索时通过短片段找到长文档，将长文档作为上下文
- 事件加权向量存储：在向量检索的基础上加上时间戳权重
	- 检索逻辑：语义相似度×时间衰减系数
	- 适用于新闻、行业动态
### 3.4.3	检索
- 上下文感知的细粒度索引：为文档和查询的每个token生成上下文向量，然后通过查询token与文档token的逐点相似度最大化来检索
	- 适用于需要高精度匹配的场景
- 分层索引：将文档按层级索引，上层用于快速筛选候选文档，下层用于精确定位
- mmr：平衡相关性和多样性
	- 适用于需要信息全面性的场景比如创意写作、复杂问题回答
	- langchain实现了MMR
### 3.4.4	后处理
去掉不相关或者多余的内容
#### 3.4.4.1	上下文压缩
利用llm来压缩检索出来的上下文
#### 3.4.4.2	结果重排序
langchain提供了RerankingRetriever，支持Cohere Rerank
#### 3.4.4.3	文档整合
将检索到的多个片段整合成一个连贯的文档
### 3.4.5	反思
比如cursor能生成代码然后检查代码的问题
#### 3.4.5.1	self-rag
修复生成阶段的幻觉和回答不完整
#### 3.4.5.2	corrective-rag
修复检索阶段文档无关问题
## 3.5	文档拆分
langchain提供了一系列拆分器：
- text
- markdown
- code
## 3.6	评估
langsmith广告
