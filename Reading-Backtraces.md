Sarama uses a number of goroutines internally, so when golang dumps a stack trace (hopefully not because Sarama itself has crashed) this page provides a quick guide of what is safe to ignore.

#### Common

- `[select]: github.com/Shopify/sarama.(*client).backgroundMetadataUpdater` is the background goroutine that wakes up every so often (controlled by `Metadata.RefreshFrequency`) and calls `RefreshMetadata()`. This can usually be ignored.

- `[chan receive]: github.com/Shopify/sarama.(*Broker).responseReceiver` indicates a broker connection that is open, but not currently sending or receiving. These can usually be ignored, unless there are more of them than you have brokers.

#### Producer

- `[chan receive]: github.com/Shopify/sarama.(*asyncProducer).topicDispatcher` is the producer's main goroutine waiting for the next message from the user
- `[chan receive]: github.com/Shopify/sarama.(*asyncProducer).partitionDispatcher` is a producer goroutine (one per topic being produced to) for partitioning messages
- `[chan receive]: github.com/Shopify/sarama.(*asyncProducer).leaderDispatcher` is a producer goroutine (one per partition) for associating messages with the correct broker
- `[chan receive]: github.com/Shopify/sarama.(*asyncProducer).retryHandler` is a single producer goroutine which only wakes up to handle message retries