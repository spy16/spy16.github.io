---
title: "Things About Kafka"
date: "2020-02-20T20:20:20Z"
tags:
- til
- kafka
---

I work with Kafka a lot and this post is a summary of some interesting facts about it.

<!--more-->

## Topic & Partitions

* Topic is a logical grouping of partitions. But a partition is the storage unit in Kafka [^1]
* Producer can decide which partition to publish to by explicitly setting the partition on the message.
* If producer does not set or sets `PartitionAny`, round-robin is used starting with an arbitrary partition.
* `offset` is maintained per partition and is a monotonically increasing number starting from 0. (new-id=last-id +1)
* Topic is always an append-only log. Message already published cannot be modified or deleted by the producer.
* Order of messages is guaranteed within a partition but not at topic level.

```shell
# create a topic with 3 partitions and no replication.
$ kafka-topics --create --topic til-kafka  --partitions 3 --replication-factor 1 --zookeeper localhost:2181

# show the metadata for the topic fetched from the bootstrap server.
$ kafkacat -b localhost:9092 -L -t til-kafka
```

## Storage

* Each partition corresponds to a log directory in `log.dirs` path. (i.e., `til-kafka-0`, `til-kafka-1`
  and `til-kafka-2`)
* Each partition directory contains storage segments[^5] each consisting of 3 files:
  * `x.log` - actual record log. append only file.
  * `x.index` - Index for fast access. Maintains record offset <> file offset mapping in `x.log`
  * `x.timeindex` - Index for fast access. Maintains record offset <> timestamp mapping [^2].
* A new segment is created when the current active segment reaches the limit set using `log.segment.bytes`
* New segment name is always the last offset in the old segment + 1 which also means the name reflects the first record
  offset in the segment.
* The newly created segment becomes the active segment, and the writes are done only on the active segment.
* Segments make it easy to enforce retention policy. The segment files with age greater than `log.retention.hours` can
  be simply deleted.

```shell
# examine a log file in partition
$ kafka-run-class kafka.tools.DumpLogSegments --deep-iteration --print-data-log \
  --files logs/til-kafka-0/00000000000000000000.log
```

## Replication

* Each topic has a replication factor that can be at max the number of brokers available at the time of creation of the
  topic.
* Configured number of copies of partitions of the topic are maintained with one acting as leader (accepts reads +
  writes) and others acting as passive followers.
* Kafka ensures only one replica of a topic is maintained on one node to provide **fault-tolerance**.
* Replicas that are no more than `replica.lag.time.max.ms` behind the leader are called `In-Sync Replicas (ISR)`.
* If the leader goes down, one of the ISRs is chosen as the new leader [^3].

## Consumer

* Consumer is an entity that consumes records from partitions of a topic [^4].
* Consumers subscribe to one or more topics and run a `poll()` loop to read messages and get partition assignments.
* Consumer group can be used to process records concurrently. Number of partitions decide how many active consumers can
  exist at a time.
  * If count(consumers) > count(partitions), consumers-partitions number of consumers will be idle.
  * If count(consumers) < count(partitions), some consumers will be assigned more than 1 partition to read from.
* Based on `enable.auto.commit` config, consumers may commit messages automatically at interval set
  by `auto.commit.interval.ms` or a manual `commit()` may be required. With auto-commit enabled, `at-least once`
  delivery semantics apply.
* Following steps can be taken to achieve required delivery semantics:
  1. at-least once: auto-commit enabled OR `poll()`, `process()` and then `commit()`
  2. at-most once: auto-commit false AND `poll()`, `commit()` and then `process()`
* Consumer must process each message within `max.poll.interval.ms` time and invoke the next `poll()` to make sure the
  broker does not end up assuming the consumer to be dead. If this does happen, a partition re-balance will occur
  causing the consumer to be kicked out of the group. In addition to this, consumer must also send heartbeats within
  every `session.timeout.ms` window. If violated, this also causes a re-balance.

## References

[^1]: https://medium.com/@durgaswaroop/a-practical-introduction-to-kafka-storage-internals-d5b544f6925f
[^2]: https://jaceklaskowski.gitbooks.io/apache-kafka/content/kafka-log-TimeIndex.html
[^3]: https://blog.knoldus.com/apache-kafka-topic-partitions-replicas-isr/
[^4]: https://www.confluent.io/blog/tutorial-getting-started-with-the-new-apache-kafka-0-9-consumer-client
[^5]: https://jaceklaskowski.gitbooks.io/apache-kafka/content/kafka-log-LogSegment.html
