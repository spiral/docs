# Broadcast - Publish and Consume
The broadcast component provides the ability to publish and consume messages (events) from one or multiple topics. The
default broadcast implementation, can not be treated as persistent and durable, meaning you have to implement error handling
and timeout strategy by yourself. 

> Try to use broadcast extension for informative messages rather than control commands. Make sure to properly handle
> timeouts and 2PC failures in alternative approaches.

## Publish from PHP
To publish message(s) into given topic use `Spiral/Broadcast/BroadcastInterface`. Each message must be wrapped using
`Spiral\Broadcast\Message` object.

```php
use Spiral\Broadcast\BroadcastInterface;
use Spiral\Broadcast\Message;

class HomeController
{
    public function index(BroadcastInterface $broadcast)
    {
        $broadcast->publish(
            new Message('topic', 'message'),
            new Message('topic2', ['key' => 'value'])
        );
    }
}
```

In your message, you must specify the topic and the payload to be published. The payload can contain any json serializable
value.

> You can consume your messages using [WebSockets extension](/broadcast/websockets.md).

## Publish from Golang
Is is possible to use Golang SDK to publish messages. Create your service as described [here](/cookbook/golang-library.md)
or [here](https://roadrunner.dev/docs/beep-beep-service).

> We are going to create service to publish message to specified topic every second.

To access broadcast functionality request the `github.com/spiral/broadcast` dependency and instance of `*broadcast.Service`:

```golang
package demo

import (
	"github.com/spiral/broadcast"
	"time"
)

type Service struct {
	pubsub *broadcast.Service
	close  chan interface{}
}

func (s *Service) Init(pubsub *broadcast.Service) (bool, error) {
	s.pubsub = pubsub
	return true, nil
}

func (s *Service) Serve() error {
	s.close = make(chan interface{})

	client := s.pubsub.NewClient()
	ticker := time.NewTicker(time.Second)

	go func() {
		for {
			select {
			case <-s.close:
				return
			case <-ticker.C:
				client.Publish(&broadcast.Message{
					Topic:   "topic",
					Payload: []byte(`"hello world"`), // json compatible
				})
			}
		}
	}()

	<-s.close
	return nil
}

func (s *Service) Stop() {
	close(s.close)
}
```

> Read how to consume this messages below.

## Consume from Golang
Subscribe and consume messages using Golang SDK. Modify `Serve` method to post all received messages from `topic` into stdout.

```golang
func (s *Service) Serve() error {
	s.close = make(chan interface{})

	client := s.pubsub.NewClient()
	ticker := time.NewTicker(time.Second)

	go func() {
		for {
			select {
			case <-s.close:
				return
			case <-ticker.C:
				client.Publish(&broadcast.Message{
					Topic:   "topic",
					Payload: []byte(`"hello world"`), // json compatible
				})
			}
		}
	}()

	client.Subscribe("topic")
	messages := client.Channel()

	go func() {
		for {
			select {
			case <-s.close:
				return
			case msg := <-messages:
				log.Printf("topic %s, body %s", msg.Topic, string(msg.Payload))
			}
		}
	}()

	<-s.close
	return nil
}
```

> Spiral does not provide the default way to consume messages from PHP due to the blocking nature of PHP execution. Read 
> about [Streaming and Batch processing](/grpc/streaming.md) to understand how to bypass such limitation.
