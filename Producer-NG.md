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

##### Maintaining Order

Maintaining the order of messages when a retry occurs is an additional challenge. When a `flusher` triggers a retry, the following events occur:
- the messages to retry are sent to the retryHandler
- the flusher sets a flag for the given topic/partition; while this flag is set any further messages for that partition (which may have already been in the pipeline) will be immediately retried
- eventually the first retried message reaches its `leaderDispatcher`
- the leaderDispatcher sends off a special "chaser" message and releases its reference to the old broker
- the leaderDispatcher updates its metadata, opens a connection to the new broker, and sends the retried broker down the new path
- the leaderDispatcher continues handling incoming messages - retried messages get sent to the new broker, while new messages are held in a queue to preserver ordering
- the flusher sees the chaser message; it clears the flag it originally sent, and "retries" the chaser message
- the leaderDispatcher sees the retried chaser message (indicating that it has seen the last retried message)
- the leaderDispatcher flushes the backlog of "new" messages to the new broker and resumes normal processing

### Shutdown

Cleanly shutting down/closing a concurrent application is always an "interesting" problem. With pipelines in go the situation is relatively straightforward except when you have multiple writers on a single channel, in which case you must reference-count them to ensure all writers are closed before the channel is.

In the producer, this pattern occurs in two places:
- multiple partitions can be lead by the same broker; connections to `messageAggregators` are explicitly reference-counted in the `getBrokerWorker` and `unrefBrokerWorker` helper functions
- all `flushers` can write to the `retryHandler`; this is reference-counted by sending ref/unref messages to it from *all* other goroutines, as we must be sure that there are no messages anywhere in the pipeline before it closes