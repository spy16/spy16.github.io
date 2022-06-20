---
title: "DelayQ"
date: 2022-03-03T11:01:59+05:30
draft: true
summary: |
  Building a reliable, efficient delayed task-queue or job-queue with Redis.
tags: [redis, scheduler, crontab]
no_toc: false
---

## Problem

> In system software, a job queue (sometimes batch queue), is a data structure maintained by job scheduler software containing jobs to run. -- Wikipedia

In a typical job/task-queue, a job is ready as soon as it is enqueued. A worker will dequeue and process as soon as possible. But what if we needed a way to enqueue a job now, but execute it only after some configurable delay or at some specific time instant?

One way would be to use `sleep()` or equivalent before enqueing:

```python
def execute_after(job, duration):
    sleep(duration)
    enqueue(job)
```

But as you can imagine, this isn't really scalable or reliable. What if we have thousands of jobs that need to be executed at specific times?, What if the program crashes while it is in sleep?, etc.

It would be ideal if the underlying job-queue system itself provides a feature to set "ready time" while enqueing jobs and allows only "ready items" to be dequeued.

In other words (or code), this is the interface we would like:

```golang
type DelayQueue interface {
    // Enqueue should ensure that the `data` becomes ready
    // for dequeue only at `readyTime`.
    Enqueue(readyTime time.Time, data []byte) error

    // Dequeue should return a "ready item" (i.e., An item that
    // was enqueued with `redyTime <= relativeTo`). Caller can
    // pass `time.Now()` here to get an item that is ready right
    // now.
    Dequeue(relativeTo time.Time) ([]byte, error)
}
```

## Solutions

I present few approaches to implement this in this post.

### Priority Queue

We could use a priority-queue (like min-heap) and store items with their `readyTime` as the
priority.

```golang
func (pq *PriorityQueue) Enqueue(readyAt time.Time, data []byte) error {
    pq.push(
        readyAt /* priority of this item */,
        data    /* the item value */,
    )
    return nil
}

func (pq *PriorityQueue) Dequeue(relTime time.Time) ([]byte, error) {
    priority, _ = pq.peek()
    if priority > relTime {
        // No job is ready yet. Try dequeue again after sometime.
        return nil, ErrNoReadyItem
    }

    return pq.pop()
}
```

If th

I was interested to see if it can be done efficiently using Redis -- Because Redis is in-memory, fast, and provides various data structures that we can use. List and sorted-set in particular are very useful for building job queues with Redis.

A normal FIFO job-queue can be implemented using list data-structure by combining `LPUSH` &
`BRPOP` commands [^3]. But the problem with this design is that if the worker that picks up
the job crashes, the job is lost because it has been removed from the `jobs` list. To support
recovery, the popped item should be moved to an ongoing list atomically and a recovery process
should reconcile these items somehow if worker crashes. This can be accomplished using `BLMOVE`[^4].

While above approach works as normal FIFO job-queue, it does not help us in delaying the items.

There is a pattern provided for building reliable delayed task-queue with Redis in the book
_Redis in Action_ [^5]. But the proposed approach uses lock and secondary list-based queues to
implement delayed-task execution.

[^1]: https://redis.io/topics/benchmarks
[^2]: https://www.2ndquadrant.com/en/blog/what-is-select-skip-locked-for-in-postgresql-9-5/
[^3]: https://redis.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-4-task-queues/6-4-1-first-in-first-out-queues/
[^4]: https://redis.io/commands/lmove#pattern-reliable-queue
[^5]: https://redis.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-4-task-queues/6-4-2-delayed-tasks/
