# GRPC - Golang Services
You can combine the PHP and Golang GRPC services within one application. 

> Make sure to read how to [build application server](/framework/application-server.md).

To demonstrate the ability to register Golang GRPC service define the proto file with needed agreements:

```json
syntax = "proto3";

package multiplier;

message Mult {
    int32 a = 1;
    int32 b = 2;
}

message MultResult {
    int32 result = 1;
}

service Multiplier {
    rpc Mult (multiplier.Mult) returns (multiplier.MultResult) {
    }
}
```

Generate the server code in `multiplier` folder:

```bash
$ mkdir multiplier
$ protoc -I proto/ proto/multiplier.proto --go_out=plugins=grpc:multiplier
```

> Make sure to install `protoc-gen-go`. Read about installation [here](/grpc/client.md).

## Create Service
Create RoadRunner compatible service in the `multiplier` directory. Register GRPC service using the `*grpc.Service`->`AddService`:

```golang
package multiplier

import (
	"context"
	grpc "github.com/spiral/php-grpc"
	grpc2 "google.golang.org/grpc"
)

const ID = "multiplier"

type Service struct {
}

func (s *Service) Init(g *grpc.Service) (bool, error) {
	return true, g.AddService(func(server *grpc2.Server) {
		RegisterMultiplierServer(server, s)
	})
}

func (s *Service) Mult(ctx context.Context, in *Mult) (*MultResult, error) {
	return &MultResult{Result: in.A * in.B}, nil
}
```

To activate the service, make sure to create and build a custom application server: 

```bash
$ go mod init demo
$ go get google.golang.org/grpc
```

Make sure to request proper dependencies in `go.mod`:

```json
module demo

require (
	github.com/golang/protobuf v1.3.2
	github.com/spiral/broadcast v0.0.0-20191206140608-766959683e74
	github.com/spiral/broadcast-ws v1.1.0
	github.com/spiral/jobs v2.1.3
	github.com/spiral/php-grpc v1.2.0
	github.com/spiral/roadrunner v1.6.2
	google.golang.org/genproto v0.0.0-20181016170114-94acd270e44e
	google.golang.org/grpc v1.21.0
)
```

Modify the `main.go` to activate the service:


```golang
package main

import (
	"demo/multiplier"

	rr "github.com/spiral/roadrunner/cmd/rr/cmd"

	// services (plugins)
	"github.com/spiral/broadcast"
	"github.com/spiral/broadcast-ws"
	"github.com/spiral/jobs/v2"
	"github.com/spiral/php-grpc"
	"github.com/spiral/roadrunner/service/env"
	"github.com/spiral/roadrunner/service/headers"
	"github.com/spiral/roadrunner/service/health"
	"github.com/spiral/roadrunner/service/http"
	"github.com/spiral/roadrunner/service/limit"
	"github.com/spiral/roadrunner/service/metrics"
	"github.com/spiral/roadrunner/service/reload"
	"github.com/spiral/roadrunner/service/rpc"
	"github.com/spiral/roadrunner/service/static"

	// queue brokers
	"github.com/spiral/jobs/v2/broker/amqp"
	"github.com/spiral/jobs/v2/broker/beanstalk"
	"github.com/spiral/jobs/v2/broker/ephemeral"
	"github.com/spiral/jobs/v2/broker/sqs"

	// additional commands and debug handlers
	_ "github.com/spiral/broadcast-ws/cmd/rr-ws/ws"
	_ "github.com/spiral/jobs/v2/cmd/rr-jobs/jobs"
	_ "github.com/spiral/php-grpc/cmd/rr-grpc/grpc"
	_ "github.com/spiral/roadrunner/cmd/rr/http"
	_ "github.com/spiral/roadrunner/cmd/rr/limit"
)

func main() {
	rr.Container.Register(env.ID, &env.Service{})
	rr.Container.Register(rpc.ID, &rpc.Service{})

	// http
	rr.Container.Register(http.ID, &http.Service{})
	rr.Container.Register(headers.ID, &headers.Service{})
	rr.Container.Register(static.ID, &static.Service{})

	rr.Container.Register(grpc.ID, &grpc.Service{})

	rr.Container.Register(jobs.ID, &jobs.Service{
		Brokers: map[string]jobs.Broker{
			"amqp":      &amqp.Broker{},
			"ephemeral": &ephemeral.Broker{},
			"beanstalk": &beanstalk.Broker{},
			"sqs":       &sqs.Broker{},
		},
	})

	// pub-sub
	rr.Container.Register(broadcast.ID, &broadcast.Service{})
	rr.Container.Register(ws.ID, &ws.Service{})

	// supervisor and metrics
	rr.Container.Register(limit.ID, &limit.Service{})
	rr.Container.Register(metrics.ID, &metrics.Service{})
	rr.Container.Register(health.ID, &health.Service{})

	// auto reloading
	rr.Container.Register(reload.ID, &reload.Service{})

	rr.Container.Register(multiplier.ID, &multiplier.Service{})

	// you can register additional commands using cmd.CLI
	rr.Execute()
}
```

> Read more how to define services [here](/cookbook/golang-library.md) and [here](https://roadrunner.dev/docs/beep-beep-service).

You can run the server now:

```bash
$ go run main.go serve -v -d
```

> Read [here](/grpc/streaming.md) how to implement streaming and batch processing using a hybrid PHP/Go approach.
