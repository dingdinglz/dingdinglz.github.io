---

title: Redis学习之旅(5) Sorted sets
description: 类型篇：排序集
date: 2025-02-22 02:00:00+0000
categories:
    - go
tags:
    - golang
    - redis

---

## Sorted Sets

Redis有序集合是一种由唯一字符串（成员）构成的数据结构，其成员按关联分值排序。当多个字符串具有相同分值时，这些字符串将按字典顺序排列。其典型应用场景包括：

- 排行榜。例如，在大型在线游戏中，您可以通过有序集合轻松维护最高分的有序列表，实现高效的游戏积分排名系统。

- 速率限制器。特别是可以利用有序集合构建滑动窗口速率限制器，以防止过多的API请求。

sorted set与set的区别是，它与set一样都是只能存放互异的元素，但是它的每个元素与一个分数（score）绑定。

sorted set有点类似于cpp的stl中的优先队列，他会将所有元素按照score的大小进行排序

## ZADD

向sorted set中加入一个元素，用法是`ZADD 有序集合名 元素的分数 元素`

当我们要修改一个元素的分数时，可以直接再用一遍ZADD，会自动修改元素的分数并重新排序

### redis-cli

```
ZADD age 20 jack
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
	client.ZAdd(ctx, "age", redis.Z{
		Member: "Jack",
		Score:  20,
	})
}

```

## ZREM

从sorted set中删除一个元素，用法是`ZREM 有序集合名 元素`

### redis-cli

```
ZREM age jack
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
	client.ZRem(ctx, "age", "Jack")
}

```

## ZRANGE && ZREVRANGE

遍历sorted set，ZRANGE是以升序方式遍历，ZREVRANGE是以降序方式遍历

用法是`ZRANGE 排序集合名 开始下标 结束下标`

比如`ZRANGE age 0 -1`就是升序方式遍历整个集合（从小分数到大分数），`ZREVRANGE`则与之相反

### redis-cli

```
ZADD age 18 dinglz
ZADD age 20 jack
ZADD age 25 alice
ZRANGE age 0 -1
```

得到的结果是

```
1) "dinglz"
2) "jack"
3) "alice"
```

可以看到是从小到大遍历的，大家可以自己试一下`ZREVRANGE`

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
	res, e := client.ZRange(ctx, "age", 0, -1).Result()
	if e != nil {
		panic(e)
	}
	for _, i := range res {
		fmt.Println(i)
	}
}

```

## ZRANK && ZREVRANK

获取排序集合中一个元素从小到大的排名（REV从大到小），用法是`ZRANK 排序集合名 元素`，注意，获取到的排名从0开始

### redis-cli

```
ZRANK age dinglz
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
	res, e := client.ZRank(ctx, "age", "dinglz").Result()
	if e != nil {
		panic(e)
	}
	fmt.Println(res)
}

```

## ZRANGEBYSCORE && ZREVRANGEBYSCORE

与ZRANGE类似，但是ZRANGEBYSCORE限制了score的范围，用法是`ZRANGEBYSCORE 排序集合名 最小分数 最大分数`和`ZREVRANGEBYSCORE 排序集合名 最大分数 最小分数`

注意！！这两个命令的最小分数和最大分数位置是相反的

比如，我们要获取年龄从小到大，从最小到20的排序

### redis-cli

```
ZRANGEBYSCORE age -inf 20
```

`inf`指无穷大，`-inf`就是无穷小

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
	res, e := client.ZRangeByScore(ctx, "age", &redis.ZRangeBy{
		Min: "-inf",
		Max: "20",
	}).Result()
	if e != nil {
		panic(e)
	}
	for _, i := range res {
		fmt.Println(i)
	}
}

```

## 回顾

从上面一些命令就可以看出，sorted sets简直就是为了排行榜这种而生的，当你的服务想引入排行榜时，我们借助它就能轻易实现，今天聊到了这些命令：

- ZADD

- ZREM

- ZRANGE && ZREVRANGE

- ZRANK && ZREVRANK

- ZRANGEBYSCORE && ZREVRANGEBYSCORE