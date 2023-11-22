---
title: "Structuring your Go code"
date: 2023-11-22T22:30:59+05:30
draft: true
summary: |
  I present my experience on trying different project structure in Go and what I
  now regularly use after 8 years of Go.
tags: [golang, patterns, practices]
no_toc: false
---

I present my experience on trying different project structure in Go and what I
now regularly use after 8 years of Go.

## Project Types

Go is typically used in 4 different settings:

1. A library project (e.g., [Cobra](https://github.com/spf13/cobra))
2. A single-binary project (e.g., [fzf](https://github.com/junegunn/fzf) -- A tool, I cannot live without ❤️)
3. A multi-binary project (e.g., [Kubernetes](https://github.com/kubernetes/kubernetes))
4. A library + binary project.

### 1. Library Project

Go libraries are meant to be imported in other Go projects. So the end-users are developers trying to use the
library in their own project. As a library writer/maintainer, your goal should be to make it as easy and simple
as possible for them.

The core functionalities, types of your library, etc. should be situated in the root of the repository.

If you come from `src/` background, this might sound like an annoying practice at first. But Go import system works by starting at the root of a module. If you were to put all your code into `github.com/spf13/cobra/src`, the import path would be that too. So the user is forced to either use `src.Command{}`:

```golang
import "github.com/spf13/cobra/src" // use as src.Command{}
```

or use import-alias everywhere:

```golang
import cobra "github.com/spf13/cobra/src" // use as cobra.Command{}
```

Without the `src/` or any such nesting, users of your package can import and simply use `cobra.Command{}` which is definitely much clearer (than using `src.`) and less-annoying (than being forced to import-alias everywhere).

```golang
import "github.com/spf13/cobra" // use as cobra.Command{}
```

### 2. Single Binary Project

A single binary project offers its end-users just that, a single binary that they can install and run.

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

* Very simple to understand. Anyone opening the repo for the first time can simply open the `main.go` to understand how things work.
* Allows users to easily install the binary by doing `go install github.com/spy16/cat`.

**Note**: Regardless of this, you should still have automated release mechanism to build binaries for different architectures and operating systems using [GoReleaser](https://goreleaser.com/) or similar build tools.

### 3. Multi-binary project

This type of projects are usually large and have multiple binaries that form an interacting system of sorts. For example, Kubernetes is structured as a multi-binary project since it generates `kubectl`, `kube-proxy`, `kubelet`, etc. which are all necessary components of the Kubernetes platform.

As an example, If we are building a Go project that has clones of some Unix tools, we can structure them like below:

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

* The `cmd/` directory helps us quickly identify the 'deliverables' section of the codebase, simplifying navigation. This is also very useful for codebase exploration, as `func main()` is usually where the reading journey starts.
* Also, notice that the names of directories inside the `cmd/` which is same as the desired binary name. This is recognized by Go toolchain and the output binary will be named appropriately.
* Anyone wanting to install the binaries, they would be able to do `go install github.com/spy16/unix/cmd/ls` etc. and the binary will be added to their `GOBIN` path with correct name (i.e., `ls`).

### 4. Binary + Library Project

*It would be ideal if all projects followed this structure to promote re-use. For example, we needed to build automations using Helm charts. But [Helm](https://github.com/helm/helm/) project exposes only the command definitions. So we can not use it as an SDK to apply helm-charts. This forced us to use the commands directly in weird ways. But libraries are contracts that need to be mainted, so this is simply impractical.*

## General Suggestions

In addition, a domain-driven packages work really well as well. For example, if I am building an auth service,
the domain concepts would be `user`, `token` and `session`.
