---
title: "Terraform Basics"
date: 2022-06-20T12:25:59+05:30
draft: false
summary: |
  My learnings of terraform.
tags: [terraform, til]
no_toc: false
---

## Basics

- Terraform is a tool for building and changing infrastructure safely and efficiently.
- Terraform centered around infrastructure-as-code (IAC) principle which makes it well suited for versioned, reproducible infrastructure definition.
- Infrastrucutre is described using `.tf` files that use the terraform configuration [langauge](https://www.terraform.io/language).

## Workflow

### Project Setup

A terraform project is just a set of files describing the provider, resources, etc.

For example, imagine this is the project setup we have:

```plaintext
+ mars
|--+ main.tf
```

And the contents of `main.tf`:

```plaintext
provider "local" {
  // attributes if any
}

resource "local_file" "hello" {
  content = "Hello, Terraform"
  filename = "hello.txt"
}
```

- The `provider` block describes the platform where resources will be created - this can include Cloud services (GCP, AWS, etc.), SaaS services, API providers, etc. [Local provider](https://registry.terraform.io/providers/hashicorp/local/latest) allows creating resources locally (e.g., files).
- The `resource` block describes a resource that needs to be created along with all the properties. The resource type (e.g., `local_file`) must be supported by the `provider`.

### Init

Running `terraform init` discovers all the providers that are needed by the project, downloads the provider plugins for them from the [Terraform Registry](https://registry.terraform.io/).

```shell
$ terraform init
Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/local from the dependency lock file
- Using previously-installed hashicorp/local v1.4.0

Terraform has been successfully initialized!
```

### Planning

In this stage, `terraform` finds the current states of all the resources the terrform module cares about
and computes the steps necessary to match them to the desired state.

```bash
$ terraform plan
Terraform used the selected providers to generate the following execution plan. Resource
actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # local_file.hello will be created
  + resource "local_file" "hello" {
      + content              = "Hello, Terraform"
      + directory_permission = "0777"
      + file_permission      = "0777"
      + filename             = "hello.txt"
      + id                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

───────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to
take exactly these actions if you run "terraform apply" now.
```

Notice the message at the end. Using `-out` can ensure that apply phase will take the exact
steps shown during `plan`. Without `-out`, some dependent state might change and terraform
might be forced to **re-plan** changing the overall steps since the last `plan` call.

### Apply

Apply phase is when the plan described by the module is actually executed. For example,
running `terraform apply` inside the above project directory will create a `hello.txt` file with
the "Hello, Terraform" message.

```shell
Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # local_file.hello will be created
  + resource "local_file" "hello" {
      + content              = "Hello, Terraform"
      + directory_permission = "0777"
      + file_permission      = "0777"
      + filename             = "hello.txt"
      + id                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

local_file.hello: Creating...
local_file.hello: Creation complete after 0s [id=392b5481eae4ab2178340f62b752297f72695d57]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

At this point, if you run `terraform plan` again, it will be skipped because the real-world state
matches the terraform description exactly. But if you modify the `hello.txt` file to contain
something else and then run, `terraform` will detect the changes and re-plan accordingly.
