---

title: etcd的功能与应用
description: 什么是etcd，怎么将etcd集成到应用中
date: 2025-03-24 02:00:00+0000
categories:
    - go
    - github
tags:
    - golang
    - etcd

---

> 伟大的etcd构成了云原生的基石

## 什么是etcd

etcd 是由 CoreOS（现为 Red Hat 旗下）开发的分布式键值存储系统，采用 Go 语言编写。它通过 Raft 共识算法实现高可用性，专为分布式系统设计，提供以下核心能力：

- **强一致性**：保证所有节点数据状态完全一致

- **高可用性**：通过多节点集群自动故障转移

- **简单易用**：通过 HTTP API 提供 RESTful 接口

- **可扩展性**：支持水平扩展和动态集群管理

在分布式系统领域，etcd 以其优雅的设计和强大的功能，成为云原生时代的基础设施基石。它就像一个"分布式系统的心脏"，为 Kubernetes、Service Mesh 等系统提供核心的分布式数据管理能力。

## etcd的实际应用

- 为分布式系统提供配置设置中心，也就是键值对存储，有点类似于redis，下面我们会谈到它和redis的异同

- 实现配置的热更新，也就是利用etcd的watch功能监听键值的变化

- 作为分布式锁的一种实现形式

- 作为微服务中，服务发现与注册中心（得益于lease和watch）

下面，我们将针对这些实际应用，将golang代码实现结合来进行讲解

在此之前，先完成前置准备工作：

### 前置工作

1. 安装etcd：[Install | etcd](https://etcd.io/docs/v3.5/install/)

2. 本文go实现代码基于ectd官方client包：[etcd/client/v3 at main · etcd-io/etcd · GitHub](https://github.com/etcd-io/etcd/tree/main/client/v3)

Let's start

## 部署

etcd支持单机部署和集群式部署，集群式部署时，etcd保证了自己不同终端间的数据高度一致性和高度可用性，也就是说，与redis相比，etcd更加一致，同步速度更快，因而作为分布式配置同步是很好的选择

由于本文主要谈一谈etcd能帮我们实现哪些内容，就不在此详细说明如何部署etcd集群（让运维哥去头疼吧哈哈哈），想要了解详细内容可以参考：[How to Set Up a Demo etcd Cluster | etcd](https://etcd.io/docs/v3.5/tutorials/how-to-setup-cluster/)

## Golang连接etcd

首先安装前置工作中的etcd官方的client包，然后参考下述代码进行连接，即可得到一个etcd的client对象

```go
package main

import (
	"fmt"

	etcd "go.etcd.io/etcd/client/v3"
)

func main() {
	client, e := etcd.New(etcd.Config{
		Endpoints: []string{
			"localhost:2379",
		},
	})
	if e != nil {
		fmt.Println(e.Error())
		return
	}
```

## 键值对的读和写

etcd最基础的功能其实就是做一个kv存储，有点类似于redis，但是etcd并不具备redis那样丰富的数据类型，所以存储和读取时的结构只有string-string。更多拓展可以考虑用json格式去封装

### 写

```go
ctx := context.Background()
client.Put(ctx, "test", "test-value")
```

这样即给test设置了值

### 读

```go
res, e := client.Get(ctx, "test")
if e != nil {
	fmt.Println(e.Error())
	return
}
for _, item := range res.Kvs {
	fmt.Println(string(item.Key) + ":" + string(item.Value))
}
```

这样即可以取到key和value

大家可能有疑惑为什么res.Kvs是一个切片而不是一个元素，因为Get可以按前缀读

### 按前缀读

```go
res, e := client.Get(ctx, "test", etcd.WithPrefix())
if e != nil {
	fmt.Println(e.Error())
	return
}
for _, item := range res.Kvs {
	fmt.Println(string(item.Key) + ":" + string(item.Value))
}
```

可以看到区别仅仅在于`etcd.WithPrefix()`，这句的意思就是读所有前缀是test的键值对，有点类似于test*，我们如果新建一个test2和test3，用上述代码可以全部读出来

## 键值对的合租（lease）

所谓lease，可以理解成一个键的保质期，lease用于与键进行绑定，一个lease可以绑定多个键，当lease过期时，对应的键都会被删除。

与正常的过期时间不同的是，lease可以被续时或者直接销毁。

基于以上特点，让我们想想lease可以拿来做什么：

- 配合监听机制去做心跳包，也就是设置一个过期时间，让服务器端不断的去维护（延时）lease，如果服务器宕机了，与lease绑定的键会自动删除，也就说明了这台服务器对应的已经不可用

- 根据上面那点可以继续拓展，做微服务的服务注册，比如一个api对应一个ip，让服务器动态维护ip的更新，服务器寄了也就导致这个api失效

### 设置过期时间

```go
// 创建一个新的lease，过期时间是5s
lease, _ := client.Grant(ctx, 5)
// 将刚刚创建的lease与键绑定
client.Put(ctx, "test", "test-lease", etcd.WithLease(lease.ID))
```

大家可以试一下5s后再去取test这个键，这个时候test已经被删除

### 服务器维护心跳包

```go
// 创建一个新的lease，过期时间是5s
lease, _ := client.Grant(ctx, 5)
// 将刚刚创建的lease与键绑定
client.Put(ctx, "heart-ping", "live", etcd.WithLease(lease.ID))
for {
	// 留1s给网络因素
	time.Sleep(4 * time.Second)
	client.KeepAliveOnce(ctx, lease.ID)
}
```

此时，验证服务器是否存活只需要验证heart-ping是否存在即可

## 监听键值对（watch）

实时监听键值对的变化，根据这个功能可以做到配置的热更新等等

```go
// 监听test键
watcher := client.Watch(ctx, "test")
for event := range watcher {
	for _, item := range event.Events {
		if item.Type == etcd.EventTypeDelete {
			fmt.Println(string(item.Kv.Key) + " deleted")
		} else {
			fmt.Println(string(item.Kv.Key) + " changed:" + string(item.Kv.Value))
		}
	}
}
```

类似的，watch也接受`etcd.WithPrefix()`参数

## 分布式锁

既然redis可以维护一个分布式锁，那么我etcd当然也行，并且同步性更强，更易于拓展

### 分布式锁的原理

通过redis、etcd这些中间件，让一把锁可以被多个应用共用，实现原理也很简单，维护一个键的存在作为锁被占用，不存在作为未加锁的状态去实现，例如使用redis的`SETNX`

分布式锁想要实现的功能与本地的`sync.MXLock`这些基本类似，区别仅仅在于前者允许多个系统同时访问

### 实现

```go
// 开启一个事务
res, _ := client.Txn(ctx).If(etcd.Compare(etcd.CreateRevision("lock"), "=", "0")). // 如果lock不存在，即为可以加锁
											Then(
		etcd.OpPut("lock", "1"), // 创建lock占用锁
	).
	Commit()
if res.Succeeded {
	// 加锁成功
	// do someting ....
	// 解锁
	client.Delete(ctx, "lock")
} else {
	fmt.Println("加锁失败，占用中")
}
```

为了不形成死锁，也就是一直保持占用状态，可以通过设置ctx的timeout或者用刚刚说的lease去维护一个过期时间来处理

加锁失败的情况也不代表彻底不能使用，只是当前被占用，可以用循环尝试多次上锁等等操作

## 服务发现与治理

比如说，一个rpc服务，有个function，调用的ip地址有多个，通过心跳包实时维护他的地址，通过watch去动态更新调用的地址，如果你想要深入了解这方面，可以去阅读go-zero的相关部分的源码，它的默认实现就是利用etcd

## etcd与redis的异同

| **特性**     | **Redis**                                              | **etcd**                                                |
| ---------- | ------------------------------------------------------ | ------------------------------------------------------- |
| **核心功能**   | 高性能键值存储系统，支持多种数据结构（字符串、列表、哈希、集合、有序集合等）。                | 分布式一致性键值存储系统，专注于服务发现、配置管理、分布式协调。                        |
| **数据模型**   | 支持丰富的数据类型，如字符串、列表、哈希、集合、有序集合、位图、地理空间等。                 | 仅支持简单的键值对（字符串），但支持版本控制、租约（Lease）和目录结构（递归操作）。            |
| **一致性模型**  | 支持最终一致性（异步复制）或强一致性（通过 Redis Cluster 或多主模式）。            | 通过 Raft 协议保证强一致性，所有节点数据实时同步。                            |
| **性能**     | 高吞吐量和低延迟，适合高频读写操作（如缓存、队列）。                             | 为一致性牺牲部分性能，但能满足分布式系统协调需求。                               |
| **持久化**    | 支持 RDB（快照）和 AOF（追加日志）两种持久化方式。                          | 数据持久化基于 Raft 日志，支持 WAL（预写日志）和快照。                        |
| **适用场景**   | 缓存、队列、实时计数器、会话管理、消息中间件（如 Redis Pub/Sub）。               | 服务发现、配置中心、分布式锁、 leader 选举、分布式状态协调（如 Kubernetes）。        |
| **一致性协议**  | 不依赖特定协议，通过主从复制或 Cluster 实现分布式。                         | 基于 Raft 协议实现分布式一致性，确保所有节点数据一致。                          |
| **集群管理**   | 需要额外工具（如 Redis Cluster、Redis Sentinel）实现高可用。           | 内置集群管理，自动故障转移和节点发现。                                     |
| **API 接口** | 原生协议（Redis Protocol）或 RESTful API（通过 RedisJSON 等模块扩展）。 | 原生 HTTP/JSON API，支持 Watch 机制（实时监听键值变化）。                 |
| **生态系统**   | 丰富的客户端库（支持多种语言）、插件和扩展模块（如 RedisModule）。                | 提供官方客户端库（Go、Java、Python 等），专注于分布式协调场景。                  |
| **部署复杂度**  | 单机部署简单，集群模式需配置主从或 Cluster。                             | 需要集群部署（至少 3 个节点），但管理相对自动化。                              |
| **典型用途示例** | 缓存高频访问数据、实现消息队列（如 Celery）、实时计数器（如限流）。                  | 存储分布式系统的配置、服务注册与发现、协调分布式任务（如 etcd 作为 Kubernetes 的存储后端）。 |

## 写在最后

在咕咕了很久之后终于又更新了一篇（），etcd在目前谈啥都要聊聊分布式的年代（虽然或许其实不需要用分布式也能带得动呢，真的有那么多用户吗？）起着很重要的作用，etcd在这个领域也不是一家独大，它的“同行”有zookeeper等等，但显然etcd是其中格外出色的。

顺便一提，最近有种很忙又不知道在忙啥的感觉（），希望下次更新可以提上日程吧，尚且咕咕了一篇kmp和eino的内容，唉唉唉