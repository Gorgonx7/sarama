Welcome to the sarama wiki!

Common user documentation is provided via godoc: https://godoc.org/github.com/Shopify/sarama

This wiki holds more architecture and maintainer-oriented information.

### Development

Travis CI will run the following commands for a build:
- `go test`
- `go vet`
- `go fmt`
- `errcheck`

You can run them locally with `make` (run `make install_dependencies` to install the necessary tools).

Sarama includes functional tests that run against a test cluster with a specific topic configuration. A [Vagrantfile](https://www.vagrantup.com/) is included to make it easy to set up a Kafka cluster with 5 brokers and the necessary topics inside a virtual machine. Just run `vagrant up` to set it up. The IP address of the VM is **192.168.100.67**.

- Kafka connection string: `192.168.100.67:6667,192.168.100.67:6668,192.168.100.67:6669,192.168.100.67:6670,192.168.100.67:6671`
- Zookeeper connection string: `192.168.100.67:2181,192.168.100.67:2182,192.168.100.67:2183,192.168.100.67:2184,192.168.100.67:2185`

When writing functional tests, always use the `KAFKA_PEERS` environment variables to discover the cluster. If they are not set, use the default connection string that points to the Vagrant cluster above. (For Zookeeper, use `ZOOKEEPER_PEERS`.)