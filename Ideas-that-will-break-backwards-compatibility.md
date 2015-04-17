The following ideas cannot be implemented in v1, because it would break backwards compatibility. Maybe in v2? Caution: these ideas may not be any good, and should be revisited once we're ready for v2

- `Client.Replicas` should return `[]*Broker`. Is safe now we have lazy connections. Or is it enough to just add `Client.Broker(int32) *Broker`?
- Remove `Encoder` interface. It complicates things, and we cannot "stream" the content anyway because we need to send a CRC32 to the broker first. This means we have to buffer the value on memory anyway.
- Move Request/Response objects to `protocol` subpackage, and export Encode and Decode methods. Maybe also move producer, consumer, clients to their own (sub)packages so that e.g. a producer binary doesn't have to pull in all the consumer code.
- `ConsumerMetadataResponse` has some deprecated fields (replaced by a single `*Broker`) which can be removed.
- Make backoff values `[]time.Duration` so that exponential and other backoff patterns can be easily specified using e.g. https://godoc.org/github.com/eapache/go-resiliency/retrier#ConstantBackoff and friends