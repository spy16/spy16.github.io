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

## The Problem 

At my day job, I was constantly running into use-cases where things needed to be done
at a pre-defined time in the future. For example, sending a reminder notification after
2 days, executing some job for a user at configured interval, etc.

We could always use unix cron[^1] of course. But Cron runs as a daemon in a Unix system,
and executes commands locally on the same system based on the `crontab` schedule and doesn't
really scale for large scale use-cases. For example, millions of users, each having multiple
dedicated schedules.

*We needed something similar but with APIs to manage (all CRUD operations) the schedules, high-availability (HA) & no Single Point of Failure (SPOF).*

{{<figure src="/scheduler.png">}}

## Decomposing It

When we unrol a crontab like `@every 1h`, it can be thought of as a timeline with **execution points** on it. If we assume the first point is at `t`, then 2nd is at `t+1h`, 3rd one is at `(t+1h)+1h`, and so on. In case of never-ending recurring crontabs like `@every 1h`, all execution points cannot be inserted at the time of schedule creation, but will have to be done lazily (i.e., after handling one execution point, compute the next and insert it into scheduler).

When we consider steps of computing & inserting next execution point into scheduler, as part of the execution process, this becomes a job queue with a special requirement -- Delayed/Timed Delivery of Jobs.


## Delayed Job Queue

A typical job-queue has no time constraint for dequeue -- A job is ready as soon as enqued.

For designing a crontab scheduler, we need to build a job queue where the enqueued jobs become ready for dequeue only after specific time.

```golang
type DelayQ interface {
    // Enqueue item and ensure it becomes ready at 'readyTime'.
    Enqueue(readyTime time.Time, payload []byte) error
    // Dequeue any item that is ready and invoke h. This includes
    // any item that was enqueued with readyTime <= relativeTo.
    Dequeue(relativeTo time.Time, apply HandlerFn) error
}

// HandlerFn is invoked by DelayQ for every ready item.
type HandlerFn func(t time.Time, payload []byte) error
```

This is rougly the interface we need to implement. There are obviously multiple approaches to 
implementing this. 

For example, we could use a database table for storing the items with index on the ready-time.
Workers can then range-scan the table repeatedly. However, it is hard to get this right while
being concurrent and efficient [^3].

For that reason, I was interested to see if it can be done efficiently using Redis -- Because
Redis is in-memory, fast, and provides various data structures that we can use.

## Using Redis

Redis provides lot of in-memory data structures including hashes, lists, sorted-sets, bitmaps, etc.
List and sorted-set in particular are very useful for building job queues with Redis.

A normal FIFO job-queue can be implemented using list data-structure by combining `LPUSH` &
`BRPOP` commands [^4]. But the problem with this design is that if the worker that picks up
the job crashes, the job is lost because it has been removed from the `jobs` list. To support
recovery, the popped item should be moved to a ongoing list atomically. This can be accomplished
using `BLMOVE`[^4].

While above approach works as normal job-queue, it does not help us in delaying the items.

There is a pattern provided for building reliable delayed task-queue with Redis in the book
*Redis in Action*. But the proposed approach uses a distributed locks to ensure duplicate
processing does not happen.

[^1]: https://man7.org/linux/man-pages/man8/cron.8.html
[^2]: https://www.2ndquadrant.com/en/blog/what-is-select-skip-locked-for-in-postgresql-9-5/
[^3]: https://redis.io/topics/benchmarks
[^4]: https://redis.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-4-task-queues/6-4-1-first-in-first-out-queues/
[^4]: https://redis.io/commands/lmove#pattern-reliable-queue
