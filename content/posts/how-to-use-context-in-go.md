---
title: "How to use Go contexts"
date: 2023-11-11T23:03:59+05:30
draft: true
summary: |
    In this article I show different patterns of using Go contexts and things to avoid.
tags: [golang, patterns, practices]
no_toc: false
---

Go provides various primitives and building blocks to facilitate the development of large-scale concurrent systems. One such important primitive is the [context.Context] type, which is extensively used in Go-based systems. As official documentation states, [context.Context](https://pkg.go.dev/context#Context) type is for carrying deadlines, cancellation signals, and other request-scoped values across API boundaries and between processes.

However, despite its widespread usage, developers often encounter confusion regarding the correct ways to make use of the context package. In this article I will discuss different problems context can be used to solve and things to avoid.

## What is Context?

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

The easiest way to create a context value is using `context.Background()` which returns an empty-context without any deadline/cancellation in it. Most of standard library and popular libraries usually provide context as appropriate as we will see in following sections.

## Cancelling with Context

One of the most important use-cases of context is handling cancellation. Cancellation is the process of terminating or stopping the execution of a task or operation. With context, we can propagate cancellation signals across API boundaries and between goroutines to gracefully stop the execution of a task when it is no longer needed or when an error occurs.

It's important to note that the context is not inherently tied to the runtime, and passing a context to a goroutine does not automatically make it stoppable. The function running in the goroutine needs to explicitly check the context at appropriate points and make the decision to exit.

Context cancellation is a mechanism to **inform your goroutines in concurrent-safe way that their shutdown is imminent, allowing them to safely save or discard their work.**

We can make any context value cancellable, using [WithCancel](https://pkg.go.dev/context#WithCancel), [WithTimeout](https://pkg.go.dev/context#WithTimeout), [WithDeadline](https://pkg.go.dev/context#WithDeadline), etc.

```golang
ctx, cancel := context.WithCancel(baseCtx)
```

And we can check if a context is cancelled using a `select` statement:

```golang
select {
    case <-ctx.Done():
        // context has been cancelled.
    
    default:
        // context is still valid. 
}
```

### Stopping Goroutines

Consider this function: 

```golang
func printTime() {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()
    for t := range ticker.C {
        fmt.Println(t.String())
    }
}
```

If you call this in your `main()`, it would block forever and keep printing time. But of course if you don't want that, you can launch it to run concurrently as a separate gouroutine using the `go printTime()` construct.

But what if you wanted to be able to stop it when needed. This is one of the use-cases where context comes in handy.

We could refactor as shown below:

```golang
func printTime(ctx context.Context) {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for {
        select {
            case <-ctx.Done():
                return

            case t := <-ticker.C:
                fmt.Println(t.String())
        }
    }
}
```

The behaviour of the function remains the same except now it also considers the `ctx` on every iteration. If the `ctx` is cancelled at any time, the next check will exit the function, thus allowing us to stop the work when needed.

### HTTP Requests

Let's consider an example where we have an HTTP endpoint `POST /compute` with the following Go handler:

```golang
func computeHandler(w http.ResponseWriter, r *http.Request) {
    // Read parameters sent by the client (as JSON body)
    var params ComputeParams
    _ = json.NewDecoder(r.Body).Decode(&params) 

    // Perform some computation.
    result, ok := veryExpensiveComputation(params)

    if ok {
        // Send the result back to the client
        _ = json.NewEncoder(w).Encode(result)
    }
}
```

In this scenario, we want to send user input from a mobile app to the server, perform a computation, and then send the result back to the client. However, there are cases where the user may close the app or the request may be dropped due to a network issue. In such situations, it is unnecessary to complete the computation and send the result back, as the client is no longer interested in receiving it. We could potentially save a lot of unnecessary work by using context here.

Let's say we refactored our `veryExpensiveComputation` to accept and use a context, we can pass it `r.Context()` to ensure it receives a cancellation when the request ends.

```golang
func computeHandler(w http.ResponseWriter, r *http.Request) {
    // ...

    result, ok := veryExpensiveComputation(r.Context(), params)

    // ...
}
```

### Graceful Shutdown

One important aspect that often gets overlooked is what happens when your server restarts (e.g., due to deployment, scaling down, etc.) and there are ongoing requests. If you do not implement graceful shutdown on your server, these requests could terminate abruptly giving a very bad experience to the end-user. 

We can use contexts for graceful shutdown as well. For example, I use this for HTTP servers:

```golang
func GracefulServe(ctx context.Context, srv *http.Server) error {
    errCh := make(chan error, 1)

    go func() {
        err := srv.ListenAndServe();
        if  err != nil && !errors.Is(err, http.ErrServerClosed) {
            errCh <- err
        }
    }()

    select {
    case <-ctx.Done():
        // we use a new context with timeout to allow the server to
        // finish already accepted requests.
        ctx, cancel := context.WithTimeout(
            context.Background(), 5*time.Second,
        )
        defer cancel()

        return srv.Shutdown(ctx)

    case err := <-errCh:
        return err
    }
}
```

It can be used to shutdown when process receives `SIGINT` as shown below:

```golang
func main() {
    ctx, cancel := signal.NotifyContext(
        context.Background(), os.Interrupt,
    )
    defer cancel()

    server := &http.Server{
        Addr: ":8080",
        Handler: myRouter,
    }

    if err := GracefulServe(ctx, server); err != nil {
        log.Fatalf("server exited with error: %v", err)
    }
}
```

The same `ctx` value can be passed to database drivers, gRPC server instances, worker goroutines, etc. to signal a graceful shutdown across the whole system. 
