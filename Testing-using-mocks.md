Testing is possible using dependency injection. Sarama ships with mock types in the `mocks` subpackage. Those mock types can be used instead of the real sarama types for testing purposes.

### Example: application using the sync producer

Given the following application:

``` go
package main

import (
	"github.com/Shopify/sarama"
	"log"
)

type App struct {
	producer sarama.SyncProducer
}

func NewApp(producer sarama.SyncProducer) *App {
	return &App{producer: producer}
}

func (a *App) DoSomething() bool {
	partition, offset, err := a.producer.SendMessage("topic", nil, sarama.StringEncoder("message"))
	if err != nil {
		log.Printf("FAILED to produce message: %s", err)
		return false
	} else {
		log.Printf("Produced message to partition %d with offset %d", partition, offset)
		return true
	}
}

func (a *App) Close() error {
	return a.producer.Close()
}
```

You can test it as follows:

``` go
package main

import (
	"github.com/Shopify/sarama"
	"github.com/Shopify/sarama/mocks"
	"testing"
)

func TestApp(t *testing.T) {
	mp := mocks.NewSyncProducer(t, nil)

	mp.ExpectSendMessageAndSucceed()
	mp.ExpectSendMessageAndFail(sarama.ErrOutOfBrokers)

	a := NewApp(mp)
	defer a.Close()

	if a.DoSomething() != true {
		t.Error("Expected to return true if the message is produced successfully")
	}

	if b.DoSomething() != false {
		t.Error("Expected to return false if the message fails to produce")
	}
}
```