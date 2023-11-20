---
title: "Go Practices I follow"
date: 2023-11-20T22:30:59+05:30
draft: true
summary: |
  I have been writing a lot of Go for the past 7 years. In this article, I share
  the practices I have refined over time, which I consistently adhere to.
tags: [golang, patterns, practices]
no_toc: false
---

## 1. Project Structure

After trying multiple different "standards" for organising project, I have realised the best approach
is to start simple and let it evolve. I use the following models as starting points:

1. single-binary (e.g., a `cat` clone in Go)

   ```plaintext
   + github.com/spy16/cat
   |--- .gitignore
   |--- main.go
   |--- go.mod
   |--- go.sum
   |--- README.md
   |--- Makefile
   ```

   This allows users to easily install the binary by doing `go install github.com/spy16/cat` and is very
   simple to understand the structure as well.

2. multi-binary project (e.g., `cat`, `ls`, `ps` in single repo):

   ```plaintext
   + github.com/spy16/unix
   |--+ cmd/
   |  |--+ cat/
   |  |  |--- main.go
   |  |--+ ls/
   |  |  |--- main.go
   |  |--+ ps/
   |  |  |--- main.go
   |--- .gitignore
   |--- go.mod
   |--- go.sum
   |--- README.md
   |--- Makefile
   ```

   This is similar to the previous one but makes room for multiple binaries. Users can easily install each binary
   separately by doing `go install github.com/spy16/unix/cmd/ls`.

In addition, a domain-driven packages works really well as well. For example, if I am building an auth service,
the domain concepts would be `user`, `token` and `session`.
