# Cookbook - Integrate Go services with your PHP apps, using RPC
You can extend the functionality of your PHP applications by including Go libraries. While for the PHP library,
you only need to run `composer require`, the Golang will require you to build your version of [application server](/framework/application-server.md).

> Fun fact: the full-text search on this website works via [blevesearch](https://github.com/blevesearch/bleve) integrated to the spiral app.

In this tutorial, we will demonstrate how to integrate the https://github.com/russross/blackfriday Markdown processing library and access it from your PHP application. 

> Attention, this article expects that you are familiar with the Go programming language.

## RoadRunner service
Make sure to require the go module dependency first:

```bash
$ go get github.com/russross/blackfriday
```

Since our service doesn't need any configuration, we can put all the code inside a single file: `markdown/service.go`:

```go
package markdown

import (
	"github.com/russross/blackfriday"
	"github.com/spiral/roadrunner/service/rpc"
)

const ID = "markdown"

// to be registered in the app server
type Service struct{}

func (s *Service) Init(rpc *rpc.Service) (bool, error) {
	if err := rpc.Register(ID, &rpcService{}); err != nil {
		return false, err
	}

	return true, nil
}

// to be called via RPC
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

Build and start your application to activate the service.

## PHP SDK
You can invoke the newly created service right away, using `Spiral\Goridge\RPC`: 

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

It is recommended to wrap direct RPC calls with some proper service code:

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

And use this service in your PHP code:

```php
public function index(Blackfriday $bf)
{
    return $bf->convert(file_get_contents('README.md'));
}
```

## Performance
The selected library in this example shows twice as much performance as the classic https://github.com/erusev/parsedown. Read how to optimize
performance even more by switching to unix sockets for RCP communications in **Performance Tuning** section.

> Obviously, RPC does not come entirely for free. Make sure to properly balance between the socket connection speed
and the complexity of computation.  
