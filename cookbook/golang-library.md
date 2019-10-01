# Cookbook - Integrating Golang Library
You are able to extend the functionality of your application by including PHP or Golang libraries. While for the PHP library
you only need to run `composer require`, the Golang will require you to build your own version of [application server](/framework/application-server.md).

> Check [Awesome Go](https://github.com/avelino/awesome-go) to find some inspiration.

In this tutorial, we will show how to integrate https://github.com/russross/blackfriday library to the application server and
define the RPC interface for your application. 

> Attention, this article excepts that you are familiar with the Golang programming language.

## RoadRunner service
Make sure to require the go module dependency first:

```bash
$ go get github.com/russross/blackfriday
```

We are able to create our service now since our service doesn't need any configuration we can locate all the code in
a single file `markdown/service.go`:

```go
package markdown

import (
	"github.com/russross/blackfriday"
	"github.com/spiral/roadrunner/service/rpc"
)

const ID = "markdown"

type Service struct{}

func (s *Service) Init(rpc *rpc.Service) (bool, error) {
	if err := rpc.Register(ID, &rpcService{}); err != nil {
		return false, err
	}

	return true, nil
}

type rpcService struct{}

func (s *rpcService) Convert(input []byte, output *[]byte) error {
	*output = blackfriday.Run(input)
	return nil
}
```

Make sure to register this service in your `main.go` file:

```go
rr.Container.Register(markdown.ID, &markdown.Service{})
```

> Read more about RoadRunner services [here](https://roadrunner.dev/docs/beep-beep-service).

## PHP SDK
You can invoke newly created service immediately via `Spiral\Goridge\RPC`: 

```php
use Spiral\Goridge\RelayInterface;
use Spiral\Goridge\RPC;

// ...

public function index(RPC $rpc)
{
    return $rpc->call(
        'markdown.Convert',
        file_get_contents('README.md'),
        RelayInterface::PAYLOAD_RAW // for []byte payloads
    );
}
```

It is recommended to wrap direct RPC calls with proper service code:

```php
use Spiral\Core\Container\SingletonInterface;
use Spiral\Goridge\Exceptions\ServiceException;
use Spiral\Goridge\RelayInterface;
use Spiral\Goridge\RPC;

class Blackfriday implements SingletonInterface
{
    private $rpc;

    public function __construct(RPC $rpc)
    {
        $this->rpc = $rpc;
    }

    public function convert(string $markdown): string
    {
        try {
            return $this->rpc->call('markdown.Convert', $markdown, RelayInterface::PAYLOAD_RAW);
        } catch (ServiceException $e) {
            throw new \RuntimeException("Unable to convert markdown", $e->getCode(), $e);
        }
    }
}
```

And use this service in your code:

```php
public function index(Blackfriday $bf)
{
    return $bf->convert(file_get_contents('README.md'));
}
```
