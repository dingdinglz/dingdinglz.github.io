---
title: 如何构建一个高效的网站后端应用
description: 一个网站后端或许并不是仅仅启动一个server就结束了，让我们用cobra和pterm走进结构、高效化的网站后端
date: 2025-05-13 02:00:00+0000
categories:
    - go
tags:
    - golang
---

## 前言

一个网站后端的核心功能当然是一个http server，但除了server之外还有其他需要设计的部分，比如生成配置文件啊，第一次使用引导安装啊，重新初始化，迁移数据，导出数据等等，直接运行应用后就启动一个http server并不是最优秀的选择。

因此，本篇blog将带领大家利用[Cobra](https://cobra.dev/)（Cobra 是一个用于创建强大的现代 CLI 应用程序的库。）和[Pterm](https://pterm.sh/)（PTerm 是一个现代的 Go 模块，可轻松美化控制台输出）来构建应用

我们需要实现的效果如下（应用名是test）：

- `test version`显示版本号
- `test server`启动服务器

## 命令框架

cobra的使用方法请大家自行参阅文档～

首先，我们构建基础命令

``` go
rootCmd := &cobra.Command{
	Use:   "test",
	Short: "一个测试应用",
	Long:  "一个测试应用，用来演示",
}
```

Version命令

``` go
versionCmd := &cobra.Command{
	Use:   "version",
	Short: "版本号",
	Long:  `获取test的版本信息`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("test v0.1 beta")
	},
}
```

Server命令

``` go
serverCmd := &cobra.Command{
	Use:   "server",
	Short: "启动服务器",
	Long:  "启动test的服务器服务",
	Run: func(cmd *cobra.Command, args []string) {
		// 启动服务器...
	},
}
```

添加到基础命令

``` go
rootCmd.AddCommand(versionCmd)
rootCmd.AddCommand(serverCmd)
```

解析

``` go
rootCmd.Execute()
```

当前完整代码

``` go
package main

import (
	"fmt"

	"github.com/spf13/cobra"
)

func main() {
	rootCmd := &cobra.Command{
		Use:   "test",
		Short: "一个测试应用",
		Long:  "一个测试应用，用来演示",
	}
	versionCmd := &cobra.Command{
		Use:   "version",
		Short: "版本号",
		Long:  `获取test的版本信息`,
		Run: func(cmd *cobra.Command, args []string) {
			fmt.Println("test v0.1 beta")
		},
	}
	serverCmd := &cobra.Command{
		Use:   "server",
		Short: "启动服务器",
		Long:  "启动test的服务器服务",
		Run: func(cmd *cobra.Command, args []string) {
			// 启动服务器...
		},
	}
	rootCmd.AddCommand(versionCmd)
	rootCmd.AddCommand(serverCmd)
	rootCmd.Execute()
}

```

编译并运行`./test`

```
一个测试应用，用来演示

Usage:
  test [command]

Available Commands:
  completion  Generate the autocompletion script for the specified shell
  help        Help about any command
  server      启动服务器
  version     版本号

Flags:
  -h, --help   help for test

Use "test [command] --help" for more information about a command.
```

可以看到，基础的命令框架已经搭建完成，我们可以通过`./test help`，`./test version`来测试一下，那么，我们要添加额外的功能只需要添加到子命令中即可，`./test server`则是启动主服务的方式

## 美化输出

后端的信息展示和日志等也是非常重要的，可以帮助我们进行溯源等等，因此我们引入`pterm`来美化输出，比如version命令我们可以先用大字美化。

``` go
pterm.DefaultBigText.WithLetters(putils.LettersFromString("test")).Render()
fmt.Println("test v0.1 beta")
```

又或者是将日志替换成pterm的logger

``` go
logger := pterm.DefaultLogger.WithLevel(pterm.LogLevelTrace)
logger.WithCaller().Debug("我是Debug")
logger.Info("你好呀")
logger.Debug("我是Debug")
```

更多妙用如进度条，选择器，表格等等命令行美化请查找pterm文档，本文仅提供部分思路

## 后记

本文介绍了构建后端服务器的结构，将单一服务器启动变成支持多功能，大家在开发自己的应用之前就要考虑好架构的设计，功能的拓展，随着时间的堆积才不会让代码越来越臃肿。