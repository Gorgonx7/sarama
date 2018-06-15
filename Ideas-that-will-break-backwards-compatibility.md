The following ideas cannot be implemented in v1, because they would break backwards compatibility. Listing them here to be revisited when we decide to do a v2. (Just because they're here doesn't necessarily mean they'll be in v2 though).

- `Client.Replicas` should return `[]*Broker`, it is safe to do now we have lazy connections. Or is it enough to just add `Client.Broker(int32) *Broker`?
- Remove `Encoder` interface. It makes the API more complicated to use, and we cannot "stream" the content anyway: we need to send a CRC32 to the broker first which requires buffering the entire value in memory.
- Move Request/Response objects to `protocol` subpackage, and export Encode and Decode methods. Maybe also move producer, consumer, clients to their own (sub)packages so that e.g. a producer binary doesn't have to pull in all the consumer code.
- `ConsumerMetadataResponse` has some deprecated fields (replaced by a single `*Broker`) which can be removed.
- Make backoff values `[]time.Duration` so that exponential and other backoff patterns can be easily specified using e.g. https://godoc.org/github.com/eapache/go-resiliency/retrier#ConstantBackoff and friends
- Use logrus for structured logging? Add log levels to the logger interface?
- The `kafka-console-partitionconsumer` tool can be removed, it is superseded by `kafka-console-consumer`.
- Go lint wants `Id` to be `ID` everywhere, e.g. `GroupId` should be `GroupID` in several protocol fields.
- Make the `Broker` a mockable interface? Or just make the mock broker code public? See e.g. https://github.com/Shopify/sarama/pull/570 for some discussion. Move the now-public mocks to their own package and settle https://github.com/Shopify/sarama/issues/497 once and for all.
- Fix the default signedness of the hash partitioner (https://github.com/Shopify/sarama/issues/1090).
- More hash-partitioner stuff: https://github.com/Shopify/sarama/pull/1118#issuecomment-397661619