## Getting started with Sarama

This document will help you get started using Sarama in your golang application. This runbook will show you how to set a basic client that will send messages to and from a kafka running in your local environment.

### Setting up Kafka Locally (optional)
If you don't have an instance of kafka running elsewhere you can use a local version of kafka, there are many guides to do this, the following instructions are for a Mac using brew, but the same principle would apply to linux and windows distributions.

1. Run `brew cask install java` this will install java to your local system
1. Run `brew install kafka` this installs both zookeeper and kafka to your local system
1. Start Zookeeper using `brew service start zookeeper` this should start zookeeper running locally 2181
1. Start Kafka using `brew service start kafka` this should start a instance of kafka running locally on port 9092
1. Create a kafka topic using `kafka-topics --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test_topic`
1. Now create two terminals as we are going to test the produce and consume of our kafka, 
    1. In one terminal run `kafka-console-consumer --bootstrap-server localhost:9092 --topic test-topic --from-beginning`
    1. In the other terminal run `kafka-console-producer --broker-list localhost:9092 --topic test-topic` then type a message such as `hello, world` and this should appear in your first terminal running the consumer

After running all these steps you should have an instance of kafka running on your local machine



