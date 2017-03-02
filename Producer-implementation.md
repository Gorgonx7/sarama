This page serves to document the broader design of Sarama's producer.

### Message Flow

The producer is effectively based on the [pipeline concurrency pattern](http://blog.golang.org/pipelines) with a few adjustments. In the normal 'success' case, each message flows through four structs before being put on the wire. The structs are arranged in order top-to-bottom in the file `producer.go`, and each one has its own goroutine.

1. `asyncProducer` (singleton): its `dispatch()` goroutine reads from the input channel and dispatches each message according to its topic in a one-to-many (fan-out) pattern.
1. `topicProducer` (one per topic): its `dispatch()` goroutine takes messages for a single topic, partitions them, and dispatches each message according to its partition in a one-to-many (fan-out) pattern.
1. `partitionProducer` (one per partition): its `dispatch()` goroutine takes messages for a single topic & partition, finds the broker leading that partition, and dispatches the messages in a many-to-many pattern (a single broker may lead multiple partitions). The `getBrokerProducer` and `unrefBrokerProducer` helpers are used to safely spawn/close the goroutines associated with each broker.
1. `brokerProducer` (one per broker): its `run()` goroutine takes messages for a single broker and batches them according to the configured size limits, time limits, etc. When a batch is complete it is turned into a request and sent to an anonymous "bridge" goroutine which puts the request on the wire.

### Retrying

When the `brokerProducer` cannot deliver a message due to a cluster leadership change, that message is retried.  A flag is set on each message indicating that it is being retried, and the messages are then sent to the `retryHandler` (an additional singleton goroutine) which puts them back onto the normal input channel.

As this introduces a loop in our pipeline, we must be careful to avoid the obvious deadlock case. To this end, the `retryHandler` goroutine is *always* available to read messages.

The number of retry attempts is configurable as `config.Producer.Retry.Max`. After every retry attempt, the producer will sleep `config.Producer.Retry.Backoff` before trying again.

##### Maintaining Order

Maintaining the order of messages when a retry occurs is an additional challenge. When a `brokerProducer` triggers a retry, the following events occur, strictly in this order:
- the messages to retry are sent to the `retryHandler`
- the `brokerProducer` sets a flag for the given topic/partition; while this flag is set any further such messages (which may have already been in the pipeline) will be immediately sent to the `retryHandler`
- eventually the first retried message reaches its `partitionProducer`
- the `partitionProducer` sends off a special "chaser" message and releases its reference to the old broker
- the `partitionProducer` updates its metadata, opens a connection to the new broker, and sends the retried message down the new path
- the `partitionProducer` continues handling incoming messages - retried messages get sent to the new broker, while new messages are held in a queue to preserver ordering
- the `brokerProducer` sees the chaser message; it clears the flag it originally sent, and "retries" the chaser message
- the `partitionProducer` sees the retried chaser message (indicating that it has seen the last retried message)
- the `partitionProducer` flushes the backlog of "new" messages to the new broker and resumes normal processing

When `n` retries are configured, each `partitionProducer` uses a set of `n` buffers for message backlogs that are being held to preserve order, as well as a set of `n` boolean flags to indicate which 'levels' currently have chaser messages in progress. For indexing simplicity, an `n+1`-length slice of structs is used; the flag at index 0 and the buffer at index `n` go unused. Each `partitionProducer` keeps one additional piece of state: the current high-watermark of retries in progress.

In normal operation (no errors, no retries), the high-watermark is 0, all the flags are unset, and all the buffers are empty. When a message is received there are three possibilities:
- the retry-count is equal to the high-watermark; the message is passed through normally
- the retry-count is greater than the high-watermark; the message is passed through, but before this happens the high-watermark is updated, a chaser is sent and the chaser flag is set
- the retry-count is less than the high-watermark; the message is saved in the appropriate buffer

There is one additional case to consider: when a chaser message is received. If its retry-count is less than the high-watermark then the flag at that index is cleared. Otherwise its retry-count must be equal to the high-watermark, which causes the high-watermark to be decremented
and any buffer at the new high-watermark to be flushed. The decrement-and-flush process is repeated until either a chaser flag is found (indicating the current high-watermark needs to be kept until its chaser is received) or the high-watermark hits 0, at which point normal operation resumes.

### Shutdown

Cleanly shutting down/closing a concurrent application is always an "interesting" problem. With pipelines in go the situation is relatively straightforward except when you have multiple writers on a single channel, in which case you must reference-count them to ensure all writers are closed before the channel is.

In the producer, this pattern occurs in two places:
- multiple partitions can be lead by the same broker; connections to `brokerProducer`s are explicitly reference-counted in the `getBrokerProducer` and `unrefBrokerProducer` helper functions
- all `brokerProducer`s can write to the `retryHandler`; this is reference-counted via the `inFlight` wait-group on the core `asyncProducer` struct