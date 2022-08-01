# GRPC - Client SDK

You must generate the client SDK to access your endpoints. This article based on proto file described
[earlier](/grpc/service.md).

## Prerequisites

The PHP client library requires an installation of PHP `grpc` extension.

```bash
sudo pecl install grpc
```

> **Note**
> Read more about the extension [here](https://grpc.io/docs/quickstart/php/).

### Compile grpc-php-plugin

You must compile `grpc-php-plugin` in order to generate the PHP client code.
Follow [this guide](https://www.grpc.io/docs/quickstart/php/#install-protobuf-plugin).

```bash
git clone --recursive -b v1.27.x https://github.com/grpc/grpc
cd grpc
cd third_party/protobuf 
./autogen.sh
./configure CC=clang CXX=clang++
make
make install
cd ../..
make 
make grpc_php_plugin
```

> **Note**
> Make sure to install clang.

The location of generated plugin is `grpc/bin/opt/grpc_php_plugin`.

## Generate SDK

We can generate the PHP client SDK in a new project. Copy the `proto/` directory into the designated folder.

```bash
protoc -I proto/ --plugin=protoc-gen-grpc=grpc/bins/opt/grpc_php_plugin --php_out=. --grpc_out=. proto/calculator.proto 
```

You can observe the generated client code in `App/Calculator/CalculatorClient.php`.

### Composer

Make sure to create the `composer.json` and run `composer update` to make the generated code loadable:

```json
{
  "require": {
    "grpc/grpc": "^1.27",
    "google/protobuf": "^3.11"
  },
  "autoload": {
    "psr-4": {
      "App\\": "App"
    }
  }
}
```

### Certificate

To connect to the running server, make sure to copy `app.crt` to your client application.

## Example

We can create the client now in `client.php`:

```php
<?php

declare(strict_types=1);

require 'vendor/autoload.php';

$client = new \App\Calculator\CalculatorClient(
    'localhost:50051',
    [
        'credentials' => \Grpc\ChannelCredentials::createSsl(file_get_contents("app.crt")),
    ]
);

[$result, $mt] = $client->Sum(
    new \App\Calculator\Sum([
            'a' => 1,
            'b' => 2
    ])
)->wait();

print_r($result->getResult());
```

Run `php client.php` to test your client.

### Passing Metadata

You can pass the metadata using the second argument of the `Sum` function, use `$mt->metadata` to read the response
metadata.

To read and send metadata in `Calculator`:

```php
<?php

declare(strict_types=1);

namespace App\Calculator;

use Spiral\GRPC;

class Calculator implements CalculatorInterface
{
    public function Sum(GRPC\ContextInterface $ctx, Sum $in): Result
    {
        dumprr($ctx->getValue('client-key'));
        $ctx->getValue(GRPC\ResponseHeaders::class)->set('server-key', 'serverValue');

        return new Result([
            'result' => $in->getB() + $in->getA()
        ]);
    }
}
```

Change `client.php` to read metadata from the call:

```php
<?php

declare(strict_types=1);

require 'vendor/autoload.php';

$client = new \App\Calculator\CalculatorClient(
    'localhost:50051',
    [
        'credentials' => \Grpc\ChannelCredentials::createSsl(file_get_contents("app.crt")),
    ]
);

$call = $client->Sum(
    new \App\Calculator\Sum([
            'a' => 1,
            'b' => 2
    ]),
    [
        'client-key' => ['value']
    ]
);

print_r($call->wait()[0]->getResult());
print_r($call->getMetadata()['server-key']);
```

You can now observe the metadata passing from the client and server.

## Golang Clients

GRPC allows you to create client SDK on any supported language. To generate client on Golang, install the GRPC toolkit
first:

```bash
go get -u github.com/golang/protobuf/protoc-gen-go
```

> **Note**
> Read more about how to create Golang GRPC clients and server [here](https://grpc.io/docs/tutorials/basic/go/).

Init the project in the same folder as PHP client to reuse `.proto` and `.crt` files. To generate client stub code:

```bash
go mod init client
go get google.golang.org/grpc
mkdir calculator
protoc -I proto/ proto/calculator.proto --go_out=plugins=grpc:calculator
```

> **Note**
> Note the `package` name in `calculator.proto`.

You can now create `main.go`:

```golang
package main

import (
	app "client/calculator"
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

	client := app.NewCalculatorClient(conn)

	res, err := client.Sum(context.Background(), &app.Sum{A: 1,B: 2})
	if err != nil {
		panic(err)
	}

	log.Printf("Result: %v", res.Result)
}
```

To test your client:

```bash
go run main.go
```

### Passing Metadata

To pass metadata between server and client:

```golang
package main

import (
	app "client/calculator"
	"context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
	"google.golang.org/grpc/metadata"

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

	client := app.NewCalculatorClient(conn)

	// attach value to the server
	ctx := metadata.AppendToOutgoingContext(context.Background(), "client-key", "client-value")

	var header metadata.MD
	res, err := client.Sum(ctx, &app.Sum{A: 1, B: 2}, grpc.Header(&header))
	if err != nil {
		panic(err)
	}

	log.Println(header["server-key"])

	log.Printf("Result: %v", res.Result)
}
```

> **Note**
> Read more about working with metadata in
> Golang [here](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md).
