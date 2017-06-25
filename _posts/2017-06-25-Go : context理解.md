---
layout: post_layout
title: Go：context理解
time: 2017年06月25日 星期日
location: 北京
pulished: true
---

在看proxy的时候，看到了context这个package.

<!--break-->

source 1: [Package context](https://golang.org/pkg/context/)


```
Incoming requests to a server should create a Context, 

and outgoing calls to servers should accept a Context

> * 对于，访问本服务器的Incoming 请求应该建立一个Context.

> * 对于，从本服务器访问其他服务器的Outgoing 请求应该接收Context.

```

下面的链接中包含了创建一个context的代码。

source 2: [Go Concurrency Patterns: Context](https://blog.golang.org/context)

```
WithCancel
WithDeadline
WithTimeout
WithValue

以上函数根据不同的含义，分别返回parent Context的child Context。

package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	d := time.Now().Add(50 * time.Millisecond)
	ctx, cancel := context.WithDeadline(context.Background(), d)

	// Even though ctx will be expired, it is good practice to call its
	// cancelation function in any case. Failure to do so may keep the
	// context and its parent alive longer than necessary.
	defer cancel()

	select {
	case <-time.After(1 * time.Second):
		fmt.Println("overslept")
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}

}
```

总结：Context是Go提供的一种机制，可以完成：

场景1：有一个请求，其中包含了若干子请求。当子请求还是与对应的服务器通信时，父请求因为各种各样的原因停止了,

此时，单纯的关闭父请求是没有意义的，Context机制就可以实现关闭父请求的同时，关闭其他的子请求。


场景2：传递 request-scoped 数据，还没完全弄清楚，待续。

