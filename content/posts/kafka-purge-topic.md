---
title: "Truncating a Kafka topic"
date: 2021-04-02T20:43:28+02:00
draft: true
---
Sometimes, you want to get rid of messages from a Kafka topic. Perhaps you have consumers that always start from the beginning, or you realize the whole topic is full of crap messages. In theory, you can easily delete a Kafka topic like so:
```bash
$ kafka-topic.sh --boostrap-server kafka:9092 --topic some-topic --delete
```
However, this is quite disruptive and is likely to fail both producers and consumers on this topic even if you quickly recreate the topic. There is no telling what consumer groups tracking their own offsets or having custom rebalancing logic might do.

Instead, the best way to truncate a Kafka topic is to temporarily set that topic's retention time (`rentention.ms` config variable) to a low value and let Kafka purge all messages in the topic. This will not affect a running consumer group, so long as the retention time is higher than its time lag plus its processing time. Newly arriving consumer groups will see only messages that arrive after the truncate.

Messages are deleted from the topic by the **log cleaner**. It wakes up every `log.retention.check.interval.ms` (which defaults to 5 minutes) and looks through the topics to see what needs to be cleaned up. This operation is performed independently by each cluster member, each of which maintains its own log cleaner, so messages will disappear from some partitions earlier than others.

The retention time can be changed runtime (as of Kafka 2.4, `kafka-config.sh` is one of those commands that still require Zookeeper access):
```bash
$ kafka-configs.sh --zookeeper zookeeper:2181 --alter \
    --entity-type topics \
    --entity-name some-topic \
    --add-config retention.ms=1000
```
Since the log cleaner runs periodically, you have to wait for a bit before data is actually deleted. When it runs, you will see the following in the log:
```
[2021-03-31 19:43:15,739] INFO [Log partition=some-topic-0, dir=/kafka/kafka-logs-906aad843b03] Rolled new log segment at offset 216 in 3 ms. (kafka.log.Log)
[2021-03-31 19:43:15,739] INFO [Log partition=some-topic-0, dir=/kafka/kafka-logs-906aad843b03] Scheduling log segment [baseOffset 200, size 1552] for deletion. (kafka.log.Log)
[2021-03-31 19:43:15,740] INFO [Log partition=some-topic-0, dir=/kafka/kafka-logs-906aad843b03] Incrementing log start offset to 216 (kafka.log.Log)
[2021-03-31 19:44:15,740] INFO [Log partition=testo-0, dir=/kafka/kafka-logs-906aad843b03] Deleting segment 200 (kafka.log.Log)
```

So what value should you pick for `retention.ms`? You can of course set it to 1 ms, but keep in mind that a consumer typically has a window between poll operations in which it processes messages that it got from the latest poll. If the log cleaner deletes messages that arrived after consumer's latest poll, data loss may occur. In the best of worlds, you have metrics on your consumer processing time and can estimate a good temporary retention time.

Once all partitions have been truncated, you can remove (or reset) the topic retention time. Since there is a small risk of data loss during subsequent wakeups of the log cleaner, you should switch back as soon as possible after the log cleaner has run.
```sh
kafka-configs.sh --zookeeper zookeeper:2181 --alter \
    --entity-type topics \
    --entity-name some-topic \
    --delete-config retention.ms
```