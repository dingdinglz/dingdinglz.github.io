---
title: Redis学习之旅(2) Lists
description: 类型篇：列表
date: 2025-02-16 02:00:00+0000
categories:
    - go
tags:
    - golang
    - redis
---

## Lists

Redis中的列表（Lists）是由字符串值构成的链表结构。

相比于大家认知中的列表，我觉得这个格式更偏向于队列和栈，而不是数组

Redis的列表通常用于：

- 构建队列和栈

- 为后台 worker 系统构建队列管理

为什么这么说呢，首先对于Lists取出元素的操作大都是从一头取（如果看成一个双头队列的话），取出以后就删除该元素，这很显然就是典型的队列和栈的结构（大家可能也知道，cpp的stl中的栈是由deque实现的），那么下面介绍一下Lists相关的命令

## LPUSH && RPUSH

我建议在理解redis的lists结构时将它看成一个线性的结构，一条横铺的线，那么`LPUSH`就是从左边插入元素（Left Push），`RPUSH`就是从右边插入元素（Right Push），按这么来记，对于下文中讲的其他R和L开头的命令来说会更好理解一点。

LPUSH和RPUSH于SET的使用方法类似，形如：`LPUSH 列表名 要插入的值`或者`RPUSH 列表名 要插入的值`，下面我们再讲R或者L开头的命令时可能就不再重复阐述，只讲解R或者L的命令。

LPUSH会在List不存在的时候自动创建List，因而不要在意

需要注意的是，LPUSH可以依次插入多个元素，例如`LPUSH tasks 1 2 3 4 5`是正确的。

下面我们举个例子，比如创建一个任务列表`tasks`

### redis-cli

```
LPUSH tasks task1
LPUSH tasks task2
```

### go-redis

```go
package main

import (
    "context"

    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()
    client := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    client.LPush(ctx, "tasks", "task1")
    client.LPush(ctx, "tasks", "task2")
}
```

## LPOP && RPOP

继续沿用我们刚刚的思想，就是分别从左边和右边弹出一个元素，但是，这里的POP和我们c++的stl中队列的pop不同的是，它并不简简单单只是删除，而是附带了一个取回的操作。

比如我们`LPOP tasks`，就能得到task2，RPOP同理

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
    res, e := client.LPop(ctx, "tasks").Result()
    if e != nil {
        panic(e)
    }
    fmt.Println(res)
}
```

## LLEN

获取列表的长度，只需要`LLEN 列表名`即可，例如：`LLEN tasks`

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
    res, e := client.LLen(ctx, "tasks").Result()
    if e != nil {
        panic(e)
    }
    fmt.Println(res)
}
```

## LRANGE

从左边开始获取一个范围到右边的某个位置，range嘛，用法是：`LRANGE 列表名 左边坐标 右边坐标`，就可以获取到从[左边坐标，右边坐标]这个范围的所有元素，注意，这个坐标从左边开始数，且从零开始。（让我很奇怪的是，这个命令为什么没有R版，也就是说RRANGE是不存在的）

另外，右边坐标可以= -1，这时候代表从左边坐标一直取到最右边，类似于go的切片留空，和py、matlab的-1，其实这里的-1代表的是从另一头数第一个元素，也就是从右侧开始数第一个元素，同理，-2代表从另一头数第二个元素

### redis-cli

```
LPUSH test 1
LPUSH test 2
LPUSH test 3
LRANGE test 0 1
```

得到：

```
1) "3"
2) "2"
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
    client.LPush(ctx, "test", 1)
    client.LPush(ctx, "test", 2)
    client.LPush(ctx, "test", 3)
    res, e := client.LRange(ctx, "test", 0, 1).Result()
    if e != nil {
        panic(e)
    }
    for _, i := range res {
        fmt.Println(i)
    }
}
```

## 示例

下面让我们用go-redis利用List模拟一下消息队列

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "strconv"
    "sync"
    "time"

    "github.com/redis/go-redis/v9"
)

var waitGroup sync.WaitGroup

// 向消息队列中添加任意个待处理的任务
func AddTasks(ctx context.Context, client *redis.Client, num int) {
    for i := 1; i <= num; i++ {
        client.LPush(ctx, "tasks", "task"+strconv.Itoa(i))
    }
}

// 处理消息队列
func DealTasks(ctx context.Context, client *redis.Client, name string) {
    //先添加的任务先处理
    for {
        task, e := client.RPop(ctx, "tasks").Result()
        if e == redis.Nil {
            fmt.Println(name + "处理完成！")
            waitGroup.Done()
            break
        }
        fmt.Println(name + "处理" + task + "中")
        // 模拟处理耗时
        dealTime := rand.Intn(3001)
        time.Sleep(time.Duration(dealTime) * time.Millisecond)
    }
}

func main() {
    ctx := context.Background()
    client := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    waitGroup.Add(3)
    AddTasks(ctx, client, 100)
    for i := 1; i <= 3; i++ {
        // 开三个goroutine来处理
        go DealTasks(ctx, client, "处理通道"+strconv.Itoa(i))
    }
    waitGroup.Wait()
}
```

可以看到，得益于redis的单线程，我们虽然开了三个线程开同时处理消息列表中的任务，仍然可以保证先进来的任务先被处理。

朋友们可以自己修改一下进程数和任务数，并体会一下如何把这个简单的消息队列运用到实战中

## 限制

Redis 列表的最大长度为 2^32 - 1 （4,294,967,295） 个元素。

## 回顾

今天说到这里差不多把Lists的一些常用命令说完了，还附带了一个小小的应用例子，让我们来回顾一下今天学习了哪些命令

- LPUSH && RPUSH

- LPOP && RPOP

- LLEN

- LRANGE

[更多Lists相关的命令](https://redis.io/docs/latest/commands/?group=list)
