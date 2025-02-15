---
title: Redis学习之旅(1) Strings
description: 类型篇：字符串类型
date: 2025-02-15 02:00:00+0000
categories:
    - go
tags:
    - golang

---

## 前言

（其实是本系列的前言）

为什么又开新坑了：因为学学redis，顺便同时记录下cli和`go-redis`的操作

为什么老坑不更了（或者没更完）：~~不想写了~~ 没人看

说正事：

本系列会对每个命令同时讲解两方面：`redis-cli`的操作和`go-redis`的操作，尽量完成官网上文档对应命令的**100%** 代码覆盖率

本系列大部分内容参考：[官网文档：Develop with Redis | Docs](https://redis.io/docs/latest/develop/)

本文讲的是，redis广大类型中最常见的一个：`Strings`

## 前前言

（补充一下redis的介绍）

**Redis 是什么？**  
Redis（**Re**mote **Di**ctionary **S**erver）是一个开源的 **内存数据库**，它以超快的读写速度和灵活的数据结构著称。你可以把它想象成一个“超级加速器”，专门用来存储那些需要频繁访问的数据（比如网站热门内容、用户登录信息），或者用来解决高并发场景下的性能瓶颈。

**Redis 的三大特点**  

1. **内存存储**  
   数据直接放在内存中，读写速度极快（微秒级），远超传统硬盘数据库（如 MySQL）。

2. **丰富的数据结构**  
   除了简单的键值对（Key-Value），还支持：  
   
   - **字符串**（String）：缓存文本、数字  
   - **哈希表**（Hash）：存储对象（如用户信息）  
   - **列表**（List）：实现消息队列  
   - **集合**（Set）/ **有序集合**（Sorted Set）：去重、排行榜  
   - 更多高级结构如位图（Bitmaps）、地理坐标（GEO）等。

3. **持久化与高可用**  
   虽然数据在内存中，但 Redis 支持定期保存到硬盘（RDB 快照）或记录操作日志（AOF），防止断电丢失。还能通过主从复制、哨兵模式、集群模式保障服务稳定。

**典型使用场景**  

- **缓存层**：减轻数据库压力，加速网页加载。  
- **会话存储**：快速管理用户登录状态。  
- **实时排行榜**：游戏积分、热搜榜单。  
- **消息队列**：异步处理任务（如发送邮件）。  
- **秒杀系统**：应对瞬时高并发请求。

**为什么选择 Redis？**  

- **简单易用**：命令简洁（例如 `SET name "Alice"`、`GET name`）。  
- **跨语言支持**：Python、Java、Go 等主流语言均可调用。  
- **生态强大**：大量企业级扩展方案（集群、监控工具等）。

**小提示**  
Redis 虽强，但并非万能：  

- 内存成本较高，不适合存储海量冷数据。  
- 适合高速读写场景，复杂计算仍需传统数据库。

## String

**总而言之**，Redis就是一个KV数据库，你可以把它理解成golang中的`map[string]interface{}` , 我们类型篇就将着手于对于V - Value的类型的介绍。

Redis字符串可存储字节序列，包括文本、序列化对象及二进制数组。因此，字符串是可与Redis键关联的最基本数据类型。它们不仅常用于缓存场景，还支持更多高级功能，如实现计数器及执行位运算，展现出多样化的应用潜力。

由于Redis的键均为字符串类型，当值也采用字符串类型时，我们实际上是将一个字符串映射至另一个字符串。这种字符串数据类型适用于多种应用场景，例如缓存HTML片段或页面。

## SET && GET

SET看这个翻译应该也能明白大概的意思，就是一个设置，给定一个Key和一个Value来存储。

比如：`SET key value`就能存储，有点像map里`map[key] = value`

存储完成以后自然可以通过key去查询，用我们的GET指令`GET key`

下面的例子是用`redis-cli`去存储并查询一个自行车的名字

```
SET bike:name name
GET bike:name
```

这个例子中：`bike:name`就是key，`name`就是value，大家可以打开自己的`redis-cli`动手敲一敲

### go-redis example

```go
package main

import (
    "context"
    "fmt"

    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()
    client := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    client.Set(ctx, "bike:name", "name", 0)
    res, e := client.Get(ctx, "bike:name").Result()
    if e != nil {
        panic(e)
    }
    fmt.Println(res)
}
```

大家可能会好奇为什么`go-redis`的`Set`跟`redis-cli`的`SET`相比多了个参数，这就是我们接下来要说的`expire`

### Expire

也就是过期时间，redis的SET是支持设置过期时间的，而0也就代表着没有过期时间，即永不过期，那么过期时间的作用是什么呢，是规定该数据最多只能存放这么长时间。

redis-cli设置过期时间的方法是`SET key value EX [time]`，time是过期时间的长度，是一个数字，单位是秒。go-redis只需要将0修改为time类型的值即可，例如：`1*time.Second`

## MSET && MGET

这两个命令其实就是SET和GET的升级版，用于设置和获取多个数据，直接来看例子吧

### redis-cli

```
MSET bike:1 test1 bike:2 test bike:3 test
MGET bike:1 bike:2 bike:3
```

上述命令输出的结果是：

```
1) "test1"
2) "test"
3) "test"
```

### go-redis

```go
package main

import (
	"context"
	"fmt"

	"github.com/redis/go-redis/v9"
)

func main() {
	ctx := context.Background()
	client := redis.NewClient(&redis.Options{
		Addr: "localhost:6379",
	})
	client.MSet(ctx, "bike:1", "test1", "bike:2", "test2", "bike:3", "test3")
	res, e := client.MGet(ctx, "bike:1", "bike:2", "bike:3").Result()
	if e != nil {
		panic(e)
	}
	for _, i := range res {
		fmt.Println(i)
	}
}

```

## INCR && DECR

这两个命令把字符串值解析成数，并分别进行自增和自减，使用的方式也很简单，`INCR key`和`DECR key`即可

比如：

```
SET test 3
INCR test
GET test
DECR test
GET test
```

两次GET分别得到4和3，也就是自增一次，然后自减一次，有趣的是，这两个命令都是原子的，也就是说你可以任意地在并发里调用他们而不会出问题，这得益于redis的单线程机制

### go-redis example

```go
package main

import (
	"context"
	"fmt"

	"github.com/redis/go-redis/v9"
)

func main() {
	ctx := context.Background()
	client := redis.NewClient(&redis.Options{
		Addr: "localhost:6379",
	})
	client.Set(ctx, "test", 3, 0)
	client.Incr(ctx, "test")
	res, _ := client.Get(ctx, "test").Int()
	fmt.Println(res)
	client.Decr(ctx, "test")
	res, _ = client.Get(ctx, "test").Int()
	fmt.Println(res)
}

```

## INCRBY && DECRBY

与上述两个命令基本相同，只是多了指定增减的数目，比如

```
SET test 2
INCRBY test 3
GET test
DECRBY test 1
GET test
```

两次GET分别输出5和4

### go-redis example

```go
package main

import (
	"context"
	"fmt"

	"github.com/redis/go-redis/v9"
)

func main() {
	ctx := context.Background()
	client := redis.NewClient(&redis.Options{
		Addr: "localhost:6379",
	})
	client.Set(ctx, "test", 2, 0)
	client.IncrBy(ctx, "test", 3)
	res, _ := client.Get(ctx, "test").Int()
	fmt.Println(res)
	client.DecrBy(ctx, "test", 1)
	res, _ = client.Get(ctx, "test").Int()
	fmt.Println(res)
}

```

## String的限制

默认情况下，单个 Redis 字符串的最大大小为 512 MB。

## 回顾

今天说到这里差不多就把Strings常见的命令说完了，我们说了：

- SET && GET

- MSET && MGET

- INCR && DECR

- INCRBY && DECRBY

[更多Strings相关的命令](https://redis.io/docs/latest/commands/?group=string)
