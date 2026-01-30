---
title: 深入理解 Go 1.23 iter 包
description: iter包的使用和原理
date: 2026-01-30 02:00:00+0000
categories:
    - go
tags:
    - golang
---

## 概述

Go 语言在 **1.23 版本**中正式引入了 `iter` 包，这是 Go 语言迭代器机制的一个重大变革。它配合语言层面的新特性 **Range Over Function**（函数迭代），让开发者能够以统一、标准的方式自定义迭代逻辑。

过去我们只能对 slice、map、channel 进行 `for-range` 遍历，现在任何符合 `iter.Seq` 或 `iter.Seq2` 签名的函数都可以直接被 `for-range` 使用。这让 Go 在保持简洁的同时，拥有了类似 Python 生成器的能力，但实现上更加轻量。

**本文内容紧跟 Go 1.23 官方发布，建议配合官方文档食用：https://pkg.go.dev/iter**

但是！我在网上找到的所有教程讲的都极其晦涩难懂，于是我写了下面的内容希望能帮助大家理解。

---

## 核心概念：什么是 `iter.Seq`？

`iter` 包定义了两个核心类型，它们本质上是**函数类型**。

### `iter.Seq[V]` (单值迭代器)

```go
type Seq[V any] func(yield func(V) bool)
```

- **用途**：用于每次迭代产出一个值（例如遍历切片的值）
- **工作原理**：
  - `Seq` 是一个高阶函数，它接受一个名为 `yield` 的回调函数
  - 编写迭代器时，调用 `yield(value)` 把数据"推"给调用者
  - 如果 `yield` 返回 `false`，表示调用者希望停止迭代（比如循环里遇到了 `break`），此时迭代器函数应该立即返回

### `iter.Seq2[K, V]` (双值迭代器)

```go
type Seq2[K, V any] func(yield func(K, V) bool)
```

和上面很类似

- **用途**：用于每次迭代产出两个值（例如遍历 map 的 key 和 value，或者 slice 的 index 和 value）

---

## 基础用法：自定义迭代器

### 示例 1：创建一个简单的数字生成器 (`iter.Seq`)

假设我们需要一个生成器，生成从 0 到 n-1 的整数：

```go
package main

import (
	"fmt"
	"iter"
)

// GenNumbers 返回一个迭代器，生成 [0, n)
func GenNumbers(n int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := 0; i < n; i++ {
			// 核心逻辑：
			// 1. 调用 yield(i) 发送数据
			// 2. 检查返回值。如果是 false，说明外部循环触发了 break，必须停止
			if !yield(i) {
				return
			}
		}
	}
}

func main() {
	// 直接使用 for-range 遍历函数
	for num := range GenNumbers(5) {
		fmt.Println(num) // 输出 0, 1, 2, 3, 4
		if num == 3 {
			break // 这里触发 break，yield 会返回 false，让 GenNumbers 退出
		}
	}
}
```

### 示例 2：遍历自定义数据结构 (`iter.Seq2`)

假设我们有一个自定义的 `MyList` 结构，想要遍历其中的元素：

```go
package main

import (
	"fmt"
	"iter"
)

type MyList struct {
	items []string
}

// All 返回索引和值的迭代器
func (l *MyList) All() iter.Seq2[int, string] {
	return func(yield func(int, string) bool) {
		for i, v := range l.items {
			// 发送两个值：索引和内容
			if !yield(i, v) {
				return
			}
		}
	}
}

func main() {
	list := MyList{items: []string{"Go", "Rust", "Python"}}

	// 像遍历 map 一样遍历自定义结构
	for i, v := range list.All() {
		fmt.Printf("Index: %d, Value: %s\n", i, v)
	}
}
```

---

## 高级用法：`iter.Pull` (拉取式迭代)

默认的 `for-range` 是一种**推送式 (Push)** 迭代：迭代器控制逻辑，不断把数据 `yield` 给循环体。

有时候我们需要**拉取式 (Pull)** 迭代，即手动控制"取下一个"的时机（例如：比较两个序列、合并排序等）。`iter.Pull` 可以把一个 `Seq` 转换为 `next` 函数。

### 函数签名

```go
func Pull[V any](seq Seq[V]) (next func() (V, bool), stop func())
```

### 示例 3：手动拉取数据

假设我们要同时遍历两个切片（Zip 操作）：

```go
package main

import (
	"fmt"
	"iter"
	"slices"
)

func main() {
	s1 := []int{1, 2, 3}
	s2 := []string{"A", "B", "C", "D"}

	// 获取 slice 的标准迭代器
	iter1 := slices.Values(s1)
	iter2 := slices.Values(s2)

	// 将 Push 迭代器转换为 Pull 迭代器
	next1, stop1 := iter.Pull(iter1)
	defer stop1() // 确保资源释放（如果迭代器涉及文件/网络操作很重要）
	
	next2, stop2 := iter.Pull(iter2)
	defer stop2()

	for {
		// 手动拉取数据
		v1, ok1 := next1()
		v2, ok2 := next2()

		// 只要有一个序列结束，就停止
		if !ok1 || !ok2 {
			break
		}

		fmt.Printf("%d -> %s\n", v1, v2)
	}
}
```

**注意**：`iter.Pull` 会启动一个新的 goroutine 来运行原有的迭代器函数，并通过 channel（底层实现优化过，不仅仅是 channel）进行同步。因此，使用完后**必须调用 `stop()`**，以防止 goroutine 泄露。

---

## 标准库的配合 (`slices` 和 `maps`)

Go 1.23 更新了 `slices` 和 `maps` 包，提供了很多基于 `iter` 的辅助函数：

| 函数 | 返回值 | 说明 |
|------|--------|------|
| `slices.All(s)` | `iter.Seq2[int, V]` | 返回 index, value 迭代器 |
| `slices.Values(s)` | `iter.Seq[V]` | 仅返回 value 迭代器 |
| `slices.Collect(seq)` | `[]V` | 将迭代器元素收集到 slice（非常常用） |
| `slices.Sorted(seq)` | `[]V` | 收集并排序后返回 slice |
| `maps.Keys(m)` | `iter.Seq[K]` | 返回 map 的键迭代器 |
| `maps.Values(m)` | `iter.Seq[V]` | 返回 map 的值迭代器 |

### 示例 4：结合 `slices` 包进行过滤和转换

```go
package main

import (
	"fmt"
	"iter"
	"slices"
	"strings"
)

// Filter 是一个简单的中间件，过滤符合条件的元素
func Filter(seq iter.Seq[string], keep func(string) bool) iter.Seq[string] {
	return func(yield func(string) bool) {
		for v := range seq {
			if keep(v) {
				if !yield(v) {
					return
				}
			}
		}
	}
}

func main() {
	data := []string{"apple", "banana", "avocado", "cherry"}

	// 1. 获取迭代器
	src := slices.Values(data)

	// 2. 包装迭代器（过滤出以 'a' 开头的）
	filtered := Filter(src, func(s string) bool {
		return strings.HasPrefix(s, "a")
	})

	// 3. 将结果收集回 slice
	result := slices.Collect(filtered)

	fmt.Println(result) // [apple avocado]
}
```

---

## 总结

1. **标准化**：`iter` 包为 Go 提供了统一的迭代协议
2. **`iter.Seq` / `iter.Seq2`**：通过定义接受 `yield` 回调的函数，即可让自定义类型支持 `for-range`
3. **控制流**：在自定义迭代器中，务必判断 `yield` 的返回值，如果为 `false` 必须 return
4. **`iter.Pull`**：用于将被动的"推送"转换为主动的"拉取"，适合处理多流合并等复杂逻辑，记得 `defer stop()`
5. **生态整合**：优先配合 `slices` 和 `maps` 包中的新函数使用，能极大简化代码（如 `Collect`）

这个特性让 Go 在保持简洁的同时，拥有了类似 Python 生成器或 Java Stream API 的部分能力，但实现上更加轻量且无额外的内存分配（编译器会进行内联优化）。

虽然iter还是相当不像人话，但是你还是可以积极使用在你的项目里hhh。
