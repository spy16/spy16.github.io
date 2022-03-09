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

We could always use unix cron[^1] of course. But Cron runs as a daemon in a Unix system,
and executes commands locally on the same system based on the `crontab` schedule and doesn't
really scale for large scale use-cases. For example, millions of users, each having multiple
dedicated schedules.

*We needed something similar but with APIs to manage (all CRUD operations)
the schedules, high-availability (HA) & no Single Point of Failure (SPOF).*

The API and schedule management aspects are the easy part of this sytem. The crux of the
problem is the need for an efficient "scheduler" that also provides HA and has no SPOF.

{{<whatis "Scheduler">}}
Some component that notifies rest of the system when a schedule becomes ready 
(i.e., next execution time matches the current time).
{{</whatis>}}

{{<mermaid>}}
sequenceDiagram
  Scheduler->>Handler: Notify handler about a ready point
  Handler-->>Handler: Do the needed work
  Handler-->>Handler: Compute the next execution point `t`
  Handler->>Scheduler: Ask scheduler to notify again at `t`
{{</mermaid>}}

If we unrol a crontab like `@every 1h` it's a timeline with **execution points** on it. If we assume the first point is at `t`, then 2nd is at `t+1h`, 3rd one is at `(t+1h)+1h`, and so on. Considering never-ending recurring crontabs like `@every 1h`, it is clear that all execution points cannot be inserted at the time of schedule creation, but will have to be done lazily (i.e., after handling one execution point, compute the next and insert it into scheduler).

When we consider steps of computing & inserting next execution point into scheduler, as part of the execution process, this becomes a job queue with a special requirement -- Delayed/Timed Delivery of Jobs.

## Delayed Job Queue

A typical job-queue has no time constraint for dequeue -- A job is ready as soon as enqued.

For designing a crontab scheduler, we need to build a job queue where the enqueued jobs become ready for dequeue only after specific time.



[^1]: https://man7.org/linux/man-pages/man8/cron.8.html
