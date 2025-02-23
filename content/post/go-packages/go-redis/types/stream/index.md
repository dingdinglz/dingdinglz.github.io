---

title: Redis学习之旅(6) Streams
description: 类型篇：流
date: 2025-02-23 02:00:00+0000
categories:
    - go
tags:
    - golang
    - redis

---

## Stream

Redis流是一种数据结构，其行为类似于仅追加日志，同时实现了多项突破常规仅追加日志局限性的操作。这些特性包括支持常量时间复杂度(O(1))的随机访问，以及实现诸如消费者组等复杂的消息消费策略。通过流结构，开发者能够实时记录并同步分发事件。其典型应用场景包括：

- 事件溯源（例如跟踪用户操作、点击等行为）

- 传感器监测（例如来自现场设备的读数数据）

- 通知功能（例如为每位用户的通知记录单独创建独立事件流存储）

流是一个非常强大的类型，因此它的特性也就非常复杂，首先流可以理解成一个列表，但是其中的每个元素是带有先后顺序的一个Hash表，流与之前提到的List相比，最突出的一个特性是监听，通过消费者组，可以去实时地收到流中传来的最新的消息（或者未处理的消息）

因此，流比List更适合实现消息队列

## XADD

XADD用于向流中添加一个元素，注意，这个元素是一个hash表的形式，所以它类似于`map[string]interface{}`，XADD的用法是`XADD 流名 ID key value ...`

XADD会返回一个ID对应这个元素，XADD中的id你即可以手动指定（需要注意的是，你添加的元素一定要比上一次添加的元素的ID新），或者用*让redis自动帮你生成，redis生成的id是和时间相绑定的

比如我们要往person流中添加一个人，这个人具有name、age这两个字段

### redis-cli

```
XADD person * name dinglz age 18
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
	client := redis.NewClient(&redis.Options{})
	client.XAdd(ctx, &redis.XAddArgs{
		Stream: "person",
		ID:     "*",
		Values: map[string]interface{}{
			"name": "dinglz",
			"age":  18,
		},
	})
}

```

## XDEL

删除流中的一个元素，用法是`XDEL 流名 ID`

### redis-cli

```
XDEL person ID
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
	client := redis.NewClient(&redis.Options{})
	client.XDel(ctx, "person", "id")
}

```



## XRANGE

按范围查询流的内容，用法是`XRANGE 流名 开始 结束 [COUNT 获取个数]`

[]内是可选内容，开始id和结束id有一个特殊写法，就是`-`和`+`，前者代表最老一个流内容，后者代表最新一个流内容，也就是说，要获取刚刚流里所有内容，我们就可以`XRANGE person - +`

如果我们想获取最初加入的三个元素，我们就可以用COUNT限定最多取到的元素个数，可以这么写：`XRANGE person - + COUNT 3`

如果我们一直一个ID，想获取它的内容，我们就可以`XRANGE person id id`

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
	client := redis.NewClient(&redis.Options{})
	res, e := client.XRange(ctx, "person", "-", "+").Result()
	if e != nil {
		panic(e)
	}
	for _, i := range res {
		fmt.Println(i)
	}
}

```

## XGROUP CREATE

新建一个监听组，用法是`XGROUP CREATE 流名 开始监听的id`, 这个id可以填成$，表示监听从新建以后新增的id。

配合下面要说的XREADGROUP命令可以完成对消息的监听

## XREADGROUP

从监听的流里获取新消息，用法是`XREADGROUP GROUP 监听组名 监听者 COUNT 最多获取消息数 STREAMS 流名 >`

最后一个>表示获取从来没有被其他监听者听到过的消息，也就保证了消息不会重复分发

下面我们来添加三个人，然后尝试获取两次次

### redis-cli

```
XGROUP CREATE person test $
XADD person * name dinglz age 18
XADD person * name jack age 20
XADD person * name alice age 22
XREADGROUP GROUP test listener COUNT 1 STREAMS person >
XREADGROUP GROUP test listener COUNT 1 STREAMS person >
```

可以看到是从新到老获取

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
	client := redis.NewClient(&redis.Options{})
	client.XGroupCreateMkStream(ctx, "person", "test", "$")
	client.XAdd(ctx, &redis.XAddArgs{
		Stream: "person",
		ID:     "*",
		Values: map[string]interface{}{
			"name": "dinglz",
			"age":  18,
		},
	})
	client.XAdd(ctx, &redis.XAddArgs{
		Stream: "person",
		ID:     "*",
		Values: map[string]interface{}{
			"name": "jack",
			"age":  20,
		},
	})
	client.XAdd(ctx, &redis.XAddArgs{
		Stream: "person",
		ID:     "*",
		Values: map[string]interface{}{
			"name": "alice",
			"age":  20,
		},
	})
	res, e := client.XReadGroup(ctx, &redis.XReadGroupArgs{
		Group:    "test",
		Consumer: "listener",
		Streams:  []string{"person", ">"},
		Count:    1,
	}).Result()
	if e != nil {
		panic(e)
	}
	fmt.Println(res[0].Messages[0].Values)
}

```

## XACK && XPENDING

XACK和XPENDING是一组命令，XPENDING用于获取被获取，但没被处理的流里的数据，XACK用于标记该数据已被处理。

XPENDING的用法是`XPENDING 流名 监听组名`，XACK的用法是`XACK 流名 监听组名 ID`

详细的用法直接参考用例吧

## 用例

这里我们用stream实现一个更完善的消息队列，实现了以下功能

- 三个workers去认领任务

- 三个workers一共最多拥有十个处理任务的goroutine

- 每3s做一次剩余任务量的通报

```go
package main

import (
	"context"
	"fmt"
	"strconv"
	"sync"
	"time"

	"math/rand"

	"github.com/redis/go-redis/v9"
)

var waitGroup sync.WaitGroup
var processLock sync.Mutex

func taskMaker(ctx context.Context, client *redis.Client, counts int) {
	for i := 1; i <= counts; i++ {
		client.XAdd(ctx, &redis.XAddArgs{
			Stream: "tasks",
			Values: map[string]interface{}{
				"name": "task" + strconv.Itoa(i),
			},
		})
	}
}

func workerMaker(ctx context.Context, client *redis.Client, name string) {
	for {
		resW, _ := client.XReadGroup(ctx, &redis.XReadGroupArgs{
			Group:    "worker",
			Consumer: name,
			Count:    1,
			Streams:  []string{"tasks", ">"},
			Block:    1 * time.Second,
		}).Result()
		if len(resW) == 0 {
			fmt.Println(name + "认领工作完成")
			break
		}
		go func() {
			for {
				processLock.Lock()
				res, _ := client.Get(ctx, "worker-count").Int()
				if res < 10 {
					client.Incr(ctx, "worker-count")
					go func() {
						max := int64(3 * time.Second)
						ns := rand.Int63n(max)
						time.Sleep(time.Duration(ns))
						client.XAck(ctx, "tasks", "worker", resW[0].Messages[0].ID)
						client.XDel(ctx, "tasks", resW[0].Messages[0].ID)
						fmt.Println(resW[0].Messages[0].Values["name"], "由", name, "完成")
						client.Decr(ctx, "worker-count")
					}()
					processLock.Unlock()
					break
				}
				processLock.Unlock()
			}
		}()
	}
}

func main() {
	ctx := context.Background()
	client := redis.NewClient(&redis.Options{})
	client.Set(ctx, "worker-count", 0, 0)
	go taskMaker(ctx, client, 100)
	client.XGroupCreateMkStream(ctx, "tasks", "worker", "$")
	waitGroup.Add(1)
	for i := 1; i <= 3; i++ {
		go workerMaker(ctx, client, "worker"+strconv.Itoa(i))
	}
	go func() {
		for {
			time.Sleep(3 * time.Second)
			res, _ := client.XPending(ctx, "tasks", "worker").Result()
			resText := ""
			for name, last := range res.Consumers {
				resText += name + ":" + strconv.FormatInt(last, 10) + " "
			}
			fmt.Println(resText)
			if res.Count == 0 {
				waitGroup.Done()
			}
		}
	}()
	waitGroup.Wait()
}

```

## 回顾

Stream非常强大，从我们上面实现的消费队列就可以看出，大家可以详细研究一下Stream相关的，我们未提到的其他命令，今天聊到了这些命令：

- XADD

- XDEL

- XRANGE

- XGROUP CREATE

- XREADGROUP

- XACK && XPENDING

[更多stream相关的命令](https://redis.io/docs/latest/commands/?group=stream)