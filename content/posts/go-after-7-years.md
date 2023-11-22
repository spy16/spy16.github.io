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

I have been writing a lot of Go for the past 7 years (For work and [personal projects](https://github.com/spy16?tab=repositories&q=&type=&language=go&sort=stargazers)). In this article, I share
the practices I have refined over time (after trying other competing patterns as well), which I consistently adhere to.

Most of these patterns are not my own inventions. They are all well-documented in Go-[CodeReviewComments](https://github.com/golang/go/wiki/CodeReviewComments) and other style guides from many companies that use Go heavily (e.g., Google, Uber, Gojek, etc.).

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
   |--- Dockerfile
   ```

   * Very simple to understand. Anyone opening the repo for the first time will simply open the `main.go`.
   * Allows users to easily install the binary by doing `go install github.com/spy16/cat`.

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
   |--- (... other packages ...)
   |--- .gitignore
   |--- go.mod
   |--- go.sum
   |--- README.md
   |--- Makefile
   ```

   * Similar to the previous one but makes room for multiple binaries.
   * Users can easily install each binary separately (e.g., `go install github.com/spy16/unix/cmd/ls`)

In addition, a domain-driven packages work really well as well. For example, if I am building an auth service,
the domain concepts would be `user`, `token` and `session`.
