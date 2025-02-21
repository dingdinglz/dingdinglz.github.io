---

title: Redis学习之旅(4) Hash
description: 类型篇：哈希表
date: 2025-02-21 02:00:00+0000
categories:
    - go
tags:
    - golang
    - redis

---

## Hashes

Redis hashes是一种记录类型，结构为字段值对的集合。

你可以使用 hashes 来

- 表示基本对象

- 存储计数器的分组等

我感觉Hash有点类似于`map[string]interface{}`

hash可以存储这种类似的json

```json
{
    "name":"dinglz",
    "age":18
}
```

简单的来说，一个hash类型的值，存储的是key-value的结构

## HSET

设置一个字段的值，用法是`HSET hash名 字段名 字段值`

比如要存储一个人的信息，存储他的姓名和年龄，我们就可以

```
HSET person name dinglz
HSET person age 18
```

### go-redis example

```go
package main

import (
	"context"

	"github.com/redis/go-redis/v9"
)

func main() {
	ctx := context.Background()
	client := redis.NewClient(&redis.Options{})
	client.HSet(ctx, "person", "name", "dinglz")
	client.HSet(ctx, "person", "age", 18)
}

```

## HGET

获取某个字段的值，用法是`HGET hash名 字段名`

### redis-cli

```
HGET person name
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
	res, e := client.HGet(ctx, "person", "name").Result()
	if e != nil {
		panic(e)
	}
	fmt.Println(res)
}

```

## HINCRBY

与`INCRBY`类似，让hash的某个字段增加一个数值，用法是`HINCRBY hash名 字段名 增加的数值`

比如要模拟一个人长大了一岁

### redis-cli

```
HINCRBY person age 1
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
	client.HIncrBy(ctx, "person", "age", 1)
}

```

## HEXISTS

判断hash中是否存在某个字段，用法是`HEXISTS hash名 字段名`

### redis-cli

```
HEXISTS person name
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
	res, e := client.HExists(ctx, "person", "name").Result()
	if e != nil {
		panic(e)
	}
	fmt.Println(res)
}

```

## HKEYS

可以获取hash中的所有字段，用法是`HKEYS hash名`

### redis-cli

```
HKEYS person
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
	res, e := client.HKeys(ctx, "person").Result()
	if e != nil {
		panic(e)
	}
	for _, i := range res {
		fmt.Println(i)
	}
}

```

## 回顾

HASHES就是一个高性能，可持久化的map，今天聊到了这些命令：

- HSET && HGET

- HINCRBY

- HEXISTS

- HKEYS

[更多Hash相关的命令](https://redis.io/docs/latest/commands/?group=hash)