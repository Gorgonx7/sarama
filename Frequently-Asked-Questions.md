## General

#### Is Sarama used in production anywhere?

Yes. At the time of writing (June, 2017) the following companies have confirmed they are using Sarama: [Shopify](https://www.shopify.com/), [IBM](https://www.ibm.com/), [Heroku](https://www.heroku.com/), [VividCortex](https://www.vividcortex.com/), [Atomx](https://www.atomx.com/).

I understand that there are other major companies using it in production too. If you work for such a company and file a ticket to let me know, I'll list it here.

#### How do I fetch an offset for a specific timestamp?

Use `Client.GetOffset()`.

Starting in Sarama v1.11.0, this method will automatically use Kafka's new precise timestamp lookup if your Kafka version (as provided by `Config.Version`) is at least 0.10.1. Otherwise, the method will fall back to the old API which returns only a *very* approximate value.

#### Does Sarama support [Kafka Streams](https://kafka.apache.org/documentation/streams/)?

No. Kafka Streams is a separate library for stream processing backed by Kafka. It is not a feature of Kafka proper and not something that Sarama intends to support. If somebody wanted to implement streams on top of Sarama, it should live in a separate package.

#### Why can't Sarama connect to my Kafka cluster using SSL?

SSL *is* supported. There are a couple of potential causes for SSL connection problems, but if everything else is working (you can connect on the non-SSL port, and other SSL clients can establish a connection) then chances are you have a cipher problem. When creating the keystore on the server, you need to pass the `-keyalg RSA` argument or else the Kafka broker will operate using an extremely limited set of ciphers, none of which are supported by Golang. See [#643](https://github.com/Shopify/sarama/issues/643) for more details.

## Consuming

#### How can I use Sarama to monitor or join a consumer group?

Consumer-groups are complicated, so this logic lives in a separate (unaffiliated) project which builds on top of Sarama. For Kafka-based tracking (Kafka 0.9 and later), use https://github.com/bsm/sarama-cluster. For Zookeeper-based tracking (Kafka 0.8.2 and earlier), use https://github.com/wvanbergen/kafka.

For issues with these libraries, please file a ticket in their respective repositories, not with Sarama.

#### Why am I getting a `nil` message from the Sarama consumer?

Sarama will never put a `nil` message on the channel. If you are getting `nil`s then the channel has been closed (a receive from a closed channel returns the zero value for that type, which is `nil` for pointers).

The channel will normally be closed only when the PartitionConsumer has been shut down, either by calling `Close` or `AsyncClose` on it. However, the PartitionConsumer may sometimes shut itself down when it encounters an unrecoverable error; in this case you will get an appropriate error on the `Errors` channel, and it will also log a message to `sarama.Logger` that looks something like: `consumer/TOPIC/PARTITION shutting down because REASON`.

#### How do I consume until the end of a partition?

You don't. Kafka doesn't have the concept of a topic or partition "ending"; it was designed for systems where new messages are constantly arriving and so the question is meaningless. There are, however, a few ways to approximate this, both with some obvious drawbacks:
- Wait for the configured `Consumer.MaxWaitTime` time to pass, and if no messages have arrived within that time then assume you're done.
- Use `Client.GetOffset` to get the last offset when you start, and consume up until that offset is reached. 

#### Why is the consumer leaking memory and/or goroutines?

It's not. First, Go uses a garbage-collector, so most memory will be automatically reclaimed unless there's a reference to it hanging around somewhere by accident. Second, the structure of the consumer is remarkably simple and there's basically nowhere for an incorrect reference to sneak in.

Common explanations for what looks like a leak in the consumer but isn't:
- Go's garbage collector returns memory to the Operating System very rarely, preferring to reuse it internally first. This means that external memory measurements may not go down immediately even when the memory is no longer used.
- If you have a lot of memory to spare, Go will use it aggressively for performance. Just because your program is currently using many GiB of memory, doesn't mean it won't run just fine on a more constrained system.
- The consumer does use a *lot* of goroutines, but not an unbounded number. Expect a goroutine for every partition you are consuming plus a few on top; this can be thousands for a process consuming many topics each with many partitions.
- The leak is in your code instead: if you have code that permanently stores a reference to the `ConsumerMessage`s coming out of the consumer, that memory will effectively leak and the profiler will point to the location in Sarama where it was allocated.

If you're going to file a ticket claiming there's a leak in the consumer, please provide a detailed profile and analysis of exactly where/how the leak is occurring.

## Producing

#### How fast is Sarama's producer?

Pretty fast, although it depends heavily on your specific configuration, network, hardware, etc. [This benchmark](http://bravenewgeek.com/dissecting-message-queues/) is a few years old now but claims to have been able to produce nearly 100k messages per second. If you need a more concrete guarantee for a given scenario, you'll have to run your own benchmark.

#### Does Sarama's producer guarantee message ordering?

Yes. Like Kafka, Sarama guarantees message order consistency only within a given partition. Order is preserved even when there are network hiccups and certain messages have to be retried.

#### Does Sarama's producer guarantee that successful messages are contiguous?

No. If five messages (1,2,3,4,5) are fed to the producer in sequence, it is possible (though rare) that the Kafka cluster will end up with messages 1, 2, 4, and 5 in order, but not message 3. In this case you will receive an error for message 3 on the `Errors()` channel.

#### Is it safe to send messages to the synchronous producer concurrently from multiple goroutines?

Yes it is, and even encouraged in order to get higher throughput.