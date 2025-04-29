---
title: 思维导图绘制（mcp版）
description: golang (eino + mcp) react agent实战，实现一个思维导图绘制器
date: 2025-04-29 2:00:00+0000
categories:
    - release
    - go
    - github
tags:
    - ai
    - mcp
---

## Intro

本文章主要介绍如何用golang的eino框架，加上mcp，实现一个绘制思维导图的小工具，项目目前已经开源在：https://github.com/dingdinglz/mind-maker ，实现的效果可以进入仓库查看readme， 本文主要聊聊设计思路，什么是mcp，如何调用mcp

## mcp

### 什么是mcp

MCP 是一个开放协议，它标准化了应用程序如何向大型语言模型（LLMs）提供上下文。可以将MCP想象成AI应用的USB-C端口。正如USB-C提供了一种标准化的方式来连接你的设备到各种外围设备和配件，MCP提供了一种标准化的方式来连接AI模型到不同的数据源和工具。

其实也就是为llm提供三方工具，让llm获得更多的能力，但这似乎听起来很像tool call，mcp与tool call相比，在于将工具整合起来，并且提供了标准的协议进行通信。在调用的底层，其实也就是将mcp的每个函数加载成tool的形式提供给llm。

但是，mcp可以由提供服务的厂家开发，而并非使用llm的人自己，方便了快速集成工具。

### 为什么mcp

MCP 帮助你在大型语言模型（LLM）之上构建代理和复杂的工作流程。LLM 经常需要与数据和工具集成，MCP 提供：

- 一个不断增长的工具集成列表，你的 LLM 可以直接接入
- 在不同的 LLM 提供商和供应商之间可以灵活切换
- 在您的基础设施内保护您的隐私数据

### 怎么mcp

mcp的调用形式目前分为两种，一种是stdio，一种是sse，前者是通过类似于调用命令行的方法去传递数据和取得结果，后者是利用sse进行通信。

大部分语言目前已经有了三方实现的mcp-client包，因此我们对于实现的技术细节可以不那么关心。

我们在项目中用到的，调用mcp的golang包是mcp-go，它同样可以作为server的开发工具包

## eino

https://www.cloudwego.io/zh/docs/eino/

看该blog之前把eino的用法看一遍

### 什么是eino

Eino 是基于 Golang 的 AI 应用开发框架 ， 类似于python的langchain

### eino的优势

- eino集成了相当多的大模型调用协议
- eino提供了一种以图的形式，进行ai调用能力编排的功能
- eino提供了几种ai的封装好的开发模式，例如muli-agent，react

当然，目前eino也支持了集成mcp，因此，我们的代码中要做的更多就是设计方面的考虑，在实现上相对来说较为简单。

## 项目流程

让我们梳理一下项目流程，首先，拿到知识点，然后给llm，让他帮我们画即可，然后把生成的内容保存。

这非常简单，但我们需要引入下列能力：

1. 联网搜索，让llm可以获得到最新的知识点（因此本项目可以对近期的热点事件进行思维导图的绘制），且避免出现幻觉。使用方法，eino提供了搜索的工具，比如google、duckduckgo，我们用他的duckduckgo提供搜索服务。
2. 绘制思维导图的能力，我们用到了mindmap-mcp-server
3. 我们可以手动保存文件，也可以让ai保存，为了演示mcp的用法，我们用@modelcontextprotocol/server-filesystem来实现llm对文件的操作

## Mcp调用部分

首先，我们的配置文件需要自主配置mcp，以支持mcp的拓展，因此我们采用了以下格式:

``` json
{
    "mcps": [
        {
            "command": "uvx",
            "env": [],
            "args": [
                "mindmap-mcp-server",
                "--return-type",
                "html"
            ]
        },
        {
            "command": "npx",
            "env": [],
            "args": [
                "-y",
                "@modelcontextprotocol/server-filesystem",
                "填上运行项目的地址！！！"
            ]
        }
    ]
}
```

写在配置文件里，这与市面上大部分三方MCP的设置形式都是相同的，加载mcp的代码如下，其实就是将mcp全部转换成了tool call。

``` go
package main

import (
	"context"
	"fmt"
	"time"

	einomcp "github.com/cloudwego/eino-ext/components/tool/mcp"
	"github.com/cloudwego/eino/components/tool"
	"github.com/mark3labs/mcp-go/client"
	"github.com/mark3labs/mcp-go/mcp"
)

func GenerateTools() []tool.BaseTool {
	startTime := time.Now()
	ctx := context.Background()
	var res []tool.BaseTool
	initRequest := mcp.InitializeRequest{}
	initRequest.Params.ProtocolVersion = mcp.LATEST_PROTOCOL_VERSION
	initRequest.Params.ClientInfo = mcp.Implementation{
		Name:    "mind-maker-client",
		Version: "1.0.0",
	}
	for _, item := range ActivateConfig.Mcps {
		cli, e := client.NewStdioMCPClient(item.Command, item.Env, item.Args...)
		if e != nil {
			panic(e)
		}
		_, e = cli.Initialize(ctx, initRequest)
		if e != nil {
			panic(e)
		}
        // 将mcp转换为工具
		tools, e := einomcp.GetTools(ctx, &einomcp.Config{
			Cli: cli,
		})
		if e != nil {
			panic(e)
		}
		res = append(res, tools...)
	}
	timeLast := time.Since(startTime)
	fmt.Println("生成mcp工具列表用时:", timeLast)
	return res
}
```

## 大模型调用部分

由于eino已经支持了react模式，因此我们直接拿过来用即可，因此要设计的核心点是prompt，目前项目中的prompt如下

```
下面，你需要对用户给出的知识点进行思维导图的绘制，然后将结果保存在mind.html，要求思维导图结构清晰，逻辑正确，覆盖面广，生成的内容的语言应当使用中文
制作前你需要进行搜索收集相关信息，如果收集到的信息不是中文，请翻译成中文，结果一定要是中文，记住，一定要将结果保存到mind.html，生成的内容的语言应当使用中文
```

主要做的内容其实也就是对语言的限制，以及嘱咐ai记得调用相关能力。

完整调用过程：

``` go
package main

import (
	"bufio"
	"context"
	"errors"
	"fmt"
	"io"
	"os"
	"strings"
	"time"

	"github.com/cloudwego/eino-ext/components/model/openai"
	"github.com/cloudwego/eino-ext/components/tool/duckduckgo"
	"github.com/cloudwego/eino/compose"
	"github.com/cloudwego/eino/flow/agent"
	"github.com/cloudwego/eino/flow/agent/react"
	"github.com/cloudwego/eino/schema"
)

func main() {
	ctx := context.Background()

	fmt.Println("mind-maker by dinglz")
	fmt.Println("https://github.com/dingdinglz/mind-maker")

	LoadConfig()

	if FileExist("mind.html") {
		os.Remove("mind.html")
	}

	fmt.Print("请输入要生成思维导图的知识点:")
	reader := bufio.NewReader(os.Stdin)
	question, e := reader.ReadString('\n')
	if e != nil {
		panic(e)
	}
	question = strings.ReplaceAll(question, " ", "")
	question = strings.ReplaceAll(question, "\n", "")
	fmt.Println("开始对知识点", question, "绘制思维导图")

	chatModel, e := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		BaseURL: ActivateConfig.Model.BaseURL,
		APIKey:  ActivateConfig.Model.ApiKey,
		Model:   ActivateConfig.Model.Model,
	})
	if e != nil {
		panic(e)
	}

	tools := GenerateTools()
	ducktool, e := duckduckgo.NewTool(ctx, &duckduckgo.Config{})
	if e != nil {
		panic(e)
	}
	tools = append(tools, ducktool)
	ragent, e := react.NewAgent(ctx, &react.AgentConfig{
		Model: chatModel,
		ToolsConfig: compose.ToolsNodeConfig{
			Tools: tools,
		},
	})
	if e != nil {
		panic(e)
	}

	startTime := time.Now()
	sr, e := ragent.Stream(ctx, []*schema.Message{
		{
			Role:    schema.System,
			Content: "下面，你需要对用户给出的知识点进行思维导图的绘制，然后将结果保存在mind.html，要求思维导图结构清晰，逻辑正确，覆盖面广，生成的内容的语言应当使用中文\n\n制作前你需要进行搜索收集相关信息，如果收集到的信息不是中文，请翻译成中文，结果一定要是中文，记住，一定要将结果保存到mind.html，生成的内容的语言应当使用中文",
		},
		{
			Role:    schema.User,
			Content: question,
		},
	}, agent.WithComposeOptions(compose.WithCallbacks(&LoggerCallback{})))
	if e != nil {
		panic(e)
	}

	defer sr.Close()
	for {
		msg, e := sr.Recv()
		if e != nil {
			if errors.Is(e, io.EOF) {
				break
			}
			panic(e)
		}
		fmt.Print(msg.Content)
	}
	fmt.Println()
	fmt.Println("生成思维导图用时：", time.Since(startTime))
	if FileExist("mind.html") {
		fmt.Println("生成思维导图已保存到：mind.html")
	} else {
		fmt.Println("生成失败！请重试")
	}
}
```

## 总结

这只是一个相当于一个简单的llm集成mcp的示例 + eino的示例，完整代码请去github查看

## 结语

近期拿到了一个生成ui原型图的prompt，还挺有意思的，也挺强，基于此开发了个ui原型图生成的网页，有时间就发篇blog唠唠