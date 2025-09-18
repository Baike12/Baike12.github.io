---
title: "eino-adk抽象"
date: 2025-09-17T16:36:29+08:00
draft: false
categories: ["eino"]
tags: ["eino"]
author: "Baike"
description: "eino梳理"
---



# 1	agent定义
一个接口
```go
// github.com/cloudwego/eino/adk/interface.go

type Agent interface {
    Name(ctx context.Context) string
    Description(ctx context.Context) string
    Run(ctx context.Context, input *AgentInput, opts ...AgentRunOption) *AsyncIterator[*AgentEvent]
}
```
- 名称
- 描述：让其他agent了解该agent的职责或功能
- run：返回一个迭代器，调用方通过这个迭代器接收这个agent产生的事件
# 2	agentinput
```go
type AgentInput struct {
    Messages        []_Message_
_    _EnableStreaming bool
}
```
- Message列表：可以是用户指令、对话历史、样例数据等任何需要传递给agent的数据
- \_EnableStreaming只是**建议**agent的输出形式，对同时支持流式和非流式的组件有效，如果组件只支持一种输出形式，就不生效
Message列表比如：
```go
import (
    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/schema"
)

input := &adk.AgentInput{
    Messages: []adk.Message{
       schema.UserMessage("What's the capital of France?"),
       schema.AssistantMessage("The capital of France is Paris.", nil),
       schema.UserMessage("How far is it from London? "),
    },
}
```
- Message列表包含对话历史
# 3	agentrunoption
在请求维度修改agent配置或者控制agent行为，提供了包装自定义agentrunoption的方法：
- WrapImplSpecificOptFn：定义
- GetImplSpecificOptFn：获取
一个在请求时修改模型的例子：
```go
// github.com/cloudwego/eino/adk/call_option.go
// func WrapImplSpecificOptFn[T any](optFn func(*T)) AgentRunOption
// func GetImplSpecificOptions[T any](base *T, opts ...AgentRunOption) *T

import "github.com/cloudwego/eino/adk"

type options struct {
    modelName string
}

func WithModelName(name string) adk.AgentRunOption {
    return adk.WrapImplSpecificOptFn(func(t *options) {
       t.modelName = name
    })
}

func (m *MyAgent) Run(ctx context.Context, input *adk.AgentInput, opts ...adk.AgentRunOption) *adk.AsyncIterator[*adk.AgentEvent] {
    o := &options{}
    o = adk.GetImplSpecificOptions(o, opts...)
    // run code...
}
```
Agentrunoption有一个designateAgent方法用于在多agent中指定option生效的agent
# 4	asyncIterator
agent.Run返回一个异步迭代器AsyncIterator\[\*AgentEvent\]
```go
// github.com/cloudwego/eino/adk/utils.go

type AsyncIterator[T any] struct {
    ...
}

func (ai *AsyncIterator[T]) Next() (T, bool) {
    ...
}
```
调用者可以以有序、阻塞的方式消费Event
- AsyncIterator是一个泛型结构体，在Agent接口中，返回的是一个AsyncIterator\[\*AgentEvent]，迭代器中获取的是一个指向AgentEvent对象的指针
- 调用方通过Next方法阻塞的获取AgentEvent，每次调用Next会暂停，直到：
	- Agent产生一个新的AgentEvent
	- Agent关闭了迭代器，这时Next返回的第二个参数是false
AsyncIterator通常在for中处理：
```go
iter := myAgent.Run(xxx) // get AsyncIterator from Agent.Run

for {
    event, ok := iter.Next()
    if !ok {
        break
    }
    // handle event
}
```
AsyncIterator可以由NewAsyncIteratorPair创建，返回迭代器和生成器，生成器用来生成数据
```go
// github.com/cloudwego/eino/adk/utils.go

func NewAsyncIteratorPair[T any]() (*AsyncIterator[T], *AsyncGenerator[T])
```
agenr.run返回AsyncIterator，run通常在一个Goroutine运行agent从而立即返回AsyncIterator让调用者监听
```go
import "github.com/cloudwego/eino/adk"

func (m *MyAgent) Run(ctx context.Context, input *adk.AgentInput, opts ...adk.AgentRunOption) *adk.AsyncIterator[*adk.AgentEvent] {
    // handle input
    iter, gen := adk.NewAsyncIteratorPair[*adk.AgentEvent]()
    go func() {
       defer func() {
          // recover code
          gen.Close()
       }()
       // agent run code
       // gen.Send(event)
    }()
    return iter
}
```
# 5	agentwithoptions
使用agentwithoptions可以在agent做一些通用配置
```go
// github.com/cloudwego/eino/adk/flow.go
func AgentWithOptions(ctx context.Context, agent Agent, opts ...AgentOption) Agent
```
比如withhistoryrewriter
# 6	AgentEvent
agent运行过程中产生的事件
```go
// github.com/cloudwego/eino/adk/interface.go

type AgentEvent struct {
    AgentName string

    RunPath []string

    Output *AgentOutput

    Action *AgentAction

    Err error
}
```
- 名字、路径、输出、行为和报错
## 6.1	AgentName
指明哪一个agent产生当前事件
## 6.2	Runpath
到达当前agent的完整链路
## 6.3	output
```go
// github.com/cloudwego/eino/adk/interface.go

type AgentOutput struct {
    MessageOutput *MessageVariant

    CustomizedOutput any
}
```
- agent产生的输出，Message输出在MessageOutput中，其他类型输出在CustomizedOutput中
MessageVariant可以统一处理流式和非流式
```go
type MessageVariant struct {
    IsStreaming bool

    Message       Message
    MessageStream MessageStream
    // message role: Assistant or Tool
    Role schema.RoleType
    // only used when Role is Tool
    ToolName string
}
```
- 实际上Message本身就包含一些元数据比如Role，这里MessageVariant将这些常用元数据提到顶层，如果需要根据消息类型执行动作时不用深入Message队形
## 6.4	AgentAction
如果agent产生agentaction，可以用来控制多agent协作，比如立即退出、中断、跳转
```go
// github.com/cloudwego/eino/adk/interface.go

type AgentAction struct {
    Exit bool

    Interrupted *_InterruptInfo_

_    _TransferToAgent *_TransferToAgentAction_

_    _CustomizedAction any
}
```
- Exit会让multiagent立即退出
- Transfer会跳转到目标agent运行
# 7	多agent协作
比如Transfer或者预先决定agent的运行顺序
## 7.1	上下文传递
### 7.1.1	History
每一个agent产生的事件都会保存到History中，调用新的agent时将所有History拼接到agentinput中
默认情况下，其他agent的Assistant和tool Message转换成User Message，将其他agent的行为当做当前agent的外部信息或事实陈述
>[!note]
>当前agent只会处理来自直接或者间接父agent的agent的event
### 7.1.2	withhistoryrewriter
可以通过withhistoryrewriter自定义agent从History中生成agentinput的方式
```go
// github.com/cloudwego/eino/adk/flow.go

type HistoryRewriter func(ctx context.Context, entries []*HistoryEntry) ([]Message, error)

func WithHistoryRewriter(h HistoryRewriter) AgentOption
```
- 比如拼接所有父agent的event
### 7.1.3	sessionvalues
一次运行中持续存在的全局临时kv存储，agent可以在任意时间读写sessionvalue
```go
// github.com/cloudwego/eino/adk/runctx.go
// 获取全部 SessionValues
func GetSessionValues(ctx context.Context) map[string]any
// 设置 SessionValues
func SetSessionValue(ctx context.Context, key string, value any)
// 指定 key 获取 SessionValues 中的一个值，key 不存在时第二个返回值为 false，否则为 true
func GetSessionValue(ctx context.Context, key string) (any, bool)
```
- 读写和判断一个key是否存在
# 8	Transfer subagents
agent产生Transferaction的AgentEvent后，adk会调用action指定的agent，创建TransferAction可以使用NewTransferToAgentAction
```go
import "github.com/cloudwego/eino/adk"

event := adk.NewTransferToAgentAction("dest agent name")
```
为了让adk找到要移交的agent，需要先注册
```go
// github.com/cloudwego/eino/adk/flow.go
func SetSubAgents(ctx context.Context, agent Agent, subAgents []Agent) (Agent, error)
```
这样注册之后agent天然知道自己可以移交给哪些agent
有一些内置的agent框架比如chatmodelagent，需要动态的注册父子agent，可以通过OnSubAgents实现：
```go
// github.com/cloudwego/eino/adk/interface.go
type OnSubAgents interface {
    OnSetSubAgents(ctx context.Context, subAgents []Agent) error
    OnSetAsSubAgent(ctx context.Context, parent Agent) error

    OnDisallowTransferToParent(ctx context.Context) error
}
```
## 8.1	一个任务移交的例子
```go
package mutiagent

import (
	"context"
	"fmt"
	"log"

	"github.com/cloudwego/eino/adk"
	"github.com/cloudwego/eino/components/tool"
	"github.com/cloudwego/eino/components/tool/utils"
	"github.com/cloudwego/eino/compose"
)

type GetWeatherInput struct {
	City string `json:"city"`
}

func NewWeatherAgent() adk.Agent {
	weatherTool, err := utils.InferTool(
		"get_weather",
		"Gets the current weather for a specific city.", // English description
		func(ctx context.Context, input *GetWeatherInput) (string, error) {
			return fmt.Sprintf(`the temperature in %s is 25°C`, input.City), nil
		},
	)
	if err != nil {
		log.Fatal(err)
	}

	a, err := adk.NewChatModelAgent(context.Background(), &adk.ChatModelAgentConfig{
		Name:        "WeatherAgent",
		Description: "This agent can get the current weather for a given city.",
		Instruction: "Your sole purpose is to get the current weather for a given city by using the 'get_weather' tool. After calling the tool, report the result directly to the user,用中文回答",
		Model:       newChatModel(),
		ToolsConfig: adk.ToolsConfig{
			ToolsNodeConfig: compose.ToolsNodeConfig{
				Tools: []tool.BaseTool{weatherTool},
			},
		},
	})
	if err != nil {
		log.Fatal(err)
	}
	return a
}

func NewChatAgent() adk.Agent {
	a, err := adk.NewChatModelAgent(context.Background(), &adk.ChatModelAgentConfig{
		Name:        "ChatAgent",
		Description: "A general-purpose agent for handling conversational chat.", // English description
		Instruction: "You are a friendly conversational assistant. Your role is to handle general chit-chat and answer questions that are not related to any specific tool-based tasks,用中文回答",
		Model:       newChatModel(),
	})
	if err != nil {
		log.Fatal(err)
	}
	return a
}

func NewRouterAgent() adk.Agent {
	a, err := adk.NewChatModelAgent(context.Background(), &adk.ChatModelAgentConfig{
		Name:        "RouterAgent",
		Description: "A manual router that transfers tasks to other expert agents.",
		Instruction: `You are an intelligent task router. Your responsibility is to analyze the user's request and delegate it to the most appropriate expert agent.If no Agent can handle the task, simply inform the user it cannot be processed.`,
		Model:       newChatModel(),
	})
	if err != nil {
		log.Fatal(err)
	}
	return a
}

func RouteAgent() {
	weatherAgent := NewWeatherAgent()
	chatAgent := NewChatAgent()
	routerAgent := NewRouterAgent()

	ctx := context.Background()
	a, err := adk.SetSubAgents(ctx, routerAgent, []adk.Agent{chatAgent, weatherAgent})
	if err != nil {
		log.Fatal(err)
	}

	runner := adk.NewRunner(ctx, adk.RunnerConfig{
		Agent: a,
	})

	// query weather
	println("\n\n>>>>>>>>>query weather<<<<<<<<<")
	iter := runner.Query(ctx, "What's the weather in Beijing?")
	for {
		event, ok := iter.Next()
		if !ok {
			break
		}
		if event.Err != nil {
			log.Fatal(event.Err)
		}
		if event.Action != nil {
			fmt.Printf("\nAgent[%s]: transfer to %+v\n\n======\n", event.AgentName, event.Action.TransferToAgent.DestAgentName)
		} else {
			fmt.Printf("\nAgent[%s]:\n%+v\n\n======\n", event.AgentName, event.Output.MessageOutput.Message)
		}
	}

	// failed to route
	println("\n\n>>>>>>>>>failed to route<<<<<<<<<")
	iter = runner.Query(ctx, "Book me a flight from New York to London tomorrow.")
	for {
		event, ok := iter.Next()
		if !ok {
			break
		}
		if event.Err != nil {
			log.Fatal(event.Err)
		}
		if event.Action != nil {
			fmt.Printf("\nAgent[%s]: transfer to %+v\n\n======\n", event.AgentName, event.Action.TransferToAgent.DestAgentName)
		} else {
			fmt.Printf("\nAgent[%s]:\n%+v\n\n======\n", event.AgentName, event.Output.MessageOutput.Message)
		}
	}
}

```
