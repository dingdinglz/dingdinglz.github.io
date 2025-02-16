---
title: Redis学习之旅(3) Sets
description: 类型篇：集合
date: 2025-02-16 03:00:00+0000
categories:
    - go
tags:
    - golang
    - redis
---

## Sets

Redis集合是一种由唯一字符串成员构成的无序集合，这个集合就很接近大家认识里的集合了，与c++ stl中的set类似，但是多支持了很多功能。注意：Sets和数组的差异在于Sets内的每个元素一定是互异的。

通过集合，你可以：

- 追踪唯一性数据（例如记录访问某篇博客文章的所有独立IP地址）

- 描述对象间关联关系（例如具有特定角色的全体用户集合）

- 执行常见的集合运算，包括交集、并集与差集操作。

## SADD

将一个元素添加到集合中，用法是：`SADD [集合名] 值`，注意，如果这个值与集合中之前存在的元素重复，将不会重复添加，添加多个元素是允许的，比如`SADD test 1 2 3 4 5 6`

### redis-cli

```
SADD test 1
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
    client.SAdd(ctx, "test", "1")
}
```

## SREM

删除集合中的一个元素，用法是：`SREM [集合名] 值`

删除多个元素也是允许的。

### redis-cli

```
SREM test 1
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
    client.SRem(ctx, "test", "1")
}
```

## SISMEMBER

判断元素在不在SET里，用法是：`SISMEMBER [集合名] 值`

### redis-cli

```
SISMEMBER test 1
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
    client := redis.NewClient(&redis.Options{})
    res, e := client.SIsMember(ctx, "test", "1").Result()
    if e != nil {
        panic(e)
    }
    fmt.Println(res)
}
```

## SCARD

获取集合中元素的个数，高中时大家应该就学过card(X) = 集合X的个数，就是这个CARD，记法就是set_card。用法是：`SCARD 集合名`

### redis-cli

```
SCARD test
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
    client := redis.NewClient(&redis.Options{})
    res, e := client.SCard(ctx, "test").Result()
    if e != nil {
        panic(e)
    }
    fmt.Println(res)
}
```

## SINTER

这是一个很有意思的命令，可以取两个集合的交集

比如，我有两个集合，test和test2，分别有1，2，3，4和1，3，4，5这些元素

`SINTER test test2`可以取得1，3，4也就是这两个集合的交集

### redis-cli

```
SINTER test test2
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
	client := redis.NewClient(&redis.Options{})
	res, e := client.SInter(ctx, "test", "test2").Result()
	if e != nil {
		panic(e)
	}
	for _, i := range res {
		fmt.Println(i)
	}
}

```

## SDIFF

这个命令用于求两个集合的差，也是一个比较有意思的命令，如果是`SDIFF setA setB`的话，给我的感觉像是`setA - SINTER(setA setB)`

### redis-cli

```
SDIFF test test2
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
	client := redis.NewClient(&redis.Options{})
	res, e := client.SDiff(ctx, "test", "test2").Result()
	if e != nil {
		panic(e)
	}
	for _, i := range res {
		fmt.Println(i)
	}
}

```

## SUNION

求两个集合的并集

### redis-cli

```
SUNION test test2
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
	client := redis.NewClient(&redis.Options{})
	res, e := client.SUnion(ctx, "test", "test2").Result()
	if e != nil {
		panic(e)
	}
	for _, i := range res {
		fmt.Println(i)
	}
}

```

## SMEMBERS

获取一个集合的全部成员

### redis-cli

```
SMEMBERS test
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
	client := redis.NewClient(&redis.Options{})
	res, e := client.SMembers(ctx, "test").Result()
	if e != nil {
		panic(e)
	}
	for _, i := range res {
		fmt.Println(i)
	}
}

```

## SRANDMEMBER

这也是个很有意思的功能，从集合中随机抽元素，用法是`SRANDMEMBER set名 个数`，个数是可选的参数，如果不填个数的话默认抽一个

这个功能感觉可以拿来做抽奖小程序啊（如果完全随机的话）

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
	res, e := client.SRandMemberN(ctx, "test", 2).Result()
	if e != nil {
		panic(e)
	}
	for _, i := range res {
		fmt.Println(i)
	}
}

```

## SPOP

这个命令用于随机从集合中抽取一个元素并把它删除，这种场景我有点难想象，假如有个不重复抽取的奖池（抽完为止，奖池里奖品数固定）就可以用这个命令抽取。

### redis-cli

```
SPOP test
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
	client := redis.NewClient(&redis.Options{})
	res, e := client.SPop(ctx, "test").Result()
	if e != nil {
		panic(e)
	}
	fmt.Println(res)
}

```

## 示例

我们就用SPOP来实现一个抽奖小程序吧

```go
package main

import (
	"context"
	"fmt"
	"strconv"

	"github.com/redis/go-redis/v9"
)

func main() {
	ctx := context.Background()
	client := redis.NewClient(&redis.Options{})
	for i := 1; i <= 10; i++ {
		// 添加十个奖品
		client.SAdd(ctx, "test", "prize"+strconv.Itoa(i))
	}
	for {
		res, e := client.SPop(ctx, "test").Result()
		if e != nil {
			if e == redis.Nil {
				fmt.Println("抽取完成！")
				break
			}
			panic(e)
		}
		fmt.Println("抽到了", res, "!")
	}
}

```

## 回顾

SET感觉可以当作数组来用，今天聊到这里讲了这些命令：

- SADD

- SREM

- SISMEMBER

- SCARD

- SINTER && SDIFF && SUNION

- SMEMBERS

- SRANDMEMBER

- SPOP

[更多SET相关的命令](https://redis.io/docs/latest/commands/?group=set)