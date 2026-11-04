# GRPC — Client

In the previous part of this [article](./service.md), we showed you how to create a gRPC service `Pinger` in PHP with
Spiral and `spiral/roadrunner-bridge` package. In this part, we will show you how to create a client for communication with gRPC services.

## PHP client

The `spiral/roadrunner-bridge` package includes the `spiral/grpc-client` package by default, which provides a simple and efficient way to communicate with gRPC services.

> **Note**
> Here you can find the [installation instructions](./configuration.md) for the `protobuf` PHP extension and the code
> generation tools: `buf`, `protoc` and `protoc-gen-php-grpc`.

### 1. Generate PHP interfaces from the `.proto` file

To create the client, we need to generate interfaces from the `.proto` file. You can follow the instructions in the [previous part](./service.md) to generate these interfaces.

### 2. Configure the gRPC client

After generating the interfaces, you need to configure the gRPC client in the `app/config/grpc.php` file. The configuration includes:

```php
return [
    // File 'app/config/grpc.php'
    // ... other options ...

    'client' => new GrpcClientConfig(
        interceptors: [
            SetTimoutInterceptor::createConfig(6_000),
            RetryInterceptor::createConfig(
                maximumAttempts: 1,
            ),
            ExecuteServiceInterceptors::class,
        ],
        services: [
            new \Spiral\Grpc\Client\Config\ServiceConfig(
                connections: ConnectionConfig::createInsecure('localhost:9001'),
                interfaces: [
                    \GRPC\Pinger\PingerInterface::class,
                ],
            ),
        ],
    ),
];
```

In this configuration:
- `interceptors` - global interceptors that will be applied to all gRPC client requests
- `services` - list of gRPC services with their connection details and interfaces

The `spiral/grpc-client` package will automatically generate proxy classes for the interfaces at runtime, so you don't need to implement them manually.

### 3. Client usage

You can now inject the gRPC client interface directly into your code and use it to call the service methods:

```php app/src/Endpoint/Console/PingServiceCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Question;
use GRPC\Pinger\PingerInterface;
use Spiral\Console\Command;
use GRPC\Pinger\PingRequest;
use Spiral\Grpc\Client\Exception\GrpcClientException;
use Spiral\RoadRunner\GRPC\Context;

#[AsCommand(name: 'ping')]
final class PingCommand extends Command
{
    #[Argument(description: 'URL to ping')]
    #[Question(question: 'Provide URL to ping')]
    private string $url;

    public function __invoke(
        PingerInterface $client,
    ): int {
        try {
            $this->writeln(\sprintf('Sending ping request [%s]...', $this->url));

            $response = $client->ping(
                new Context([]),
                new PingRequest(['url' => $this->url]),
            );

            $this->writeln(\sprintf(
                'Response: code - %d',
                $response->getStatusCode(),
            ));

        } catch (GrpcClientException $e) {
            $this->writeln(\sprintf(
                'Error: code - %d, message - %s',
                $e->getCode(),
                $e->getMessage(),
            ));
        }

        return self::SUCCESS;
    }
}
```

To use this command, run it from the command line:

```terminal
php app.php ping https://google.com
```

This will call the Pinger service and print the HTTP status code of the response.

## Advanced Client Features

The `spiral/grpc-client` package provides several advanced features to enhance your gRPC client implementation.

### Interceptors

Interceptors allow you to modify or extend the behavior of gRPC requests. The package includes several built-in interceptors:

- `SetTimoutInterceptor` - sets a timeout for the gRPC request
- `RetryInterceptor` - automatically retries failed requests
- `ExecuteServiceInterceptors` - executes service-specific interceptors

You can also create your own interceptors by implementing the `InterceptorInterface`:

```php
use Spiral\Core\InterceptorInterface;
use Spiral\Core\CoreInterceptorInterface;

final class AuthContextInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly AuthContextInterface $authContext,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $token = $this->authContext->getToken();

        if ($token === null) {
            return $handler->handle($context);
        }

        $metadata = \Spiral\Grpc\Client\Interceptor\Helper::withMetadata($context);
        $metadata['auth-token'] = [$token];

        return $handler->handle(\Spiral\Grpc\Client\Interceptor\Helper::withMetadata($context, $metadata));
    }
}
```

### Multiple Connections

You can configure multiple connections to a service for load balancing or failover:

```php
new ServiceConfig(
    connections: [
        ConnectionConfig::createInsecure('service-1:9001'),
        ConnectionConfig::createInsecure('service-2:9001'),
    ],
    interfaces: [
        \GRPC\Pinger\PingerInterface::class,
    ],
)
```

### Secure Connections

To create a secure connection, use the `TlsConfig` class:

```php
new ConnectionConfig(
    address: 'secure-service:9001',
    tls: new TlsConfig(
        privateKey: '/my-project.key',
        certChain: '/my-project.pem',
    ),
),
```

## Golang Clients

GRPC allows you to create a client SDK in any supported language. To generate a client in Golang, install the GRPC toolkit first:

### 1. Install the necessary dependencies

To use gRPC in Go, you will need to install the necessary dependencies. You can do this using the Go package manager:

```bash
go get -u google.golang.org/grpc
go get -u github.com/golang/protobuf/protoc-gen-go
```

> **See more**
> Read more about how to create Golang GRPC clients and server [here](https://grpc.io/docs/tutorials/basic/go/).

### 2. Compile the `.proto` file

Next, you will need to compile the `.proto` file into Go code. You can do this using the protoc compiler and the Go
plugin:

```bash
protoc -I proto/ proto/pinger.proto --go_out=plugins=grpc:pinger
```

This will generate a` pinger.pb.go` file, which contains the Go classes for the service and messages defined in the
`.proto` file.

> **Note**
> Notice the `package` name in `pinger.proto`.

### 3. Create the client

Now you can create a client in Go for the Pinger service. Here is an example of a client that calls the `ping()` method 
of the Pinger service and prints the HTTP status code to the console:

```golang
package main

import (
	"context"
	"fmt"
	"log"

	"google.golang.org/grpc"
	"pinger"
)

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial("127.0.0.1:9001", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	
	defer conn.Close()

	client := pinger.NewPingerClient(conn)

	// Call the ping method.
	response, err := client.Ping(context.Background(), &pinger.PingRequest{
		Url: "https://google.com",
	})

	if err != nil {
		log.Fatalf("error calling ping: %v", err)
	}

	// Print the HTTP status code.
	fmt.Println(response.StatusCode)
}
```

You can run this client using the Go command:

```bash
go run main.go
```

### Passing Metadata

To pass metadata between server and client:

```golang
package main

import (
	"context"
	"fmt"
	"log"

	"google.golang.org/grpc"
	"google.golang.org/grpc/metadata"
	"pinger"
)

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial("127.0.0.1:9001", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	
	defer conn.Close()

	client := pinger.NewPingerClient(conn)
	
	// attach value to the server
	ctx := metadata.AppendToOutgoingContext(context.Background(), "client-key", "client-value")

	var header metadata.MD
	
	// Call the ping method.
	response, err := client.Ping(ctx, &pinger.PingRequest{
		Url: "https://google.com",
	}, grpc.Header(&header))

	if err != nil {
		log.Fatalf("error calling ping: %v", err)
	}

	// Print the HTTP status code.
	fmt.Println(response.StatusCode)
}
```

> **See more**
> Read more about working with metadata in
> Golang [here](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md).
