---
title: "eino-chatmodelagent"
date: 2025-09-17T16:36:29+08:00
draft: false
categories: ["eino"]
tags: ["eino"]
author: "Baike"
description: "eino梳理"
---


# 1	示例：图书推荐agent
根据用户输入推荐相关图书
## 1.1	工具定义
```go
import (
    "context"
    "log"

    "github.com/cloudwego/eino/components/tool"
    "github.com/cloudwego/eino/components/tool/utils"
)

type BookSearchInput struct {
    Genre     string `json:"genre" jsonschema:"description=Preferred book genre,enum=fiction,enum=sci-fi,enum=mystery,enum=biography,enum=business"`
    MaxPages  int    `json:"max_pages" jsonschema:"description=Maximum page length (0 for no limit)"`
    MinRating int    `json:"min_rating" jsonschema:"description=Minimum user rating (0-5 scale)"`
}

type BookSearchOutput struct {
    Books []string
}

func NewBookRecommender() tool.InvokableTool {
    bookSearchTool, err := utils.InferTool("search_book", "Search books based on user preferences", func(ctx context.Context, input *BookSearchInput) (output *BookSearchOutput, err error) {
       // search code
       // ...
       return &BookSearchOutput{Books: []string{"God's blessing on this wonderful world!"}}, nil
    })
    if err != nil {
       log.Fatalf("failed to create search book tool: %v", err)
    }
    return bookSearchTool
}
```
- 定义图书搜索的输入输出
- 定义工具，使用InferTool封装图书搜索工具
## 1.2	创建chatmodel
```go
import (
    "context"
    "fmt"
    "log"
    "os"

    "github.com/cloudwego/eino-ext/components/model/openai"
    "github.com/cloudwego/eino/components/model"
)

func NewChatModel() model.ToolCallingChatModel {
    ctx := context.Background()
    apiKey := os.Getenv("OPENAI_API_KEY")
    openaiModel := os.Getenv("OPENAI_MODEL")

    cm, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
       APIKey: apiKey,
       Model:  openaiModel,
    })
    if err != nil {
       log.Fatal(fmt.Errorf("failed to create chatmodel: %w", err))
    }
    return cm
}
```
- 返回一个带工具调用的chatmodel
## 1.3	创建chatmodelagent
- name
- description：用于对其他agent暴露当前agent的功能
- instruction：描述当前agent的作用，用于传递给chatmodel定义当前agent的作用
```go
import (
    "context"
    "fmt"
    "log"

    "github.com/cloudwego/eino/adk"
    "github.com/cloudwego/eino/components/tool"
    "github.com/cloudwego/eino/compose"
)

func NewBookRecommendAgent() adk.Agent {
    ctx := context.Background()

    a, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
       Name:        "BookRecommender",
       Description: "An agent that can recommend books",
       Instruction: `You are an expert book recommender. Based on the user's request, use the "search_book" tool to find relevant books. Finally, present the results to the user.`,
       Model:       NewChatModel(),
       ToolsConfig: adk.ToolsConfig{
          ToolsNodeConfig: compose.ToolsNodeConfig{
             Tools: []tool.BaseTool{NewBookRecommender()},
          },
       },
    })
    if err != nil {
       log.Fatal(fmt.Errorf("failed to create chatmodel: %w", err))
    }

    return a
}
```
- 还需要定义工具，使用adk的工具配置接口
# 2	工具调用
react模式：
- 调用chatmodel
- llm返回action
- chatmodelagent执行工具调用
- 将工具调用结果返回chatmodel，开始新的循环，知道Chatmodel判断不需要调用Tool，结束
## 2.1	为agent配置tool
```go
// github.com/cloudwego/eino/adk/chatmodel.go

type ToolsConfig struct {
    compose.ToolsNodeConfig

    // Names of the tools that will make agent return directly when the tool is called.
    // When multiple tools are called and more than one tool is in the return directly list, only the first one will be returned.
    ReturnDirectly map[string]bool
}
```
- 复用了graph中的ToolsNodeConfig，但是提供了returnDirectly，chatmodelagent调用ReturnDirectly中的tool直接退出
- 如果没有配置工具，chatmodelagent就是一次chatmodel调用
# 3	GenModelInput
chatmodelagent创建时可以配置GenModelInput，agent被调用时会使用这个方法生成chatmodel的初始输入：
```go
type GenModelInput func(ctx context.Context, instruction string, input *AgentInput) ([]Message, error)
```
- 相当于这个方法可以定义agent自身的功能和职责
agent提供了默认的genmodelinput：
- 将instruction作为system Message加到agentinput前，相当于定义agent自身
- 以sessionvalues为valriables渲染上一步中得到的Message list
# 4	outputkey
创建chatmodelagent时可以配置outputkey，配置后agent的最后输入Message会以outputkey为key添加到sessionvalues中
# 5	Exit
可以用newchatmodelagent中的exit字段配置一个tool，llm调用这个tool后，chatmodelagent将直接退出。eino adk内置了一个exitTool，可以直接使用
```go
// github.com/cloudwego/eino/adk/chatmodel.go

type ExitTool struct{}

func (et ExitTool) Info(_ context.Context) (*schema.ToolInfo, error) {
    return ToolInfoExit, nil
}

func (et ExitTool) InvokableRun(ctx context.Context, argumentsInJSON string, _ ...tool.Option) (string, error) {
    type exitParams struct {
       FinalResult string `json:"final_result"`
    }

    params := &exitParams{}
    err := sonic.UnmarshalString(argumentsInJSON, params)
    if err != nil {
       return "", err
    }

    err = SendToolGenAction(ctx, "exit", NewExitAction())
    if err != nil {
       return "", err
    }

    return params.FinalResult, nil
}
```
# 6	Transfer
chatmodelagent实现了OnsubAgents接口，可以设置agent的父或子agent，设置之后，chatmodelagent会增加一个Transfer Tool，并在Prompt中指示chatmodel在需要Transfer时调用这个tool并以Transfer目标agentname作为tool的输入
相当于内置了委派功能
# 7	agenttool
chatmodelagent提供了newagenttool，可以直接把agent转换为tool给其他agent调用
```go
// github.com/cloudwego/eino/adk/agent_tool.go

func NewAgentTool(_ context.Context, agent Agent, options ...AgentToolOption) tool.BaseTool
```
比如将图书推荐agent转换为tool给其他应用调用
```go
bookRecommender := NewBookRecommendAgent()
bookRecommendeTool := NewAgentTool(ctx, bookRecommender)

// other agent
a, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
    // xxx
    ToolsConfig: adk.ToolsConfig{
        ToolsNodeConfig: compose.ToolsNodeConfig{
            Tools: []tool.BaseTool{bookRecommendeTool},
        },
    },
})
```
# 8	中断与恢复
提供一个新工具，当用户提供的信息补足语支持推荐时，agent调用这个工具来询问更多的信息，工具返回特殊的错误使graph触发中断并向外抛出自定义信息，恢复时重新运行这个工具
创建一个中断
```go
// github.com/cloudwego/eino/adk/interrupt.go

func NewInterruptAndRerunErr(extra any) error
```
创建一个toolOption在恢复时传递新的输入
```go
import (
    "github.com/cloudwego/eino/components/tool"
)

type askForClarificationOptions struct {
    NewInput *string
}

func WithNewInput(input string) tool.Option {
    return tool.WrapImplSpecificOptFn(func(t *askForClarificationOptions) {
       t.NewInput = &input
    })
}
```
- 将询问请求封装成一个tool.Option
完整实现：
```go
import (
    "context"
    "log"

    "github.com/cloudwego/eino/components/tool"
    "github.com/cloudwego/eino/components/tool/utils"
    "github.com/cloudwego/eino/compose"
)

type askForClarificationOptions struct {
    NewInput *string
}

func WithNewInput(input string) tool.Option {
    return tool.WrapImplSpecificOptFn(func(t *askForClarificationOptions) {
       t.NewInput = &input
    })
}

type AskForClarificationInput struct {
    Question string `json:"question" jsonschema:"description=The specific question you want to ask the user to get the missing information"`
}

func NewAskForClarificationTool() tool.InvokableTool {
    t, err := utils.InferOptionableTool(
       "ask_for_clarification",
       "Call this tool when the user's request is ambiguous or lacks the necessary information to proceed. Use it to ask a follow-up question to get the details you need, such as the book's genre, before you can use other tools effectively.",
       func(ctx context.Context, input *AskForClarificationInput, opts ...tool.Option) (output string, err error) {
          o := tool.GetImplSpecificOptions[askForClarificationOptions](nil, opts...)
          if o.NewInput == nil {
             return "", compose.NewInterruptAndRerunErr(input.Question)
          }
          return *o.NewInput, nil
       })
    if err != nil {
       log.Fatal(err)
    }
    return t
}
```
- 创建一个询问工具
可以将这个询问请求放到之前的图书推荐中
```go
func NewBookRecommendAgent() adk.Agent {
    // xxx
    a, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
       // xxx
       ToolsConfig: adk.ToolsConfig{
          ToolsNodeConfig: compose.ToolsNodeConfig{
             Tools: []tool.BaseTool{NewBookRecommender(), NewAskForClarificationTool()},
          },
       },
    })
    // xxx
}
```
然后在runner中配置checkpointstotr，然后在调用agent的时候传入checkpointid，在恢复时使用
eino graph在中断时，会将graph的interruptinfo放到interrupted.data中：
```go
func main() {
    ctx := context.Background()
    a := internal.NewBookRecommendAgent()
    runner := adk.NewRunner(ctx, adk.RunnerConfig{
       Agent:           a,
       CheckPointStore: newInMemoryStore(),
    })
    iter := runner.Query(ctx, "recommend a book to me", adk.WithCheckPointID("1"))
    for {
       event, ok := iter.Next()
       if !ok {
          break
       }
       if event.Err != nil {
          log.Fatal(event.Err)
       }
       if event.Action != nil && event.Action.Interrupted != nil {
          fmt.Printf("\ninterrupt happened, info: %+v\n", event.Action.Interrupted.Data.(*adk.ChatModelAgentInterruptInfo).Info.RerunNodesExtra["ToolNode"])
          continue
       }
       msg, err := event.Output.MessageOutput.GetMessage()
       if err != nil {
          log.Fatal(err)
       }
       fmt.Printf("\nmessage:\n%v\n======\n\n", msg)
    }
    
    // xxxxxx
}
```
恢复
```go
	nInput := "fiction"

	iter, err := runner.Resume(ctx, "1", adk.WithToolOptions([]tool.Option{WithNewInput(nInput)}))
	if err != nil {
		log.Fatal(err)
	}
	for {
		event, ok := iter.Next()
		if !ok {
			break
		}

		if event.Err != nil {
			log.Fatal(event.Err)
		}
		// fmt.Println("======\nresume event:\n======\n")
		prints.Event(event)
	}
```
# 9	完整的例子
```go
package main

import (
	// "bufio"
	"context"
	"fmt"
	"log"
	"os"

	"github.com/cloudwego/eino-examples/adk/common/prints"
	"github.com/cloudwego/eino-ext/components/model/openai"
	"github.com/cloudwego/eino/components/model"
	"github.com/cloudwego/eino/components/tool"
	"github.com/cloudwego/eino/components/tool/utils"

	"github.com/cloudwego/eino/adk"
	"github.com/cloudwego/eino/compose"
)

type BookSearchInput struct {
	Genre     string `json:"genre" jsonschema:"description=Preferred book genre,enum=fiction,enum=sci-fi,enum=mystery,enum=biography,enum=business"`
	MaxPages  int    `json:"max_pages" jsonschema:"description=Maximum page length (0 for no limit)"`
	MinRating int    `json:"min_rating" jsonschema:"description=Minimum user rating (0-5 scale)"`
}

type BookSearchOutput struct {
	Books []string
}

func NewBookRecommender() tool.InvokableTool {
	bookSearchTool, err := utils.InferTool("search_book", "Search books based on user preferences", func(ctx context.Context, input *BookSearchInput) (output *BookSearchOutput, err error) {
		// search code
		// ...
		return &BookSearchOutput{Books: []string{"God's blessing on this wonderful world!"}}, nil
	})
	if err != nil {
		log.Fatalf("failed to create search book tool: %v", err)
	}
	return bookSearchTool
}

func NewChatModel() model.ToolCallingChatModel {
	ctx := context.Background()
	apiKey := os.Getenv("OPENAI_API_KEY")
	openaiModel := os.Getenv("OPENAI_MODEL")
	openaiBaseURL := os.Getenv("OPENAI_BASE_URL")

	cm, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey:  apiKey,
		Model:   openaiModel,
		BaseURL: openaiBaseURL,
	})
	if err != nil {
		log.Fatal(fmt.Errorf("failed to create chatmodel: %w", err))
	}
	return cm
}

func NewBookRecommendAgent() adk.Agent {
	ctx := context.Background()

	a, err := adk.NewChatModelAgent(ctx, &adk.ChatModelAgentConfig{
		Name:        "BookRecommender",
		Description: "An agent that can recommend books",
		Instruction: `You are an expert book recommender. Based on the user's request, use the "search_book" tool to find relevant books. Finally, present the results to the user.`,
		Model:       NewChatModel(),
		ToolsConfig: adk.ToolsConfig{
			ToolsNodeConfig: compose.ToolsNodeConfig{
				Tools: []tool.BaseTool{NewAskForClarificationTool(), NewBookRecommender()},
			},
		},
	})
	if err != nil {
		log.Fatal(fmt.Errorf("failed to create chatmodel: %w", err))
	}

	return a
}

type askForClarificationOptions struct {
	NewInput *string
}

func WithNewInput(input string) tool.Option {
	return tool.WrapImplSpecificOptFn(func(t *askForClarificationOptions) {
		t.NewInput = &input
	})
}

type AskForClarificationInput struct {
	Question string `json:"question" jsonschema:"description=The specific question you want to ask the user to get the missing information"`
}

func NewAskForClarificationTool() tool.InvokableTool {
	t, err := utils.InferOptionableTool(
		"ask_for_clarification",
		// "Call this tool when the user's request is ambiguous or lacks the necessary information to proceed. Use it to ask a follow-up question to get the details you need, such as the book's genre, before you can use other tools effectively.",
		"you must always call this tool to ask the user for the book's genre",
		func(ctx context.Context, input *AskForClarificationInput, opts ...tool.Option) (output string, err error) {
			o := tool.GetImplSpecificOptions[askForClarificationOptions](nil, opts...)
			if o.NewInput == nil {
				return "", compose.NewInterruptAndRerunErr(input.Question)
			}
			return *o.NewInput, nil
		})
	if err != nil {
		log.Fatal(err)
	}
	return t
}

type inMemoryStore struct {
	mem map[string][]byte
}

func newInMemoryStore() compose.CheckPointStore {
	return &inMemoryStore{
		mem: map[string][]byte{},
	}
}

func (i *inMemoryStore) Set(ctx context.Context, key string, value []byte) error {
	i.mem[key] = value
	return nil
}

func (i *inMemoryStore) Get(ctx context.Context, key string) ([]byte, bool, error) {
	v, ok := i.mem[key]
	return v, ok, nil
}

func main() {
	ctx := context.Background()
	a := NewBookRecommendAgent()
	runner := adk.NewRunner(ctx, adk.RunnerConfig{
		Agent:           a,
		EnableStreaming: true,
		CheckPointStore: newInMemoryStore(),
	})
	iter := runner.Query(ctx, "recommend a fiction book to me", adk.WithCheckPointID("1"))
	for {
		event, ok := iter.Next()
		if !ok {
			break
		}
		if event.Err != nil {
			log.Fatal(event.Err)
		}
		if event.Action != nil && event.Action.Interrupted != nil {
			fmt.Printf("======\ninterrupt happened, info: %+v\n===========\n", event.Action.Interrupted.Data.(*adk.ChatModelAgentInterruptInfo).Info.RerunNodesExtra["ToolNode"])
			continue
		}
		msg, err := event.Output.MessageOutput.GetMessage()
		if err != nil {
			log.Fatal(err)
		}
		fmt.Printf("=======\nmessage:\n%v\n======\n", msg)
	}

	// scanner := bufio.NewScanner(os.Stdin)
	// fmt.Print("\nyour input here: ")
	// scanner.Scan()
	// fmt.Println()
	// nInput := scanner.Text()
	// fmt.Println("======\nresume\n======\n")
	nInput := "fiction"

	iter, err := runner.Resume(ctx, "1", adk.WithToolOptions([]tool.Option{WithNewInput(nInput)}))
	if err != nil {
		log.Fatal(err)
	}
	for {
		event, ok := iter.Next()
		if !ok {
			break
		}

		if event.Err != nil {
			log.Fatal(event.Err)
		}
		// fmt.Println("======\nresume event:\n======\n")
		prints.Event(event)
	}
}

```
