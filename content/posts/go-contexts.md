---
title: "Contexts in Go"
date: 2023-11-11T23:03:59+05:30
draft: true
summary: |
    This article explores the concept of contexts in Go, their purpose, and best practices for using them effectively.
tags: [golang, patterns, practices]
no_toc: false
---

Go provides various primitives and building blocks to facilitate the development of large-scale concurrent systems. One such important primitive is the [context.Context] type, which is extensively used in Go-based systems. As official documentation states, [context.Context](https://pkg.go.dev/context#Context) type is for carrying deadlines, cancellation signals, and other request-scoped values across API boundaries and between processes.

However, despite its widespread usage, developers (especially with different background) often encounter confusion regarding the correct ways to make use of the context package. In this article I will discuss different problems context can be used to solve and things to avoid.

## What is a Context?

The `context.Context` is the following interface:

```golang
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

Any value that implements this interface (with all the expected behavior) is considered a valid Context. In most cases, there is no need to implement a custom context since Go provides all the necessary functionality. Custom contexts are strongly discouraged.

You will frequently see functions in Go defined like shown below:

```golang
func sayHello(ctx context.Context, name string) {
    // do some work.
}
```

The most straightforward way to create a context value is by using `context.Background()`, which returns an empty context without any deadline or cancellation. If you need to invoke a function like the one shown above but you don't have a context value, the correct approach would be:

```golang
sayHello(context.Background(), "Bob")
```
