# Cookbook - Custom PSR-15 handlers
Spiral Framework is compliant with `PSR-7`, `PSR-15` and `PSR-17` community standards, it allows you to swap HTTP layer implementation to any alternative.

> By default, `Psr\Http\Server\RequestHandlerInterface` is implemented and binded to `Spiral\Router\Router`. You would have to disable
the bootloader `Spiral\Bootloader\Http\RouterBootloader` in your application.

## Fast Route
As an example, we can replace the default spiral router with one based on [FastRoute](https://github.com/nikic/FastRoute). The implementation
is provided by [https://github.com/middlewares/fast-route](https://github.com/middlewares/fast-route).

```bash
$ composer require middlewares/fast-route middlewares/request-handler
```

Create bootloader to bind this implementation to our http server. We can either bind handler via `HttpBootloader` or simply
declare `Psr\Http\Server\RequestHandlerInterface` in `SINGLETONS`:

```php
use FastRoute;
use Middlewares;
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Boot\Bootloader\Bootloader;

class FastRouteBootloader extends Bootloader
{
    const SINGLETONS = [
        RequestHandlerInterface::class => [self::class, 'psr15Handler']
    ];

    private function createRoutes(FastRoute\RouteCollector $r)
    {
        $r->addRoute('GET', '/hello/{name}', function ($request) {
            $name = $request->getAttribute('name');

            return sprintf('Hello %s', $name);
        });
    }

    private function psr15Handler(ResponseFactoryInterface $responseFactory)
    {
        $dispatcher = FastRoute\simpleDispatcher(function (FastRoute\RouteCollector $r) {
            $this->createRoutes($r);
        });

        return new Middlewares\Utils\Dispatcher([
            new Middlewares\FastRoute($dispatcher, $responseFactory),
            new Middlewares\RequestHandler(),
        ]);
    }
}
```

> Make sure that `Spiral\Bootloader\Http\HttpBootloader` is enabled.

Add this bootloader to your application, the route `/name/{name}` will be available immediately.

## Custom Handler
You can also implement your own handler, to shorten time we will implement handler directly in the bootloader:

```php
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Boot\Bootloader\Bootloader;

class HttpHandlerBootloader extends Bootloader implements RequestHandlerInterface
{
    const SINGLETONS = [
        RequestHandlerInterface::class => self::class
    ];

    private $responseFactory;

    public function __construct(ResponseFactoryInterface $responseFactory)
    {
        $this->responseFactory = $responseFactory;
    }

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        $response = $this->responseFactory->createResponse(200);
        $response->getBody()->write("hello world");

        return $response;
    }
}
```

> You can find a lot of open-source routers which implement this interface.
