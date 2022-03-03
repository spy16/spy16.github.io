---
title: "DelayQ"
date: 2022-03-03T11:01:59+05:30
draft: true
summary: |
    I was constantly running into use-cases where things needed to be done
    at a pre-defined time in the future. DelayQ is an experiment that turned
    out to be useful.
tags: [redis, scheduler, crontab]
no_toc: true
---

At my day job, I was constantly running into use-cases where things needed to be done
at a pre-defined time in the future. For example, sending a reminder notification after
2 days, executing some job for a user at configured interval, etc.

Unix crontab system immediately comes to mind when we hear any combination of "periodic",
"sechedule", "job", etc. in the same sentence. Crontab runs as a daemon in a Unix system,
and executes commands locally on the same system based on the `crontab` schedule and is
meant for creating system maintenance jobs, etc. and doesn't really scale well for large
scale use-cases.

We needed something similar but with APIs to manage (all CRUD operations)
the schedules, high-availability & no SPOF.
