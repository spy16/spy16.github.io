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

## Decoupling with interfaces
