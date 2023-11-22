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

Go uses a [structural type system](https://en.wikipedia.org/wiki/Structural_type_system). This is a big shift if you're used to interfaces in languages like Java or C#, where a type can **implement an interface only by explicitly declaring**.

I think this key difference gets missed/ignored by newcomers to Go (even experienced sometime) due to the bias of how they have always used interfaces in these other languages. But this has strong implications on where and how you should use interfaces, on decoupling, on refactoring, etc. And also provides really significant benefits over the explicit-interfaces (i.e., [nominal type-system](https://en.wikipedia.org/wiki/Nominal_type_system))

## The Anti-pattern

You might have noticed this pattern in some Go codebases (e.g., in your org):

```golang
package redis

type Client interface {
  Get(key string) (string, error)
}

func New() Client {
  return &client{}
}

type client struct {}

func (cl *client) Get(key string) (string, error) {
  return "value-for-key", nil
}
```

And then in another package, the client would be used by importing the interface:

```golang
package caching

import "redis"

type service struct {
  client redis.Client
}
```

This is the "anti" of the "accept interfaces, return structs" pattern that Go community and many Go style-guides preach. And based on my experience, I concur and honestly believe that **this is the worst way to use interfaces in Go**.

## Accept Interfaces

Take this function for example:

```golang
package mybusiness

func writeResult(target *os.File, data []byte) error {
  result := compute(data)
  _, err := target.Write(result)
  return err
}
```

This function computes and writes data to a file. But what if you need to write to a *bytes.Buffer instead? You could create a new function, but there's a more elegant solution.

The key is realizing `writeResult()` only needs a "writable" target (i.e., a value with `Write()` method). The actual type of the value or whether it has `Read()` or `Close()` or any other infinite possibilities do not matter to the `writeResult()` function. This is the true purpose of interfaces:

```golang
package mybusiness

type Writer interface {
  Write(data []byte) (n int, err error)
}

func writeResult(target Writer, data []byte) error {
  result := compute(data)
  _, err := target.Write(result)
  return err
}
```

The `Writer` interface now clearly describes what `writeResult()` really needs from `target` to do its job and abstracts it away from what the `target` value actually is. This is what abstraction is all about - Hiding away unnecessary details from those who don't need them.

A bonus: `writeResult()` now magically supports TCP sockets, UDP sockets, byte buffers, string buffers, files, stdout, stderr, etc 😁.

And the best part is, it does so without breaking current usages of `writeResult()` - In languages like Java, that's impossible because you need to ensure all current values being passed have `implements Writer` on their class definitions, which would lead to a lot of refactor in large codebases.

## Return Structs

Package `bar` is forced to depend on package `foo`. This is opposite of decoupling - which is one of the reasons to use interfaces in the first place.

Projects that use this also run into cyclic-dependency issues if they try to apply layered architectural patterns together (e.g., `storage` package needs `user.User` type and `user.Service` needs `storage.Store` interface). The way they end up resolving this issue is by having a `models` package 🤦 - which has no clear domain/purpose apart from acting as a dumping ground for any type that needs to be shared.

## Accept Interfaces, Return Structs

In the "Accept Interfaces" section, we disucssed how we can abstract unnecssary details from "user" of different values. In the "Return Structs" section we discussed how returning concrete-typed values instead of private-value-behind-interface can give the users of your type the flexibility to pick-and-choose what they need.
