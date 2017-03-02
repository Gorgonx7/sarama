## General

#### Is Sarama used in production anywhere?

Yes. At the time of writing (March, 2017) Shopify uses Sarama heavily. I understand that there are other major companies using it in production too. If you work for such a company and file a ticket to let me know, I'll list it here.

## Consuming

#### How can I use Sarama to join a consumer group?

Consumer-groups are complicated, so this logic lives in a separate (unaffiliated) project which builds on top of Sarama. For Kafka-based tracking (Kafka 0.9 and later), use https://github.com/bsm/sarama-cluster. For Zookeeper-based tracking (Kafka 0.8.2 and earlier), use https://github.com/wvanbergen/kafka.

For issues with these libraries, please file a ticket in their respective repositories, not with Sarama.

## Producing

#### How fast is Sarama's producer?

Pretty fast, although it depends heavily on your specific configuration, network, hardware, etc. [This benchmark](http://bravenewgeek.com/dissecting-message-queues/) is a few years old now but claims to have been able to produce nearly 100k messages per second. If you need a more concrete guarantee for a given scenario, you'll have to run your own benchmark.

#### Does Sarama's producer guarantee message ordering?

Yes. Like Kafka, Sarama guarantees message order consistency only within a given partition. Order is preserved even when there are network hiccups and certain messages have to be retried.

#### Does Sarama's producer guarantee that successful messages are contiguous?

No. If five messages (1,2,3,4,5) are fed to the producer in sequence, it is possible (though rare) that the Kafka cluster will end up with messages 1, 2, 4, and 5 in order, but not message 3. In this case you will receive an error for message 3 on the `Errors()` channel.

#### Is it safe to send messages to the synchronous producer concurrently from multiple goroutines?

Yes it is, and even encouraged in order to get higher throughput.