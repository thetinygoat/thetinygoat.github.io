---
layout: post
title: "Understanding Go Context"
date: 2022-01-29 00:00:00 +0530
categories: go
---

# Introduction

When I was learning Go around late 2019 I used to build simple web servers using different libraries and many of them had a weird `context` thingy all over the docs. I had no idea what it was supposed to do and when I looked around the inter-webs I was very confused, mainly because concurrent programming as a whole was new to me. It took me quite a while to understand what context was and why It was needed. In this blog post, I hope to lay out my mental model as best as I can to make the `context` package easy to understand.

# Why Do We Need Context?

Let’s take an example, let’s say you have 5 worker goroutines and you need to stop them if the main thread encounters an error.

One simple way to do this would be to use a channel and pass it to all the goroutines.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func doWork(done <-chan bool, wg *sync.WaitGroup) {
	defer wg.Done()
	select {
	case <-done:
		fmt.Println("worker closed before work could be finished")
		return
	case <-time.After(2 * time.Second):
		fmt.Println("work done...")
	}
}

func main() {
	done := make(chan bool)

	wg := sync.WaitGroup{}

	for i := 0; i < 5; i++ {
		wg.Add(1)
		go doWork(done, &wg)
	}
	// fake error
	if true {
		close(done)
	}

	wg.Wait()
}

```

```shell
$ go run main.go
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
```

Here we are launching 5 workers and on error, we are closing the done channel which causes the workers to exit.

Okay simple enough, what’s the issue with this?

Let’s take another example where we need to stop all the workers after a specific timeout. How would we achieve this with the done channel?

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func doWork(done <-chan bool, wg *sync.WaitGroup) {
	defer wg.Done()
	select {
	case <-done:
		fmt.Println("worker closed before work could be finished")
		return
	case <-time.After(10 * time.Second):
		fmt.Println("work done...")
	}
}

func main() {
	done := make(chan bool)

	wg := sync.WaitGroup{}

	for i := 0; i < 5; i++ {
		wg.Add(1)
		go doWork(done, &wg)
	}

	wg.Add(1)
	go func() {
		<-time.After(5 * time.Second)
		close(done)
		wg.Done()
	}()

	wg.Wait()
}
```

```shell
$ go run main.go
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
```

Just like before we are launching 5 workers but this time we launch another goroutine that closes the done channel after 5 seconds effectively setting a timeout on the done channel.

As you can see even for this trivial example it gets pretty chaotic, and in a bigger codebase where you might have to set several timeouts, this can get pretty bad.

Setting aside timeouts, what if you want to send additional info as to why the done channel was closed?

To solve all this Go introduced the context package.

# The Context Package

The best way to understand the context package is to use it. Let’s refactor the first example to use context instead of a done channel, and I will explain what’s going on.

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func doWork(ctx context.Context, wg *sync.WaitGroup) {
	defer wg.Done()
	select {
	case <-ctx.Done():
		fmt.Println("worker closed before work could be finished")
		return
	case <-time.After(10 * time.Second):
		fmt.Println("work done...")
	}
}

func main() {

	ctx, cancel := context.WithCancel(context.Background())

	wg := sync.WaitGroup{}

	for i := 0; i < 5; i++ {
		wg.Add(1)
		go doWork(ctx, &wg)
	}

	if true {
		cancel()
	}

	wg.Wait()
}

```

```shell
$ go run main.go
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
```

Okay, let’s go bit by bit to understand what’s going on.

## `context.Background()`

Everything starts with `context.Background()`, it initializes an empty context that can be passed on to the other functions provided by the context package that take in the empty context and create a new context.

## `context.WithCancel()`

By itself `context.Background()` does nothing we pass it to other functions provided by the context package that create a new context out of it. `context.WithCancel()` takes in an empty context and returns a new context and a `cancelFunc`, this `cancelFunc` can be used to close the new context's `Done` channel, which is what we are doing here. Unlike the first example where we were manually creating a done channel and closing it, we just pass the context and call the `cancelfunc` whenever we want.

This is pretty good but it feels we just changed a hand-made channel to a managed one.

Well...let’s refactor the second example.

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func doWork(ctx context.Context, wg *sync.WaitGroup) {
	defer wg.Done()
	select {
	case <-ctx.Done():
		fmt.Println("worker closed before work could be finished")
	case <-time.After(10 * time.Second):
		fmt.Println("work done...")
	}
}

func main() {

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	// release resources if the work is finished before the context timeout
	defer cancel()

	wg := sync.WaitGroup{}

	for i := 0; i < 5; i++ {
		wg.Add(1)
		go doWork(ctx, &wg)
	}

	wg.Wait()
}
```

```shell
$ go run main.go
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
```

Even from the first look, this looks a lot cleaner than our hand-made version.

## `context.WithTimeout()`

Just like `context.WithCancel()` we also have `context.WithTimeout()` which returns a new context and a cancel function. The new context's `Done` channel is automatically closed after the timeout.

Why do we have a cancel function if the `Done` channel is automatically closed?

What if the worker finished its work before the timeout? We must call the cancel function to make sure that all the resources are released.

## Other functions

There are other functions like `context.WithDeadline()` and `context.WithValue()` I think it's pretty clear from their names what they do but for the sake of completeness let's take a look at a few examples.

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func doWork(ctx context.Context, wg *sync.WaitGroup) {
	defer wg.Done()
	select {
	case <-ctx.Done():
		fmt.Println("worker closed before work could be finished")
	case <-time.After(10 * time.Second):
		fmt.Println("work done...")
	}
}

func main() {

	ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(5*time.Second))
	// release resources if the work is finished before the context timeout
	defer cancel()

	wg := sync.WaitGroup{}

	for i := 0; i < 5; i++ {
		wg.Add(1)
		go doWork(ctx, &wg)
	}

	wg.Wait()
}

```

```shell
$ go run main.go
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
worker closed before work could be finished
```

`context.WithDeadLine()` is pretty similar to `context.WithTimeout()` but instead of passing a timeout we pass a duration.

```go
package main

import (
	"context"
	"fmt"
)

type ctxKey string

func doWork(ctx context.Context, key ctxKey) {
	if value := ctx.Value(key); value != nil {
		fmt.Println("value found: ", value)
		return
	}
	fmt.Println("value not found")
}

func main() {
	key := ctxKey("hello")
	ctx := context.WithValue(context.Background(), key, "world")

	doWork(ctx, key)
	doWork(ctx, ctxKey("foo"))
}
```

```shell
$ go run main.go
value found:  world
value not found
```

`context.WithValue()` is used to send request-scoped data (key-value pairs) with the context. The Go docs state that the users of `context.WithValue()` should define their own types to avoid collisions with packages using context and that's what we have done with `ctxKey`

I would highly suggest reading the documentation as it goes into more detail about each of the functions.

# Conclusion

Context is a pretty powerful tool and once I understood it I have been using it in all my projects. I hope this post gave you a basic understanding of what context is and what problem it solves.
