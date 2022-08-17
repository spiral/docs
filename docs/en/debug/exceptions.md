# Debug - Handling Exceptions

Spiral Framework provides multiple ways to handle critical and application-level exceptions.

## Using Middleware

You can create HTTP middleware to intercept any specific exception type thrown inside your controllers. We can use
Whoops to demonstrate how to write it.

```bash
composer require filp/whoops
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
    private ResponseFactoryInterface $responseFactory;

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

> **Note**
> Do not forget to disable default `Spiral\Bootloader\Http\ErrorHandlerBootloader`.

You can use this approach to suppress the specific type of exceptions in your application.

## Using Interceptors

You can also handle domain-specific exceptions in `Spiral\Core\CoreInterface`. We can use Whoops to demonstrate how to
write it.

```bash
composer require filp/whoops
```

And our interceptor:

```php
namespace App\Interceptor;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use \Spiral\Core\CoreInterface;
use \Spiral\Core\CoreInterceptorInterface;

class WhoopsInterceptor implements CoreInterceptorInterface
{
    private ResponseFactoryInterface $responseFactory;

    public function __construct(ResponseFactoryInterface $responseFactory)
    {
        $this->responseFactory = $responseFactory;
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): ResponseInterface
    {
        try {
            return $core->callAction($controller, $action, $parameters);
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

Make sure to enable this interceptor via Bootloader:

```php
use App\Middleware\WhoopsMiddleware;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\HttpBootloader;
use Spiral\Bootloader\DomainBootloader;

class AppBootloader extends DomainBootloader
{
    protected const INTERCEPTORS = [
        //...
        WhoopsInterceptor::class,
    ];
}
```

## Snapshots

In addition to the custom middleware framework provides a unified way to handle exceptions registration (including fatal
exceptions). Such functionality is delivered by `spiral/snapshots` package and intended for exception registration in
external monitoring solutions (for example, Sentry):

You can implement your snapshot provider via `Spiral\Snapshots\SnapshotterInterface`:

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

> **Note**
> Make sure to bind your implementation to `Spiral\Snapshots\SnapshotterInterface` to enable it.
