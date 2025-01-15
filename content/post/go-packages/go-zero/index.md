---
title: Golang微服务实战
description: 基于go-zero框架的微服务架构初实战
date: 2025-01-15 02:00:00+0000
categories:
    - go
tags:
    - golang
---

## 概述

### 微服务概述

近几年**微服务**的概念非常火，各大大厂也都已经在很多项目上采用的微服务式的设计，相比于直接用中间件做负载均衡的设计，微服务的优点在于它的模块式开发，支持热更新，对分布式更加友好。

对于一家大公司而言，模块式的开发，以及保持产品的稳定性及其重要，前者可以通过微服务设计，后者可以通过在分布式架构下，找个用户量小的时间段分批次更新来做到。

通常来说，微服务架构由很多微服务（负责实现不同的功能）和一个中心网关来实现，中心网关负责做负载均衡和服务治理，以及其他的统一处理请求。每个单一的微服务提供不同的功能。

大家较为熟知的rpc就是中心网关与微服务直接沟通的方式之一。

那么，想要实现一个微服务架构的应用要考虑的点非常之多，包括服务治理，负载均衡，熔断，主从同步等等内容。我也因此很长时间对于微服务的了解只停留在概念阶段。

近期参与到知易项目的开发中，便决定采用微服务架构，在一次次尝试中，发现了本文要介绍的非常好用，让开发者专注于逻辑的微服务框架**go-zero**

### go-zero概述

[go-zero]([https://go-zero.dev/](https://go-zero.dev/))是一个集成了各种工程实践的 web 和 rpc 框架。通过弹性设计保障了大并发服务端的稳定性，经受了充分的实战检验。

也就是说，你能想到要实现的所有功能，已经完全被内置在了go-zero这个框架中。

### 本文实战内容概述

我们用书店服务做示例，并且实现其中的增加书目和检查价格这两个功能，来模拟实际项目开发的流程。

那么，Let's go!!

## 环境准备

### 安装golang

上官网或者镜像站下载即可，别忘记配置go package的代理

可用的一个代理：[七牛云](https://goproxy.cn/)

### 安装mysql

我们通过mysql作为数据库来存储图书。

我推荐直接使用docker的mysql镜像来启动mysql，便于管理。

也可以在本机安装mysql，安利下之前我开发过的一款mysql管理工具：[MysqlManager: 可视化管理mysql，新建库，删除库，更改密码，初始化mysql，启动，关闭等日常操作。](https://gitee.com/dinglz/mysql-manager)  

### 安装etcd

我们用etcd作为服务发现中心，etcd是一款go开发的高性能的配置中心

[Releases · etcd-io/etcd · GitHub](https://github.com/etcd-io/etcd/releases) 下载并运行即可

### 安装protoc-gen-go

rpc协议的代码生成器，用下面的代码安装即可

```shell
 go get -u github.com/golang/protobuf/protoc-gen-go@v1.3.2
```

### 安装goctl

goctl是go-zero提供的一款代码生成工具，用于生成go-zero框架的代码，加速开发速度

```shell
go get -u github.com/zeromicro/go-zero/tools/goctl@latest
```

### 创建目录

下面创建一个目录`bookstore`

在目录中执行go mod init命令初始化项目

```shell
go mod init bookstore
```

## 编写中心网关

在上一步中我们安装了goctl，我们可以通过生成并编写api文件，来让goctl帮我们生成代码，迅速的把微服务的框架大家起来。

### 创建api文件

首先，创建api文件夹并进入，在文件夹下执行

```shell
goctl api -o bookstore.api
```

可以看到目录下已经新建了bookstore.api文件。

编辑它，我们把该项目的api设计为如下

```go
type (
	addReq {
		book string `form:"book"`
		price int64 `form:"price"`
	}
	
	addResp {
		ok bool `json:"ok"`
	}
)

type (
	checkReq {
		book string `form:"book"`
	}
	
	checkResp {
		found bool `json:"found"`
		price int64 `json:"price"`
	}
)

service bookstore-api {
	@handler AddHandler
	get /add (addReq) returns (addResp)
	
	@handler CheckHandler
	get /check (checkReq) returns (checkResp)
}
```

type用法和go一致，service用来定义get/post/head/delete等api请求，解释如下：

- `service bookstore-api {`这一行定义了service名字
- `@handler`定义了服务端handler名字
- `get /add(addReq) returns(addResp)`定义了get方法的路由、请求参数、返回参数等

api文件的具体语法可以前往官网查看。

[点我，点我查看具体语法](https://go-zero.dev/docs/tutorials)

然后我们就可以使用goctl为我们一键式生成代码

### 生成网关代码

```shell
goctl api go -api bookstore.api -dir .
```

生成的文件结构如下：

```
api
├── bookstore.api                  // api定义
├── bookstore.go                   // main入口定义
├── etc
│   └── bookstore-api.yaml         // 配置文件
└── internal
    ├── config
    │   └── config.go              // 定义配置
    ├── handler
    │   ├── addhandler.go          // 实现addHandler
    │   ├── checkhandler.go        // 实现checkHandler
    │   └── routes.go              // 定义路由处理
    ├── logic
    │   ├── addlogic.go            // 实现AddLogic
    │   └── checklogic.go          // 实现CheckLogic
    ├── svc
    │   └── servicecontext.go      // 定义ServiceContext
    └── types
        └── types.go               // 定义请求、返回结构体
```

此时网关便是符合我们api文件中声明的结构。

你可以直接在api文件夹`go build`来生成网关的可执行文件

> 如果遇到依赖相关的报错，记得`go mod tidy`更新下依赖哦

### 总结

目前的这个中心网关并没有实现对应的功能，只是一个空壳子，下面让我们分别完成add和check这部分的微服务。

## 编写微服务

### add rpc服务

在bookstore下创建rpc目录。

在rpc目录中分别创建add和check的文件夹。

首先我们来编写add的服务声明，进入add文件夹中

通过命令生成rpc声明的文件模版

```shell
goctl rpc template -o add.proto
```

修改文件如下

```protobuf
syntax = "proto3";

package add;

option go_package = "./add";

message addReq {
    string book = 1;
    int64 price = 2;
}

message addResp {
    bool ok = 1;
}

service adder {
    rpc add(addReq) returns(addResp);
}
```

该文件是用来为grpc定义微服务传输的格式和相关信息。

通过下面的命令生成rpc的代码

```shell
goctl rpc protoc add.proto --go_out=. --go-grpc_out=. --zrpc_out=.
```

文件结构如下：

```
rpc/add
├── add                   // pb.go
│   ├── add.pb.go
│   └── add_grpc.pb.go
├── add.go                // main函数入口
├── add.proto             // proto源文件
├── adder                 // rpc client call entry
│   └── adder.go
├── etc                   // yaml配置文件
│   └── add.yaml
└── internal              
    ├── config            // yaml配置文件对应的结构体定义
    │   └── config.go
    ├── logic             // 业务逻辑
    │   └── addlogic.go
    ├── server            // rpc server
    │   └── adderserver.go
    └── svc               // 资源依赖
        └── servicecontext.go
```

此时的add rpc已经编写完成，同样是可以`go build`并运行的

### check rpc服务

与add rpc服务想似

```shell
goctl rpc template -o check.proto
```

修改check.proto如下

```protobuf
syntax = "proto3";

package check;

option go_package = "./check";

message checkReq {
    string book = 1;
}

message checkResp {
    bool found = 1;
    int64 price = 2;
}

service checker {
    rpc check(checkReq) returns(checkResp);
}
```

生成代码：

```shell
goctl rpc protoc check.proto --go_out=. --go-grpc_out=. --zrpc_out=.
```

`etc/check.yaml`文件里可以修改侦听端口等配置

需要修改`etc/check.yaml`的端口为`8081`，因为`8080`已经被`add`服务使用了

## 中心网关对接微服务

修改配置文件`api/etc/bookstorrer-api.yaml`，增加如下内容

```yaml
Add:
  Etcd:
    Hosts:
      - localhost:2379
    Key: add.rpc
Check:
  Etcd:
    Hosts:
      - localhost:2379
    Key: check.rpc
```

通过etcd自动去发现可用的add/check服务

修改`api/internal/config/config.go`如下，增加add/check服务依赖

```go
type Config struct {
    rest.RestConf
    Add   zrpc.RpcClientConf     // 手动代码
    Check zrpc.RpcClientConf     // 手动代码
}
```

修改`api/internal/svc/servicecontext.go`，如下：

```go
type ServiceContext struct {
    Config  config.Config
    Adder   adder.Adder          // 手动代码
    Checker checker.Checker      // 手动代码
}

func NewServiceContext(c config.Config) *ServiceContext {
    return &ServiceContext{
        Config:  c,
        Adder:   adder.NewAdder(zrpc.MustNewClient(c.Add)),         // 手动代码
        Checker: checker.NewChecker(zrpc.MustNewClient(c.Check)),   // 手动代码
    }
}
```

修改`api/internal/logic/addlogic.go`里的`Add`方法，如下：

```go
func (l *AddLogic) Add(req *types.AddReq) (resp *types.AddResp, err error) {
	// 手动代码开始
	r, err := l.svcCtx.Adder.Add(l.ctx, &adder.AddReq{
		Book:  req.Book,
		Price: req.Price,
	})
	if err != nil {
		return nil, err
	}

	return &types.AddResp{
		Ok: r.Ok,
	}, nil
	// 手动代码结束
}
```

通过调用`adder`的`Add`方法实现添加图书到bookstore系统

修改`api/internal/logic/checklogic.go`里的`Check`方法，如下：

```go
func (l *CheckLogic) Check(req *types.CheckReq) (resp *types.CheckResp,err error) {
	// 手动代码开始
	r, err := l.svcCtx.Checker.Check(l.ctx, &checker.CheckReq{
		Book: req.Book,
	})
	if err != nil {
		logx.Error(err)
		return &types.CheckResp{}, err
	}

	return &types.CheckResp{
		Found: r.Found,
		Price: r.Price,
	}, nil
	// 手动代码结束
}
```

通过调用`checker`的`Check`方法实现从bookstore系统中查询图书的价格

## 定义数据库表结构，并生成CRUD代码

好的，做到目前这一步，我们已经完成了中心网关的全部内容，下面我们就需要挨个实现add、check两个服务的具体功能。

数据库操作无疑是主要的。

让我们来定义数据库表结构，并生成CRUD代码

bookstore下创建`rpc/model`目录

### 定义表结构

在rpc/model目录下编写创建book表的sql文件`book.sql`，如下：

```sql
CREATE TABLE `book`
(
  `book` varchar(255) NOT NULL COMMENT 'book name',
  `price` int NOT NULL COMMENT 'book price',
  PRIMARY KEY(`book`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

在mysql数据库中创建一个库，并运行这个sql文件建立表`book`

### 通过goctl命令生成CRUD代码

rpc/model目录下执行该命令

```shell
goctl model mysql ddl -c -src book.sql -dir .
```

生成后的文件结构如下：

```
rpc/model
├── book.sql
├── bookstoremodel.go     // CRUD+cache代码
└── vars.go               // 定义常量和变量
```

amazing，curd的结构已经全部生成，我们只要调用即可。

## rpc中处理数据库

修改`rpc/add/etc/add.yaml`和`rpc/check/etc/check.yaml`，增加如下内容：

```yaml
DataSource: root:@tcp(localhost:3306)/gozero 
# mysql链接地址，满足 $user:$password@tcp($ip:$port)/$db?$queries 格式即可
```

修改`rpc/add/internal/config/config.go`和`rpc/check/internal/config/config.go`，如下：

```go
type Config struct {
    zrpc.RpcServerConf
    DataSource string             // 手动代码
}
```

增加了mysql配置

修改`rpc/add/internal/svc/servicecontext.go`和`rpc/check/internal/svc/servicecontext.go`，如下：

```go
type ServiceContext struct {
    c     config.Config
    Model model.BookModel   // 手动代码
}

func NewServiceContext(c config.Config) *ServiceContext {
    return &ServiceContext{
        c:             c,
        Model: model.NewBookModel(sqlx.NewMysql(c.DataSource), c.Cache), // 手动代码
    }
}
```

修改`rpc/add/internal/logic/addlogic.go`，如下：

```go
func (l *AddLogic) Add(in *add.AddReq) (*add.AddResp, error) {
    // 手动代码开始
    _, err := l.svcCtx.Model.Insert(l.ctx,&model.Book{
        Book:  in.Book,
        Price: in.Price,
    })
    if err != nil {
        return nil, err
    }

    return &add.AddResp{
        Ok: true,
    }, nil
    // 手动代码结束
}
```

修改`rpc/check/internal/logic/checklogic.go`，如下：

```go
func (l *CheckLogic) Check(in *check.CheckReq) (*check.CheckResp, error) {
    // 手动代码开始
    resp, err := l.svcCtx.Model.FindOne(l.ctx,in.Book)
    if err != nil {
        return nil,err
    }

    return &check.CheckResp{
        Found: true,
        Price: resp.Price,
    }, nil
    // 手动代码结束
}
```

这几步是在调用数据库存储数据，所见即所得。

## 测试

我们分别在add，check，api文件夹中运行`go build`

生成了add.exe，check.exe，api.exe

首先运行etcd，启动mysql

再依次运行add，check和api（add和check运行顺序无所谓）即可启动服务

让我们用curl调用测试一下

### add

```shell
curl -i "http://localhost:8888/add?book=go-zero&price=10"
```

返回

```http
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 03 Sep 2020 09:42:13 GMT
Content-Length: 11

{"ok":true}
```

成功添加了一本书

### check

```shell
curl -i "http://localhost:8888/check?book=go-zero"
```

返回如下

```http
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 03 Sep 2020 09:47:34 GMT
Content-Length: 25

{"found":true,"price":10}
```

成功查询到了这本书！

## 本文所做项目的完整代码

[zero-examples/bookstore at main · zeromicro/zero-examples · GitHub](https://github.com/zeromicro/zero-examples/tree/main/bookstore)

## 反思与总结

本文所做的事情其实仅仅是搭建了一个微服务的体系，然后实现了两个非常简单的功能

真实的项目中需要完成完整的逻辑，并完成其他功能，比如redis缓存，限流器，鉴权等等，这些在go-zero中都为我们提供了现成的组件，不得不说，go-zero是一个强大的框架，同时，由于被众多公司用于真实项目，稳定性与安全性也是可以保障的。

## 参考资料

[zero-doc/docs/zero/bookstore.md at main · zeromicro/zero-doc · GitHub](https://github.com/zeromicro/zero-doc/blob/main/docs/zero/bookstore.md)

[Go-zero](https://go-zero.dev/)


