---
title: "Scheduled Job Queue"
date: 2022-11-19T11:01:59+05:30
draft: false
summary: |
  Building a reliable, efficient scheduled task-queue or job-queue with Redis.
tags: [redis, scheduler, crontab, job-queue]
no_toc: false
---

I was working on a product where we needed the ability to schedule certain background jobs to happen at specific time in the future. After evaluating certain existing solutions including Google Cloud Scheduler, Dkron, etc., we decided to build something internally due to either reliability, scale, cost or security reasons.

## Problem

At a high level, we can think of what we need as a special kind of job-queue. In a typical job/task-queue, a job is ready as soon as it is enqueued. A worker will dequeue and process as soon as possible. But for our requirement, we need a job queue where we can enqueue each job with a ready-time and it can be dequeued only after that ready-time.

A simple illustration of this requirement using a normal job-queue would be:

```python
def enqueue(job, ready_at):
    sleep_until(ready_at)
    typical_job_queue.enqueue(job)

def dequeue():
    return typical_job_queue.dequeue()
```

Obviously, sleeping isn't a feasible strategy at all. What if we have thousands of jobs that need to be executed at specific times?, What if the program crashes while it is in sleep?, etc.

It would be ideal if the underlying job-queue system itself provides a feature to set "ready time" while enqueing jobs and allows only "ready items" to be dequeued.

In other words (or code), this is the interface we would like:

```golang
type DelayQueue interface {
    // Enqueue should ensure that the `job` becomes ready
    // for dequeue only at `readyTime`.
    Enqueue(readyTime time.Time, job []byte) error

    // Dequeue should return a "ready item" (i.e., An item
    // that was enqueued with `redyTime <= relativeTo`).
    // For example, passing `time.Now()` should return an
    // item that is ready right now.
    Dequeue(relativeTo time.Time) (job []byte, err error)
}
```

## Using a Priority Queue

We could use a priority-queue (like min-heap) and store items with their `readyTime` as the priority. Then, we simply peek the queue to see if the item at the front is ready, prevent dequeue if not.

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

    // The item at the front of the queue is ready.
    return pq.pop()
}
```

This is definitely better than the `sleep_until` model, but this is still in-memory and does not support distributed setup needed for a highly-available system (Unless we can implement a persistent & distributed priority-queue).

## Need for state-storage

When we think about distributed setup for this job-queue, we also run into another major problem: What happens when a worker removes a ready job for processing by doing a `Dequeue()` but crashes immediately? Dequeue would have removed the item from the queue and now it is lost. We cannot leave the item in the queue until successful processing also because other workers might try to claim the same job and process it again.

We need some state storage that we can use to co-ordinate all our workers and achieve the following objectives:

1. If a worker crashes, the overall system should continue working without causing partial/complete downtime
2. A job or set of jobs should NOT be impacted by unavailability of a specific worker.
3. A worker crash should NOT cause data loss (i.e., `at-least-once` guarantee is a must).
4. As the number of jobs grow, we should be able to add new workers easily and the system should somehow distribute the work evenly across all available workers.

## Using Redis

Redis is in-memory, fast, and provides various data structures that we can use. List and sorted-set in particular are very useful for building job queues. For this reason, it is used in many typical job-queue frameworks as well (e.g., gocraft/work) and we will also develop our delayed-queue with Redis in following sections.

### FIFO Queues with Lists

A normal FIFO job-queue can be implemented using list data-structure by combining `LPUSH` &
`BRPOP` commands [^1]. But the problem with this design is that if the worker that picks up
the job crashes, the job is lost because it has been removed from the `jobs` list. To support
recovery, the popped item should be moved to an ongoing list atomically and a recovery process
should reconcile these items somehow if worker crashes. This can be accomplished using the Lua support for example [^2]. But it still lacks the time-constraitnt that we want to support. Sorted-Sets to the rescue!

### Scheduled Queues with Sorted-Sets

A Redis sorted set is a collection of unique strings (members) ordered by an associated score[^6]

#### Enqueue

Let's say we have have a sorted-set named `delay_set`. We can store jobs in this set just like we can with a normal List. But in addition, we can also attach a priority/score to each item. Since we need to order items based on their ready-time and it is monotonically increasing, we can simply use the ready-time itself as the priority and the set gets sorted naturally. So our enqueue will look like:

```python
def enqueue(job, ready_at):
    # seconds since unix epoch
    unix_ts = ready_at.unix_seconds()

    # add the item to 'delay_set' with 'unix_ts' as score.
    # ZADD has O(log(n)) time-complexity.
    redis_call("ZADD", key="delay_set", item=job, score=unix_ts)
```

`ZADD` has `O(log(n))` time-complexity[^5].

Since Redis commands are atomic, multiple clients can do `enqueue()` concurrently without any issues, Great! Now, we need a way to `dequeue()`.

#### Dequeue

Since we are using the ready-time itself as the score of each item in our `delay_set`, we can assume any item in the set that has score less than current-timestamp as ready-job.

So, we can find the ready-jobs by doing a `ZRANGE` with `[-inf, now().unix_seconds()]` as the scan range. For example, assuming the time right now is `1668842026`:

```redis
ZRANGE delay_set -inf 1668842026 BYSCORE
```

This will return (without removing) all items in the set that have scores less than current time (i.e., items that are ready for processing). But this is where it gets a little tricky. As mentioned earlier, `dequeue()` SHOULD NOT remove item from the Redis until it is successfully processed to prevent data loss. On the other hand, it cannot leave it in the set *as-is* also because then other workers will have no way of knowing that job is already being processed. So we need some way to leave the job in Redis, but somehow prevent other workers from "seeing" it as ready. There are many ways to do this ranging from locks to secondary container for ongoing jobs, etc.

An interesting technique that I found out is to keep the item in the set, but simply increase the score of that item by some configurable period. So everytime the worker polls the `delay_set` for jobs, it should

1. Execute `ZRANGE` by-score with `[-inf, now()]` score range.
2. For each item returned, it should increase the score by configurable period (call it `reclaim_ttl`).

This way, once a worker picks up a job, it will not appear in polls done by other workers for sometime. If worker finishes processing successfully, it can acknowledge by removing the item permanently using `ZREM`. If the worker that picked up the job crashes, it's not a problem since other workers will start seeing this job as ready after `reclaim_ttl` is over. One important point to note is that, this technique also implies that the worker that picked up the job has at-most `reclaim_ttl` time to finish and acknowledge it. If not done, it can lead to duplicate processing of the same item. But it's easy to prevent by tuning the `reclaim_ttl` and also by using this time to move the job request to another normal job-queue and acknowledge instead of actually executing business logic. This technique single-handedly resolves most issues we have!

Unfortunately, Redis does not provide a built-in command do these steps atomically. Fortunately though, it does provide Lua support with atomicity guarantees which we can use to build custom atomic commands by combining built-in commands. Nice!

```lua
local from_set = KEYS[1]
local max_score, new_score, batch_sz = ARGV[1], ARGV[2] or 0.0, ARGV[3]

local items = redis.call('ZRANGE', from_set, '-inf', max_score, 'BYSCORE')
for i, value in ipairs(items) do
    redis.call('ZADD', from_set, 'XX', new_score, value)
end
return items
```

The `ZRANGE` has `O(log(N) + M)` complexity [^4] where M is number of items returned. Which can vary from 0 to batch_sz. The loop is `O(M)`, `ZADD` is `O(log N)` [^5] and together is `O(M*log(N))` complexity. So overall, the Lua script itself has O(M*log(N)) complexity (dropping the non-significant part).

Dequeue implementation would simply be to invoke this Lua script now.

```python
def dequeue_all_ready(batch_size=100):
    max_score = now().unix_seconds()
    new_score = max_score + 30 # reclaim_ttl = 30 seconds
    ready_jobs = exec_lua(
        keys=["delay_set"],
        args=[max_score, new_score, batch_size],
    )

    # This loop MUST be finished in 30 seconds. Otherwise, it can
    # lead to duplicate processing.
    for job in ready_jobs:
        # enqueue the job into another normal job-queue or publish
        # to a message-queue (e.g., Kafka), etc.
        execute_job(job)

        # acknowledge by removing the item.
        redis_call("ZREM", "delay_set", job)
```

The `ZREM` command has `O(M*log(N))` [^3]. Effectively, dequeue has `O(M*log(N))` complexity.

Workers can invoke the `dequeue_all_ready()` function repeatedly to keep processing items as they become ready. If a worker from the pool crashes, it makes no difference to overall health of the system because other workers will still continue processing any jobs that are ready. Adding new worker to the pool is also seamless because, the worker starts polling as soon as it starts up without disturbing anything else.

### Sharding

One thing to note is that we are using a single set to store all jobs which might grow signficanlty over time. This can have impact on the RAM & disk requirements of the Redis node. Also, since Redis is single-threaded, at a time, only one worker is actually able to poll the set. If Redis cluster is an option, we can resolve this by sharding the `delay_set`. For example, we can shard the delay_set into 12 shards (e.g., delay_set_0, delay_set_1, etc). Redis will distribute these across nodes/slots in the cluster. During enqueue/dequeue, we can randomly pick a shard and use. This way, chances of multiple workers trying to poll the same key on the same node reduces.

[^1]: https://redis.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-4-task-queues/6-4-1-first-in-first-out-queues/
[^2]: https://redis.io/commands/lmove#pattern-reliable-queue
[^3]: https://redis.io/commands/zrem/
[^4]: https://redis.io/commands/zrange/
[^5]: https://redis.io/commands/zadd/
[^6]: https://redis.io/docs/data-types/sorted-sets/
