---
title: "eino-adk概述"
date: 2025-09-17T16:36:29+08:00
draft: false
categories: ["eino"]
tags: ["eino"]
author: "Baike"
description: "eino梳理"
---


# 1	agent接口
agent行为：
- 从agentinput、agentrunoption和可选的context session中获取任务和数据
- 执行任务，将执行过程、执行结果输出到agentevent iterator
- 可以通过context中的session暂存数据
```go
type Agent interface {
    Name(ctx context.Context) string
    Description(ctx context.Context) string
    Run(ctx context.Context, input *AgentInput) *AsyncIterator[*AgentEvent]
}
```
- 名字
- 描述
- 运行接口
## 1.1	run实现
future模式（尚未完成但是最终会有结果的操作）：
- 创建一对迭代器+生成器
- 启动agent，传入生成器处理agentinput，产生新的实践时写入生成器，调用方在迭代器中消费
- agent任务启动返回一个迭代器供调用方消费
## 1.2	父子
- 一个agent可以添加父agent，也可以添加子agent
- agent执行任务时可以将任务transfer给其父agent或子agent
```go
type OnSubAgents interface {
    OnSetSubAgents(ctx context.Context, subAgents []Agent) error
    OnSetAsSubAgent(ctx context.Context, parent Agent) error

    OnDisallowTransferToParent(ctx context.Context) error
}
```
## 1.3	状态管理
在一次运行生命周期内，每个agent可以通过context中的session读写数据，这是一个map，一次中断和恢复也是一个生命周期
```go
func SetSessionValue(ctx context.Context, key string, value any) {
    // omit code
}

func GetSessionValue(ctx context.Context, key string) (any, bool) {
    // omit code
}
```
# 2	agent组合
agent间协作方式：
- transfer：直接将一个任务转交给另一个任务，当前agent自己的任务结束后直接退出，不关心转交的agent的执行状态
- toolcall：将另一个agent当做一个tool，等待另一个agent响应进行接下来的处理
agentinput上下文策略：
- 上游agent全对话：获取上游agent的完整对话记录
- 全新任务描述：忽略上游agent的完整对话记录，给出一个全新任务总结作为子agent的输入
决策自主性：
- 自主决策：agnet自己觉得下游agent调用
- 预设决策：事先预设好的agent调用顺序
## 2.1	subagent
- transfer
- 上游全对话
- 自主决策
父agent+子agent列表：
- 只能有一个父
- 多个子agent
- agentname要唯一
```go
func SetSubAgents(ctx context.Context, agent Agent, subAgents []Agent) (Agent, error) {
    // omit code
}
```
## 2.2	workflow
每个agent拿到的agentinput都一样，然后按照顺序、并行和循环三种拓扑执行
### 2.2.1	顺序
- transfer
- 上游全对话
- 预设决策
将子agent列表按顺序依次执行一遍
注意：agent只能看到上游输出，这里子agent之间不是上下游关系，所以后执行的子agent看不到先执行的子agent的输出
```go
type SequentialAgentConfig struct {
    Name        string
    Description string
    SubAgents   []Agent
}

func NewSequentialAgent(ctx context.Context, config *SequentialAgentConfig) (Agent, error) {
    // omit code
}

```
### 2.2.2	并行
- transfer
- 上游全对话
- 预设决策
子agent列表中所有agent基于同样的上下文并发执行，直到所有agent执行结束
```go
type ParallelAgentConfig struct {
    Name        string
    Description string
    SubAgents   []Agent
}

func NewParallelAgent(ctx context.Context, config *ParallelAgentConfig) (Agent, error) {
    // omit code
}
```
### 2.2.3	loop
- transfer
- 上游全对话
- 预设决策
子agent列表按顺序执行，循环n（配置）轮
```go
type LoopAgentConfig struct {
    Name        string
    Description string
    SubAgents   []Agent

    MaxIterations int
}

func NewLoopAgent(ctx context.Context, config *LoopAgentConfig) (Agent, error) {
    // omit code
}
```
## 2.3	agentastool
- toolcall：需要等待子agent的响应
- 全新任务描述
- 自主决策
将一个agent转换成tool供其他agent调用
```go
func NewAgentTool(_ context.Context, agent Agent, options ...AgentToolOption) tool.BaseTool {
    // omit code
}
```
# 3	singleagent
提供了开箱即用的agent
## 3.1	chatmodelagent
- 一种react范式的agent
- 基于graph编排的react agent控制流
- 通过callback.Handler导出react agent运行过程中产生的事件转换成AgentEvent
```go
type ChatModelAgentConfig struct {
    Name        string
    Description string
    Instruction string

    Model model.ToolCallingChatModel

    ToolsConfig ToolsConfig

    // optional
    GenModelInput GenModelInput

    // Exit tool. Optional, defaults to nil, which will generate an Exit Action.
    // The built-in implementation is 'ExitTool'
    Exit tool.BaseTool

    // optional
    OutputKey string
}

func NewChatModelAgent(_ context.Context, config *ChatModelAgentConfig) (*ChatModelAgent, error) {
    // omit code
}
```
- 包含model和tools
# 4	agent运行
AgentRunner是Agent的执行器，使用adk.NewRunner包装，只有通过Runner执行时才能使用adk功能：
- 中断与恢复
- 切面
- context环境预处理
```go
type RunnerConfig struct {
    EnableStreaming bool
}

func NewRunner(_ context.Context, conf RunnerConfig) *Runner {
    // omit code
}
```
## 4.1	一个例子
```go
func runWithRunner() {
    ctx := context.Background()
    
    // 创建 Runner
    runner := adk.NewRunner(ctx, adk.RunnerConfig{
        EnableStreaming: true,
    })
    
    // 执行代理
    messages := []adk.Message{
        schema.UserMessage("What's the weather like today?"),
    }
    
    events := runner.Run(ctx, agent, messages)
    for {
        event, ok := events.Next()
        if !ok {
            break
        }
        
        // 处理事件
        handleEvent(event)
    }
}

```