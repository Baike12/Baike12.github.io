---
title: "eino-adk扩展"
date: 2025-09-17T16:36:29+08:00
draft: false
categories: ["eino"]
tags: ["eino"]
author: "Baike"
description: "eino梳理"
---


# 1	中断与恢复
允许一个正在运行的agent主动中断执行，保存当前状态，在稍后从中断点恢复，用于：
- 需要外部输入
- 长时间等待或者可暂停的任务
## 1.1	interrupted action
通过产生interrupted action这个AgentEvent来主动中断Runner运行：
```go
// github.com/cloudwego/eino/adk/interface.go
type AgentAction struct {
    // other actions
    Interrupted *InterruptInfo
    // other actions
}

// github.com/cloudwego/eino/adk/interrupt.go
type InterruptInfo struct {
    Data any
}
```
中断发生时，interruptinfo中附带自定义的中断信息，这个信息
- 被传递给调用者，让调用者知道为什么中断
- 在恢复时传递给中断的agent，让这个agent根据中断信息恢复
# 2	状态持久化
当Runner捕获到带有interrupted action的event时，会立即终止当前流程，如果设置了checkpointstore
```go
// github.com/cloudwego/eino/adk/runner.go
type RunnerConfig struct {
    // other fields
    CheckPointStore CheckPointStore
}

// github.com/cloudwego/eino/adk/interrupt.go
type CheckPointStore interface {
    Set(ctx context.Context, key string, value []byte) error
    Get(ctx context.Context, key string) ([]byte, bool, error)
}
```
并在调用Runner时通过withcheckpointid传入了checkpointid
```go
// github.com/cloudwego/eino/adk/interrupt.go
func WithCheckPointID(id string) _AgentRunOption_
```
Runner会在终止运行后将当前运行状态包括对话历史、原始输入、interruptinfo等信息存储以checkpointid作为key的checkpointstore中
>[!note]
>eino会使用gob来序列化要保存的信息，所以要存储的类型是自定义类型需要使用gob.register或gob.registerName注册，eino内置的类型已经默认注册了。
>为什么要序列化？因为要保存到外部存储比如redis中，序列化后可以提供标准的二进制格式存储；然后恢复的时候通过注册信息恢复成类型
# 3	恢复
调用Runner的resume方法，传入checkpointid就可以恢复
```go
// github.com/cloudwego/eino/adk/runner.go
func (r *Runner) Resume(ctx context.Context, checkPointID string, opts ...AgentRunOption) (*AsyncIterator[*AgentEvent], error)
```
可以恢复的agent需要实现resumeableAgent接口，其中中断信息和enablestreaming会作为输入提供给agent
```go
// github.com/cloudwego/eino/adk/interface.go
type ResumableAgent interface {
    Agent

    Resume(ctx context.Context, info *ResumeInfo, opts ...AgentRunOption) *AsyncIterator[*AgentEvent]
}

// github.com/cloudwego/eino/adk/interrupt.go
type ResumeInfo struct {
    EnableStreaming bool
    *_InterruptInfo_
}
```
也可以通过在调用resume时传入Agentrunoption来在恢复时传入新消息



