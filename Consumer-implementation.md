This page serves to document the broader design of Sarama's consumer.

### Overview

There are three primary structs: `Consumer`, `PartitionConsumer`, and `brokerConsumer`.

### Consumer

The `Consumer` is a simple structure that does nothing but store state that has to be shared between the many instances of the other two structs. It has no goroutines. The primary public method is `ConsumePartition` which spawns a new `PartitionConsumer`.

### PartitionConsumer

`PartitionConsumer`s are spawned by a `Consumer` and are responsible for consuming from a single topic/partition. They store state specific to that partition, and manage one goroutine (`dispatcher`).

The dispatcher is only responsible for associating the `PartitionConsumer` with a broker, so it is idle in normal operation. It only activates when leadership changes in the cluster and the `PartitionConsumer` must change brokers.

### brokerConsumer

`consumerWorker`s work consuming from a single broker, and manage the set of `PartitionConsumers` that are currently led by their broker. They manage two goroutines: `subscriptionConsumer` which loops fetching from the broker, and `subscriptionManager` which serves to collect any new `PartitionConsumers` while `subscriptionConsumer` is blocked in network IO.