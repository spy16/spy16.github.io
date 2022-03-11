---
title: "Building a distributed cron"
date: 2022-03-03T11:01:59+05:30
draft: false
summary: |
    I was constantly running into use-cases where things needed to be done
    as per a pre-defined schedule. This post summarises the problem and solution.
tags: [redis, scheduler, cron]
no_toc: false
---

## The Problem 

At work, I was constantly running into use-cases where things needed to be done
at a pre-defined time in the future and sometimes in a recurring manner (Similar to 
unix cron[^1]). For example, sending a reminder notification after 2 days, executing 
some job for a user at configured interval, etc.

Considering the execution logic is different in every use-case, it made sense to think
of this problem in a generic sense reduce its overall requirement to just *emitting
events based on schedules*.

Let's say a schedule looks like this:

```golang
type Schedule struct {
    ID      string `yaml:"id"`
    Crontab string `yaml:"crontab"`
    Payload string `yaml:"payload"`
}
```

An example schedule:

```yaml
id: "schedule1"
crontab: "@every 1h"
payload: '{"user": 1234, "type": "renewal_reminder"}'
```

If the above schedule is created at `t`, we expect the scheduler to publish messages (e.g., over Kafka) with the following data at `t1=t+1h`, `t2=t1+1h`, and so on: 

```yaml
id: "schedule1-<tx>"
generated_at: <tx>
schedule_id: "schedule1"
payload: '{"user": 1234, "type": "renewal_reminder"}'
```

Systems interested in these schedules would consume these messages and execute the business logic based on `payload`.

So in summary: *We needed a system that generates events based on schedules, has APIs to manage (i.e., CRUD) those schedules, is horizontally scalable, is highly-available (HA) & has no Single Point of Failure (SPOF).*

{{<figure src="/scheduler.png">}}

## The Solution

In case of never-ending recurring crontabs like `@every 1h`, all execution points cannot be inserted at the time of schedule creation because there are infinite execution points. So this will have to be done lazily - i.e., after handling one execution point, compute the next and enqueue it.

If we think of each crontab as series of timed execution points, what we really need is a job-queue where jobs are ready for dequeue at a specific time instead of being ready as soon as enqueued -- A delay-queue.

So, this is rougly the interface we need to implement. 

```golang
type DelayQ interface {
    // Delay should enqueue item and ensure it becomes ready at
    // the given 'readyTime'.
    Delay(readyTime time.Time, data []byte) error

    // Run should continously look for ready items and invoke 
    // apply for each. This includes any item that was enqueued
    // with readyTime <= time.Now().
    Run(apply HandlerFn) error
}

// HandlerFn is invoked by DelayQ for every ready item.
type HandlerFn func(t time.Time, data []byte) error
```

For our scheduler use-case, the `HandlerFn` should:

1. Publish the message with payload from the schedule.
2. Compute the next execution time and enqueue a job to be ready at that time.

There are obviously multiple approaches to implementing the `DelayQ` interface ranging from in-memory priority-queue based solutions to distributed database solutions. I discuss implementing it with Redis in another post. For the purpose of this discussion, we will assume a `DelayQ` implementation exists and an instance is available as `dq`. We will also assume, there is a schedule definition storage that can be accessed using `saveSchedule`, `fetchSchedule`, etc.

With all this introduction and assumptions, we are ready to get to the workings of the system.

### Creating

We can create a schedule using:

```golang
func createSchedule(sc Schedule) error {
    // enqueue the first execution point into delay-queue so
    // that we get a callback at that time.
    computeRelativeTo := time.Now()
    nextExecutionAt := ComputeNext(computeRelativeTo, sc.Crontab)
    err := dq.Delay(nextExecutionAt, sc.ID)
    if err != nil {
        return err
    }

    // store into some persistent storage.
    // this is done after enqueue to make sure we don't end up
    // with schedules that were not enqueued properly and retry
    // is also not possible.
    return saveSchedule(sc)
}
```

### Updating

Updating can be bit tricky if changing the `crontab` of an active schedule is allowed. Because updating the crontab changes the timeline of the schedule and most-likely makes the already inserted execution points in `DelayQ` invalid. 

One way to handle this is to maintain a version on the schedule which changes on every update. This version should also be stored in the delay-queue as part of `data`. When it becomes ready, the `handle` function can compare the version from the `data` passed to the current version of the schedule-definition. If they do not match, ignore the callback.

### Deleting

Deletion is as simple as removing the schedule definition from the storage. We don't really need to do anything about the execution point added to the delay-queue because the `handle` function ignores the callback if schedule is not found.


### Execution

So the worker setup would look like this:

```golang
package main

func main() {
    // launch the worker threads that continously look for ready-items
    // & invoke handler for those. 
    if err := dq.Run(handler); err != nil {
        log.Fatalf("delayq workers exited: %v", err)
    }
}

func handle(t time.Time, data []byte) error {
    scheduleID := string(data)
    scheduleDef, found := fetchSchedule(scheduleID)
    if !found {
        // the schedule was probably deleted. so nothing to do.
        return nil
    }

    // next execution point should be computed relative to current
    // execution to establish the correct timeline that matches the
    // crontab.
    nextAt := ComputeNext(t, scheduleDef.Crontab)
    err := dq.Delay(nextAt, scheduleID)
    if err != nil {
        return err
    }

    // if enqueue is successful, we publish the event to notify external
    // systems. if this fails, the DelayQ will end-up retrying.
    return publishEvent(Event{
        ID: fmt.Sprintf("%s-%s", scheduleID, t.Unix()),
        Payload: scheduleDef.Payload,
        ScheduleID: scheduleID,
        GeneratedAt: t,
    })
}
```

With this setup running, we should start seeing events being published as expected by the system for any active schedules.

[^1]: https://man7.org/linux/man-pages/man8/cron.8.html
