---
title: "DelayQ"
date: 2022-03-03T11:01:59+05:30
draft: true
summary: |
    I was constantly running into use-cases where things needed to be done
    at a pre-defined time in the future. DelayQ is an experiment that turned
    out to be useful.
tags: [redis, scheduler, crontab]
no_toc: false
---

At my day job, I was constantly running into use-cases where things needed to be done
at a pre-defined time in the future. For example, sending a reminder notification after
2 days, executing some job for a user at configured interval, etc.

## Unix Cron

Unix cron[^1] system immediately comes to mind when we hear any combination of "periodic",
"sechedule", "job", etc. in the same sentence. Cron runs as a daemon in a Unix system,
and executes commands locally on the same system based on the `crontab` schedule and is
meant for creating system maintenance jobs, etc. But it doesn't really scale for large
scale use-cases. For example, millions of users, each having multiple dedicated schedules.

*We needed something similar but with APIs to manage (all CRUD operations)
the schedules, high-availability & no SPOF.*

## Problem of Scheduling

If we think about it, the API and schedule management aspects are the easy part of this sytem.
The crux of the problem is the need for an efficient "scheduler". Some component that notifies
the rest of the system when a schedule matches the current time.


[^1]: https://man7.org/linux/man-pages/man8/cron.8.html
