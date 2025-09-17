---
title: "eino-workflow-agent"
date: 2025-09-17T16:36:29+08:00
draft: false
categories: ["eino"]
tags: ["eino"]
author: "Baike"
description: "eino梳理"
---


所谓工作流就是agent之间的协作是预先定义好的，而不是运行时agent动态决定的
agent输入：可以通过withhistoryrewriter自定义agentinput的生成方式
# 1	顺序agent
实现一个生成研究报告的序列agent，包含：
- planagent
- writeagent
依次调用他们
```go
package mutiagent

import (
	"context"
	"fmt"
	"log"
	"os"

	"github.com/cloudwego/eino-ext/components/model/openai"
	"github.com/cloudwego/eino/adk"
	"github.com/cloudwego/eino/components/model"
)

func newChatModel() model.ToolCallingChatModel {
	cm, err := openai.NewChatModel(context.Background(), &openai.ChatModelConfig{
		APIKey:  os.Getenv("OPENAI_API_KEY"),
		Model:   os.Getenv("OPENAI_MODEL"),
		BaseURL: os.Getenv("OPENAI_BASE_URL"),
	})
	if err != nil {
		log.Fatal(err)
	}
	return cm
}

func NewPlanAgent() adk.Agent {
	a, err := adk.NewChatModelAgent(context.Background(), &adk.ChatModelAgentConfig{
		Name:        "PlannerAgent",
		Description: "Generates a research plan based on a topic.",
		Instruction: `
You are an expert research planner. 
Your goal is to create a comprehensive, step-by-step research plan for a given topic. 
The plan should be logical, clear, and easy to follow.
The user will provide the research topic. Your output must ONLY be the research plan itself, without any conversational text, introductions, or summaries.`,
		Model: newChatModel(),
	})
	if err != nil {
		log.Fatal(err)
	}
	return a
}

func NewWriterAgent() adk.Agent {
	a, err := adk.NewChatModelAgent(context.Background(), &adk.ChatModelAgentConfig{
		Name:        "WriterAgent",
		Description: "Writes a report based on a research plan.",
		Instruction: `
You are an expert academic writer.
You will be provided with a detailed research plan.
Your task is to expand on this plan to write a comprehensive, well-structured, and in-depth report.
The user will provide the research plan. Your output should be the complete final report.`,
		Model: newChatModel(),
	})
	if err != nil {
		log.Fatal(err)
	}
	return a
}

func SequentialAgent() {
	ctx := context.Background()

	a, err := adk.NewSequentialAgent(ctx, &adk.SequentialAgentConfig{
		Name:        "ResearchAgent",
		Description: "A sequential workflow for planning and writing a research report.",
		SubAgents:   []adk.Agent{NewPlanAgent(), NewWriterAgent()},
	})
	if err != nil {
		log.Fatal(err)
	}

	runner := adk.NewRunner(ctx, adk.RunnerConfig{
		Agent: a,
	})

	iter := runner.Query(ctx, "The history of Large Language Models")
	for {
		event, ok := iter.Next()
		if !ok {
			break
		}
		if event.Err != nil {
			fmt.Printf("Error: %v\n", event.Err)
			break
		}
		msg, err := event.Output.MessageOutput.GetMessage()
		if err != nil {
			log.Fatal(err)
		}
		fmt.Printf("Agent[%s]:\n %+v\n\n===========\n\n", event.AgentName, msg)
	}
}

```
# 2	循环agent
定义agent然后指定最大迭代次数
```go
adk.NewLoopAgent(ctx, &adk.LoopAgentConfig{
    Name:          "name",
    Description:   "description",
    SubAgents:     []adk.Agent{a1,a2},
    MaxIterations: 3,
})
```
# 3	并发agent
定义并发的agent列表
```go
adk.NewParallelAgent(ctx, &adk.ParallelAgentConfig{
    Name:        "name",
    Description: "desc",
    SubAgents:   []adk.Agent{a1,a2},
})
```

