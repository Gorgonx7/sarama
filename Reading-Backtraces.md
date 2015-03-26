Sarama uses a number of goroutines internally, so when golang dumps a stack trace (hopefully not because Sarama itself has crashed) this page provides a quick guide of what is safe to ignore.

#### Common

- `[select]: github.com/Shopify/sarama.(*client).backgroundMetadataUpdater` is the background goroutine that wakes up every so often (controlled by `Metadata.RefreshFrequency`) and calls `RefreshMetadata()`.

- `[chan receive]: github.com/Shopify/sarama.(*Broker).responseReceiver` indicates a broker connection that is open, but not currently sending or receiving.