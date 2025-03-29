---
title: Mcp with golang
description: golang如何与mcp server交互，如何用eino将mcp集成到llm的工作流中
date: 2025-03-29 02:00:00+0000
categories:
    - go
    - github
tags:
    - golang
    - mcp
---

## 什么是mcp

[MCP](https://modelcontextprotocol.io/) 是一种开放协议，通过标准化的服务器实现，使 AI 模型能够安全地与本地和远程资源进行交互。此列表重点关注可用于生产和实验性的 MCP 服务器，这些服务器通过文件访问、数据库连接、API 集成和其他上下文服务来扩展 AI 功能。

mcp最近如此之火，已经成为这几天llm界最大的话题（不是），市面上也涌现了一批功能强大，覆盖各个方面的mcp server，本文主要介绍如何调用这些mcp server，以及如何把mcp server集成到[Eino](https://www.cloudwego.io/zh/docs/eino/)（字节跳动开源的大模型应用开发框架）中，使我们的ai也可以调用mcp的能力。

Let‘s start

## 如何寻找mcp servers

这里有一份非常棒的mcp server列表，你可以寻找任何你想要的功能：https://github.com/punkpeye/awesome-mcp-servers

另外，各大api厂商，在原有http api的基础上，陆续开放了mcp server，例如百度地图，更方便ai的接入

下文，我们以https://github.com/blackwhite084/playwright-plus-python-mcp为例，进行mcp的调用，使用的client包是：https://github.com/mark3labs/mcp-go

## 前置准备

现有的mcp servers大部分都是基于npx，uvx或者直接一个可执行文件，我们要用到的这个mcp server基于uvx，[点我安装uv](https://docs.astral.sh/uv/)

playwright是一个浏览器自动化工具，playwright-plus-python-mcp基于其实现功能，[点我安装playwright](https://playwright.dev/docs/intro)

## mcp server 构成

一个mcp server会提供一个tool list，也就是支持的能力，并介绍对应tool的功能和需要传入的参数，以及返回的参数

## 获取tool list

根据文档，可以轻松的把获取tool list的功能写出来

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/mark3labs/mcp-go/client"
    "github.com/mark3labs/mcp-go/mcp"
)

func main() {
    c, err := client.NewStdioMCPClient(
        "uvx",
        []string{},
        "playwright-server",
    )
    if err != nil {
        log.Fatalf("Failed to create client: %v", err)
    }
    ctx := context.Background()
    initRequest := mcp.InitializeRequest{}
    initRequest.Params.ProtocolVersion = mcp.LATEST_PROTOCOL_VERSION
    initRequest.Params.ClientInfo = mcp.Implementation{
        Name:    "example-client",
        Version: "1.0.0",
    }
    initResult, err := c.Initialize(ctx, initRequest)
    if err != nil {
        log.Fatalf("Failed to initialize: %v", err)
    }
    fmt.Printf(
        "Initialized with server: %s %s\n\n",
        initResult.ServerInfo.Name,
        initResult.ServerInfo.Version,
    )
    fmt.Println("Listing available tools...")
    toolsRequest := mcp.ListToolsRequest{}
    tools, err := c.ListTools(ctx, toolsRequest)
    if err != nil {
        log.Fatalf("Failed to list tools: %v", err)
    }
    for _, tool := range tools.Tools {
        fmt.Printf("- %s: %s\n", tool.Name, tool.Description)
    }
    fmt.Println()

}
```

主要的流程也就是新建一个mcp的client，然后指定连接的方式，然后进行初始化，获取到工具列表

可以看到，支持的功能如下：

```
- playwright_navigate: Navigate to a URL,thip op will auto create a session
- playwright_screenshot: Take a screenshot of the current page or a specific element
- playwright_click: Click an element on the page using CSS selector
- playwright_fill: Fill out an input field
- playwright_evaluate: Execute JavaScript in the browser console
- playwright_click_text: Click an element on the page by its text content
- playwright_get_text_content: Get the text content of all elements
- playwright_get_html_content: Get the HTML content of the page
```

当然，这部分的内容在mcp-server的文档中也可以查询到

## 利用mcp server实现截图特定网页

那么，让我们利用这些tool实现一个简单的功能，对github.com进行截图，我们需要用到两个tool，`playwright_navigate`和`playwright_screenshot`，也就是先导航到对应网站，再进行截图，实现如下：

```go
package main

import (
    "context"
    "encoding/base64"
    "encoding/json"
    "fmt"
    "log"
    "os"

    "github.com/mark3labs/mcp-go/client"
    "github.com/mark3labs/mcp-go/mcp"
)

func main() {
    c, err := client.NewStdioMCPClient(
        "uvx",
        []string{},
        "playwright-server",
    )
    if err != nil {
        log.Fatalf("Failed to create client: %v", err)
    }
    ctx := context.Background()
    initRequest := mcp.InitializeRequest{}
    initRequest.Params.ProtocolVersion = mcp.LATEST_PROTOCOL_VERSION
    initRequest.Params.ClientInfo = mcp.Implementation{
        Name:    "example-client",
        Version: "1.0.0",
    }
    initResult, err := c.Initialize(ctx, initRequest)
    if err != nil {
        log.Fatalf("Failed to initialize: %v", err)
    }
    fmt.Printf(
        "Initialized with server: %s %s\n\n",
        initResult.ServerInfo.Name,
        initResult.ServerInfo.Version,
    )
    fmt.Println("Listing available tools...")
    toolsRequest := mcp.ListToolsRequest{}
    tools, err := c.ListTools(ctx, toolsRequest)
    if err != nil {
        log.Fatalf("Failed to list tools: %v", err)
    }
    for _, tool := range tools.Tools {
        fmt.Printf("- %s: %s\n", tool.Name, tool.Description)
    }
    fmt.Println()
    navigateRequest := mcp.CallToolRequest{}
    navigateRequest.Params.Name = "playwright_navigate"
    navigateRequest.Params.Arguments = map[string]interface{}{
        "url": "https://github.com/",
    }
    result, err := c.CallTool(ctx, navigateRequest)
    if err != nil {
        log.Fatalf("Failed to navigate: %v", err)
    }
    printToolResult(result)
    jRequest := mcp.CallToolRequest{}
    jRequest.Params.Name = "playwright_screenshot"
    jRequest.Params.Arguments = map[string]interface{}{
        "name": "test.png",
    }
    result, err = c.CallTool(ctx, jRequest)
    if err != nil {
        log.Fatalf("Failed to navigate: %v", err)
    }
    printToolResult(result)
    image := result.Content[0].(mcp.ImageContent)
    resPng, _ := base64.StdEncoding.DecodeString(image.Data)
    os.WriteFile("test.png", resPng, os.ModePerm)
}

func printToolResult(result *mcp.CallToolResult) {
    for _, content := range result.Content {
        if textContent, ok := content.(mcp.TextContent); ok {
            fmt.Println(textContent.Text)
        } else {
            jsonBytes, _ := json.MarshalIndent(content, "", "  ")
            fmt.Println(string(jsonBytes))

        }
    }
}
```

通过mcp.CallToolRequest进行调用，可以看到，返回的调用结果是mcp.Content的形式，根据文档的说明可以转成mcp.TextContent，mcp.ImageContent等等

运行完后本地的test.png图片已经保存：

![test.png](test.png)

## 回顾

上述利用mcp-server的能力简单实现了一个浏览器自动化的例子，如果吧这个mcp server接入到ai中，允许llm使用的话，可以实现一系列浏览器自动化操作，像cursor等等以及支持了mcp协议的引入，让我们看看go中如何与外部llm集成，也就是利用eino框架，让mcp去代替原有的function calling的功能。

## eino引入

官方引入介绍：[Tool - MCP | CloudWeGo](https://www.cloudwego.io/zh/docs/eino/ecosystem_integration/tool/tool_mcp/)

eino原本实现类似langchain的功能相对来说就比较轻松，官方给出的mcp接入方法感觉像是把mcp的tools转换成了原本的function tool。

通过：

```go
import "github.com/cloudwego/eino-ext/components/tool/mcp"

tools, err := mcp.GetTools(ctx, &mcp.Config{Cli: cli})
```

然后再把tools加入Tool Node节点，就能跟以前的function call一样去使用mcp，也算是给出了一种简洁的接入方法吧

本文主要介绍mcp，就不再赘述eino的使用方式了

## 结尾

咕咕爽啊（不是），有时间再详细介绍一下eino框架吧，golang自己的langchain
