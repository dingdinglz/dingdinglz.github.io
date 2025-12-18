---
title: agent一把梭，google家Agent Development Kit
description: adk的概念和使用
date: 2025-11-24 02:00:00+0000
categories:
    - go
    - github
tags:
    - golang
draft: true
---

## 概述

Agent已经成为今年最热的话题，从workflow到agent，也意味着人们对ai的能力更加信任，llm更上一层楼，这也让包括我在内的技术狂热者更加兴奋，前端时间gemini3的发布更是直接引爆整个圈子，也不由得让我把目光投到这个google开源的新框架上，我本来以为这个框架就是对标字节的eino，然后才发现google不需要对标任何人，adk连续冲上github trending也证实了其在圈子内的认可度，这真的是在golang中可以替代langchain的超级利器

https://github.com/google/adk-go

**本文书写顺序紧跟官方文档，配合官方文档食用更佳：https://google.github.io/adk-docs/get-started/go/**

**但是官方文档中对go-sdk的解释相当不详细，很多已实现但未呈现的内容，甚至很多章节没有go的示例，这才是本篇文章的目的**

## Get Started

先来看它的入门示例：

``` go
package main

import (
	"context"
	"log"
	"os"

	"google.golang.org/adk/agent"
	"google.golang.org/adk/agent/llmagent"
	"google.golang.org/adk/cmd/launcher"
	"google.golang.org/adk/cmd/launcher/full"
	"google.golang.org/adk/model/gemini"
	"google.golang.org/adk/tool"
	"google.golang.org/adk/tool/geminitool"
	"google.golang.org/genai"
)

func main() {
	ctx := context.Background()

	model, err := gemini.NewModel(ctx, "gemini-3-pro-preview", &genai.ClientConfig{
		APIKey: os.Getenv("GOOGLE_API_KEY"),
	})
	if err != nil {
		log.Fatalf("Failed to create model: %v", err)
	}

	timeAgent, err := llmagent.New(llmagent.Config{
		Name:        "hello_time_agent",
		Model:       model,
		Description: "Tells the current time in a specified city.",
		Instruction: "You are a helpful assistant that tells the current time in a city.",
		Tools: []tool.Tool{
			geminitool.GoogleSearch{},
		},
	})
	if err != nil {
		log.Fatalf("Failed to create agent: %v", err)
	}

	config := &launcher.Config{
		AgentLoader: agent.NewSingleLoader(timeAgent),
	}

	l := full.NewLauncher()
	if err = l.Execute(ctx, config, os.Args[1:]); err != nil {
		log.Fatalf("Run failed: %v\n\n%s", err, l.CommandLineSyntax())
	}
}
```

我们可以看到也是模型分离的设计，你的模型可以直接`model.GenerateContent()`当一个普通的llm用，这也是非常好的抽象，同时可以方便我们集成很多其他的模型

然后是定义agent，这个跟python的mcp-agent很像，很方便的定义形式

再往下有个launcher，这个就是很方便的东西，方便直接启动一个终端or一个web网站调试，大家可以试下直接`go run .`就是交互式命令行界面，用`go run agent.go web api webui`就会启动一个web界面。这种很方便调试，同时带有trace和eval方便评估效果等等

入门的example就是带大家看个效果玩玩，那我们进入正题

## Build your Agent

### Muli-tool agent

多工具，老生常谈了，写到这里我们来谴责一下官方文档里没有go示例吧（

不过在这里有一个示例可以看，下面给大家剖析一下：https://github.com/google/adk-go/blob/main/examples/tools/multipletools/main.go

首先，大家都知道，Agent = LLM + Tool，那么它跟workflow最大的区别在哪呢，我认为最大的区别是自主决策能力，也就是workflow是人类规划好的流程，先调哪个工具再调哪个工具，而agent是由ai自主决定要调哪个工具，所以是质的突破，当然，在agent中内嵌workflow（一般通过prompt限定）也是件很常见的事

回归主题，定义一个tool如下：

``` go
package main

import (
	"context"
	"fmt"
	"log"
	"os"

	"google.golang.org/adk/agent"
	"google.golang.org/adk/agent/llmagent"
	"google.golang.org/adk/cmd/launcher"
	"google.golang.org/adk/cmd/launcher/full"
	"google.golang.org/adk/model/gemini"
	"google.golang.org/adk/tool"
	"google.golang.org/adk/tool/functiontool"
	"google.golang.org/genai"
)

func main() {
	ctx := context.Background()

	model, err := gemini.NewModel(ctx, "gemini-3-pro-preview", &genai.ClientConfig{
		APIKey: os.Getenv("GOOGLE_API_KEY"),
	})

	if err != nil {
		log.Fatalf("Failed to create model: %v", err)
	}

	type Input struct {
		A int `json:"a"`
		B int `json:"b"`
	}

	type Output struct {
		Answer int `json:"answer"`
	}

	calcTool, err := functiontool.New(functiontool.Config{
		Name:        "calculate",
		Description: "Calculate the sum of two numbers",
	}, func(ctx tool.Context, input Input) (Output, error) {
		fmt.Println(input.A, "+", input.B, "is calc")
		return Output{
			Answer: input.A + input.B,
		}, nil
	})
	if err != nil {
		log.Fatalf("Failed to create tool: %v", err)
	}

	timeAgent, err := llmagent.New(llmagent.Config{
		Name:        "hello_agent",
		Model:       model,
		Description: "Helpful assistant",
		Instruction: "You are a helpful assistant that solve user's problem with tools.",
		Tools: []tool.Tool{
			calcTool,
		},
	})
	if err != nil {
		log.Fatalf("Failed to create agent: %v", err)
	}

	config := &launcher.Config{
		AgentLoader: agent.NewSingleLoader(timeAgent),
	}

	l := full.NewLauncher()
	if err = l.Execute(ctx, config, os.Args[1:]); err != nil {
		log.Fatalf("Run failed: %v\n\n%s", err, l.CommandLineSyntax())
	}
}
```

可以看到，定义一个工具主要通过：`functiontool.New`，用过function call的朋友对这个应该很熟悉，定义函数名和描述，然后传一个实现函数，这个实现函数通过定义input和output结构体实现，这个应该很简单，不过多赘述了

但这里有个坑，我们自己定义的function tool不能和gemini tool混用，详见这个issue：https://github.com/google/adk-go/issues/265

上面的问题设计的有点垃圾，不过我相信谷歌霸霸会解决这个问题的

你也可以用子agent的模式来解决该问题，即把一个agent作为一个工具：`agenttool.New`，详见这个示例：https://github.com/google/adk-go/blob/main/examples/tools/multipletools/main.go

文档还不太行，有点难啃，这也就是这篇blog要解决的核心问题，我们继续看吧

补充一下：参数其实也可以打description，这个就需要自定义`InputSchema`和`OutputSchema`，看下面的示例应该就能懂：

``` go
// 1. 定义参数结构
type Args struct {
    Fruit string `json:"fruit"`
}
 
// 2. 获取自动推断的模式
ischema, err := jsonschema.For[Args](nil)
 
// 3. 自定义模式：限制枚举值
fruit, ok := ischema.Properties["fruit"]
fruit.Description = "print the remaining quantity of the item."
fruit.Enum = []any{"mandarin", "kiwi"}  // 只允许这两个值
 
// 4. 创建工具时使用自定义模式
inventoryTool, err := functiontool.New(functiontool.Config{
    Name:        "print_quantity",
    Description: "print the remaining quantity of the given fruit.",
    InputSchema: ischema,  // 使用自定义的模式
}, func(ctx tool.Context, input Args) (any, error) {
    // 处理逻辑
    return nil, nil
})
```

### Agent team

