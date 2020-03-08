# GRPC - Streaming
In some cases you might need to provide the large portions of data for the consumer. Combine the ability to write custom
Golang GRPC service, Jobs and Broadcast to stream data from PHP application.

> Attention, this article an example implementation. Make sure to implement proper backoff strategy and timeout management
> before going to production. Make sure to read other GRPC articles prior to this section.

## Service Definition
We can define the service as singular endpoint with streaming response. The client will connect the streaming
service and must stop consuming after the null message received. The consuming will be initiated using given `id`.
 
```json
syntax = "proto3";

package stream;

message Request {
    string id = 1;
}

message Data {
    int32 sequence = 1;
    bytes data = 2;
}

service Streamer {
    rpc Stream (Request) returns (stream Data) {
    }
}
``` 
 
Create direction `stream` and generate client and server SDK for Golang using:

```bash
$ mkdir stream
$ protoc -I proto/ proto/stream.proto --go_out=plugins=grpc:strea
```

## Consumer
The client/consumer application will be displaying all streamed content directly into `strdout`. You can create it in a 
separate directory. Copy the `stream` directory and `app.crt` to your client application. 

```bash
$ go mod init client
```

Application will look as following:

```golang 
package main

import (
	stream "client/stream"
	"context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
	"log"
)

func main() {
	creds, err := credentials.NewClientTLSFromFile("app.crt", "")
	if err != nil {
		panic(err)
	}

	conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(creds))
	if err != nil {
		panic(err)
	}
	defer conn.Close()

	str, err := stream.NewStreamerClient(conn).Stream(context.Background(), &stream.Request{Id: "request-id"})
	if err != nil {
		panic(err)
	}

	for {
		data, err := str.Recv()
		if err != nil {
			panic(err)
		}

		if data.Sequence == 0 {
			log.Println("Stream is over")
			return
		}

		log.Printf("Sequence[%v]: %s\n", data.Sequence, string(data.Data))
	}
}
```

## Producer
The producer application will be split into Golang and PHP part. The Golang will route message to backgroud PHP process
using spiral/jobs package and later read the produced response using unique broadcast topic.

The service will look as following:

```golang
package stream

import (
	"encoding/json"
	"github.com/spiral/broadcast"
	"github.com/spiral/jobs"
	grpc "github.com/spiral/php-grpc"
	grpc2 "google.golang.org/grpc"
)

const ID = "stream"

type Message struct {
	SequenceID int    `json:"sequenceID"`
	Data       string `json:"data"`
}

type Service struct {
	queue  *jobs.Service
	pubsub *broadcast.Service
}

func (s *Service) Init(
	g *grpc.Service,
	q *jobs.Service,
	p *broadcast.Service,
) (bool, error) {
	s.queue = q
	s.pubsub = p

	return true, g.AddService(func(server *grpc2.Server) {
		RegisterStreamerServer(server, s)
	})
}

func (s *Service) Stream(r *Request, srv Streamer_StreamServer) error {
	client := s.pubsub.NewClient()
	defer client.Close()

	// use request id as unique topic name
	if err := client.Subscribe(r.Id); err != nil {
		return err
	}

	// start the background producer
	_, err := s.queue.Push(&jobs.Job{
		Job:     "app.job.Produce",
		Payload: `{"requestID":"` + r.Id + `"}`,
		Options: &jobs.Options{},
	})
	if err != nil {
		return err
	}

	// forward data from topic to stream
	for msgData := range client.Channel() {
		msg := &Message{}
		if err := json.Unmarshal(msgData.Payload, msg); err != nil {
			return err
		}

		if err := srv.Send(&Data{
			Sequence: int32(msg.SequenceID),
			Data:     []byte(msg.Data),
		}); err != nil {
			return err
		}
	}

	return nil
}
```

Make sure to register service in `main.go`:

```golang
rr.Container.Register(stream.ID, &stream.Service{})
```

The application will require `jobs` and `broadcast` services enabled (in both `.rr` and application bootloaders). 
You do not need any GRPC workers.

```yaml
grpc:
  listen: tcp://0.0.0.0:50051
  tls.key:  "app.key"
  tls.cert: "app.crt"

jobs:
  dispatch:
    app-job-*.pipeline: "local"
  pipelines:
    local:
      broker: "ephemeral"
  consume: ["local"]
  workers:
    command: "php app.php"
    pool.numWorkers: 2
```

### PHP Job
The PHP Job will be located in `app/src/Job/Produce.php`:

```php
<?php

declare(strict_types=1);

namespace App\Job;

use Spiral\Broadcast\BroadcastInterface;
use Spiral\Broadcast\Message;
use Spiral\Jobs\JobHandler;

class Produce extends JobHandler
{
    public function invoke(string $requestID, BroadcastInterface $broadcast)
    {
        dumprr("Streaming for: {$requestID}");

        for ($i = 0; $i < 100; $i++) {
            usleep(100000);
            $broadcast->publish(new Message($requestID, ['sequenceID' => $i + 1, 'data' => "DATA $i"]));
        }

        // the stream is over
        $broadcast->publish(new Message($requestID, ['sequenceID' => 0, 'data' => null]));
    }
}
```

> You can split the streaming into multiple smaller jobs.

Run server and then client to test the streaming.
