---

title: Redis学习之旅(7) Geospatial
description: 类型篇：地理空间
date: 2025-02-24 02:00:00+0000
categories:
    - go
tags:
    - golang
    - redis

---

## Geospatial

Redis地理空间索引可用于存储地理坐标并支持基于坐标的检索功能。这种数据结构适用于查找给定半径或边界框范围内的邻近位置点，在定位周边目标场景中具有重要应用价值。

其实我感觉这是个挺奇怪的类型，地理坐标似乎只对特定业务有用，其实就是一个pair<double,double>的set，附带了一些计算方法和筛选方法。

## GEOADD

将一个位置添加到地理空间里，用法是`GEOADD 地理空间名 经度 纬度 地点名`

### redis-cli

```
GEOADD test -122.27652 37.805186 station:1
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
	client.GeoAdd(ctx, "test", &redis.GeoLocation{
		Name:      "station:1",
		Longitude: -122.27652,
		Latitude:  37.805186,
	})
}

```

## GEODIST

可以获取两点间的距离，用法是`GEODIST 地理空间名 地点1 地点2`

### redis-cli

```
GEODIST test station:1 station:2 km
```

km可以替换成其他长度，例如m

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
	res, e := client.GeoDist(ctx, "test", "station:1", "station:2", "m").Result()
	if e != nil {
		panic(e)
	}
	fmt.Println(res)
}

```

## GEOPOS

获取地理空间中一个点的坐标，用法是`GEOPOS 地理空间名 地点名`

### redis-cli

```
GEOPOS test station:1
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
	res, e := client.GeoPos(ctx, "test", "station:1").Result()
	if e != nil {
		panic(e)
	}
	fmt.Println(res[0])
}

```

## GEOSEARCH

搜索一个圆或者一个长方形内的地理位置的个数，用法是`GEOSEARCH 地理空间名 <FROMMEMBER member | FROMLONLAT longitude latitude>
  <BYRADIUS radius <M | KM | FT | MI> | BYBOX width height <M | KM |
  FT | MI>>`

### redis-cli

```
GEOSEARCH test FROMLONLAT -122.2612767 37.7936847 BYRADIUS 5 km WITHDIST
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
	res, e := client.GeoSearch(ctx, "test", &redis.GeoSearchQuery{
		Longitude:  -122.2612767,
		Latitude:   37.7936847,
		Radius:     5,
		RadiusUnit: "km",
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

地理空间对于某些特定业务非常有用，例如地图中的定位，搜索等等，我们今天聊了这些命令：

- GEOADD

- GEODIST

- GEOPOS

- GEOSEARCH

[更多地理空间相关的命令](https://redis.io/docs/latest/commands/?group=geo)