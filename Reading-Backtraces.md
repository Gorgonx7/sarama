Sarama uses a number of goroutines internally, so when golang dumps a stack trace (hopefully not because Sarama itself has crashed) this page provides a quick guide of what is safe to ignore.

#### Common

- `[select]: github.com/Shopify/sarama.(*client).backgroundMetadataUpdater` is the background goroutine that wakes up every so often (controlled by `Metadata.RefreshFrequency`) and calls `RefreshMetadata()`. This can usually be ignored.

- `[chan receive]: github.com/Shopify/sarama.(*Broker).responseReceiver` indicates a broker connection that is open, but not currently sending or receiving. These can usually be ignored, unless there are more of them than you have brokers.

#### Producer

- `[chan receive]: github.com/Shopify/sarama.(*asyncProducer).dispatcher` is the producer's main goroutine waiting for the next message from the user
- `[chan receive]: github.com/Shopify/sarama.(*topicProducer).dispatch` is a producer goroutine (one per topic being produced to) for partitioning messages
- `[chan receive]: github.com/Shopify/sarama.(*partitionProducer).dispatch` is a producer goroutine (one per partition) for associating messages with the correct broker
- `[select]: github.com/Shopify/sarama.(*brokerProducer).run` is a producer goroutine (one per broker) for collecting messages and constructing/sending produce requests
- `[chan receive]: github.com/Shopify/sarama.(*asyncProducer).retryHandler` is a single producer goroutine which only wakes up to handle message retries

#### Consumer

- `[chan receive]: github.com/Shopify/sarama.(*partitionConsumer).dispatcher` is a consumer goroutine (one per partition being consumed) which handles associating partitions with their brokers and is usually quiet
- `[chan receive]: github.com/Shopify/sarama.(*partitionConsumer).responseFeeder` is a consumer goroutine (one per partition being consumed) which handles feeding messages to the user
- `[select]: github.com/Shopify/sarama.(*brokerConsumer).subscriptionManager` is a consumer goroutine (one per broker) which manages subscriptions for consuming from individual partitions and is usually quiet
-  `[chan receive]: github.com/Shopify/sarama.(*brokerConsumer).subscriptionConsumer` is a consumer goroutine (one per broker) which actually sends fetch requests and handles the responses