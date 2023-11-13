---
title: "Accept Interfaces - Return Structs, Please!!"
date: 2023-11-11T23:03:59+05:30
draft: true
summary: |
  My thoughts on Go interfaces idiom after 7 years of doing Go.
tags: [golang, patterns, practices]
no_toc: false
---

I want to talk about a Go idiom that I have been using for a while now: _Accept interfaces, return structs_.
I think it is a pattern that can dictate how your codebase evolves over time (and in many cases, whether it
will reach a point that needs a complete rewrite).

## Interfaces in Go: A refresher

Here's how interfaces are defined in Go:

```golang
package foo

type Executor interface {
    Execute() string
}
```

And here's how they are implemented:

```golang
package bar

type ExecutorA struct {}

func (r *ExecutorA) Execute() string { return "I am A" }
```

If you look at the second snippet closely, you will notice that `ExecutorA` does not explicitly declare that it
implements `Executor` interface. It just has the `Execute` method with the same signature as that in the 
`Executor` interface, and hence, it is considered to be an implementor of `Executor` interface.

The lack of `implements` or equivalent keyword is the result of [structural type-system](https://en.wikipedia.org/wiki/Structural_type_system) in Go.
This also has some interesting implications on how interfaces are used in Go.

For example, consider the following function:

```golang
func writeMessage(f *os.File, name string) {
    msg := fmt.Sprintf("Hello %s", name)
    _, _ = f.Write([]byte(msg))
}
```

The `f` parameter of the function is a pointer to `os.File` type -- a concrete type. This means that
the function can only be called with an actual writeable file. In languages like Java, C#, etc., we
might run into problem if we wrote a function like this when we wanted to test it, or re-use it to
write to a buffer, or a network connection, or a file.

Interfaces are a great way to define contracts between different components of a system. They allow
us to define the behaviours that a component expects from its dependencies. This allows us to build
components that are decoupled from their dependencies and can be tested in isolation. We can also
swap out the dependencies with different implementations without changing the component's code.

For example, consider the following function:

```golang
package foo

func DoWork(e Executor) bool {
    result := e.Execute()
    return result == "success"
}
```

The function `DoWork` does not care which types implement the `Executor` interface. It just expects
value `e` to have a method `Execute` that returns a string. This *allows us to think* of the function
`DoWork` in isolation. The value could be of type `ExecutorA` or `ExecutorB` or any other type.

The best practices that the Go community has developed over the years around interfaces are:

1. Define small interfaces with well defined scope
   - Single-method interfaces are ideal (e.g. `io.Reader`, `io.Writer` etc.)
   - [Bigger the interface, weaker the abstraction - Go Proverbs by Rob Pike](https://www.youtube.com/watch?v=PAAkCSZUG1c&t=5m17s)
2. Accept interfaces, return structs.
