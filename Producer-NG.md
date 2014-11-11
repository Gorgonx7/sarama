"Producer-NG" is the name of a total redesign and rewrite of sarama's producer logic, in [Pull Request #132](https://github.com/Shopify/sarama/pull/132) (not yet quite merged, though it will be soon). This page will serve to document its broader design.

### Message Flow

The producer is effectively based on the [pipeline concurrency pattern](http://blog.golang.org/pipelines) with a few adjustments. In the normal 'success' case, each message flows through five goroutines before being put on the wire. Each goroutine consists of a single function, and the functions are arranged in order top-to-bottom in the file `producer.go`. These goroutines are:

1. `topicDispatcher` (singleton): reads from the input channel and dispatches each message according to its topic in a one-to-many (fan-out) pattern.
1. `partitionDispatcher` (one per topic): takes messages for a single topic, partitions them, and dispatches each message according to is partition in a one-to-many (fan-out) pattern.
1. `leaderDispatcher` (one per partition): takes messages for a single topic & partition, finds the broker leading that partition, and dispatches the messages in a many-to-many pattern (a single broker may lead multiple partitions). The `getBrokerWorker` and `unrefBrokerWorker` helpers are used to safely spawn/close the goroutines associated with each broker.
1. `messageAggregator` (one per broker): takes messages for a single broker and batches them according to the configured size limits, time limits, etc. When a batch is complete it is sent to the flusher
1. `flusher` (one per broker): takes message batches from the messageAggregator, builds them into actual kafka requests (using the `buildRequest` helper) and puts them on the wire

### Retrying

When the flusher cannot deliver a message due to a cluster leadership change, that message is automatically retried at most once. A flag is set on each message indicating that it is being retried, and the messages are then flushed to the `retryHandler` (an additional singleton goroutine) which puts them back into the normal input channel.

As this introduces a loop in our pipeline, we must be careful to avoid the obvious deadlock case. To this end, the `retryHandler` goroutine is *always* available to read messages.