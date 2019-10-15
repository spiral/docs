# Debug - Handling Exceptions
Spiral Framework provides multiple way to handle critical and application level exceptions. 

## Using Middleware
You can create HTTP middleware to intercept any specific exception type thrown inside your controllers. We can use Whoops 
to demonstrate how to write it.

```bash
$ composer require filp/whoops
```

And our middleware:

```php
namespace App\Middleware;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class WhoopsMiddleware implements MiddlewareInterface
{
    private $responseFactory;

    public function __construct(ResponseFactoryInterface $responseFactory)
    {
        $this->responseFactory = $responseFactory;
    }

    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        try {
            return $handler->handle($request);
        } catch (\Throwable $e) {
            $response = $this->responseFactory->createResponse(500);
            $response->getBody()->write($this->renderWhoops($e));

            return $response;
        }
    }

    private function renderWhoops(\Throwable $e): string
    {
        $whoops = new \Whoops\Run();
        $whoops->allowQuit(false);
        $whoops->writeToOutput(false);

        $handler = new \Whoops\Handler\PrettyPageHandler();
        $handler->handleUnconditionally(true); // whoops does not know about RoadRunner

        $whoops->prependHandler($handler);

        return $whoops->handleException($e);
    }
}
```

Make sure to enable this middleware via Bootloader:

```php
use App\Middleware\WhoopsMiddleware;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\HttpBootloader;

class WhoopsBootloader extends Bootloader
{
    public function boot(HttpBootloader $http)
    {
        $http->addMiddleware(WhoopsMiddleware::class);
    }
}
```

> Do not forget to disable default `Spiral\Bootloader\Http\ErrorHandlerBootloader`.

You can use this approach to handle suppress specific type of exceptions in your application.

> You can also handle domain specific exceptions in `Spiral\Core\CoreInterface`.

## Snapshots
In addition to custom middleware framework provides the unified way to handle exceptions registration (including fatal exceptions).
Such functionality is delivered by `spiral/snaphots` package and intended for exception registration in external monitoring
solutions (for example Sentry):

You can implement your own snapshot provider via `Spiral\Snapshots\SnapshotterInterface`:

```php
use Spiral\Snapshots\Snapshot;
use Spiral\Snapshots\SnapshotInterface;
use Spiral\Snapshots\SnapshotterInterface;

class MySnapshotter implements SnapshotterInterface
{
    public function register(\Throwable $e): SnapshotInterface
    {
        // register exception in log, Sentry or etc

        return new Snapshot('unique-id', $e);
    }
}
```

> Make sure to bind your implementation to `Spiral\Snapshots\SnapshotterInterface` to enable it.