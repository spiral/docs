# HTTP — Error Pages

Your application will expose some errors and exceptions, some of which must be delivered to the client and some of them
must not.

## Configuration

### Bootloader

Tha spiral application `spiral/app` bundle includes a default exception handler
bootloader `App\Application\Bootloader\ExceptionHandlerBootloader` which is used to bind some default implementations to
the container:

- `Spiral\Http\Middleware\ErrorHandlerMiddleware\SuppressErrorsInterface` - using to determine if the error should be
  suppressed or not
- `Spiral\Http\ErrorHandler\RendererInterface` - using to render HTTP exceptions.

Add the bootloader to your application:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\ExceptionHandlerBootloader::class,
    // ...
];
```

> **Note**
> If you don't use a `spiral/app` bundle, you can use the `Spiral\Bootloader\Http\ErrorHandlerBootloader` instead.

### Middleware

After the bootloader is added you need to enable the error handler middleware.

The HTTP component includes a default error handling middleware `Spiral\Http\Middleware\ErrorHandlerMiddleware` which is
used to intercept and log critical errors and user-level exceptions.

To enable the middleware, add it to global middleware list, for example via `RoutesBootloader`:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Http\Middleware\ErrorHandlerMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            ErrorHandlerMiddleware::class,
           // ...
        ];
    }

    // ...
}
```

> **See more**
> Read more about global middleware in the [HTTP — Middleware](middleware.md#global-middleware) section.

The middleware will handle application exceptions and will render them in a developer-friendly mode.

To suppress the delivery of the exception details to the browser:

```dotenv .env
DEBUG=false
```

In this case, the default error page will be displayed.

> **Warning**
> Do not deploy your application to production with an enabled debug mode.

Another way to suppress the delivery of the exception details is to implement
the `Spiral\Http\Middleware\ErrorHandlerMiddleware\SuppressErrorsInterface` interface:

```php
use Spiral\Http\Middleware\ErrorHandlerMiddleware\SuppressErrorsInterface;

class SuppressErrors implements SuppressErrorsInterface
{
    public function suppressed(): bool
    {
        return true;
    }
}
```

And bind it to the container:

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
use Spiral\Http\Middleware\ErrorHandlerMiddleware\SuppressErrorsInterface;

class ExceptionHandlerBootloader extends Bootloader
{
    protected const SINGLETONS = [
        SuppressErrorsInterface::class => SuppressErrors::class,
        // ...
    ];
    
    // ...
}
```

## Client Exceptions

There are several exceptions you can throw from your controllers and middleware to cause HTTP level error page. For
example, we can trigger `404 Not Found` using `NotFoundException`:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Core\Container\SingletonInterface;
use Spiral\Http\Exception\ClientException\NotFoundException;

class HomeController implements SingletonInterface
{
    public function index()
    {
        throw new NotFoundException();
    }
}
```

Other exceptions include:

| Code | Exception                                                   |
|------|-------------------------------------------------------------|
| 400  | Spiral\Http\Exception\ClientException\BadRequestException   |
| 401  | Spiral\Http\Exception\ClientException\UnauthorizedException |
| 403  | Spiral\Http\Exception\ClientException\ForbiddenException    |
| 404  | Spiral\Http\Exception\ClientException\NotFoundException     |
| 500  | Spiral\Http\Exception\ClientException\ServerErrorException  |

> **Note**
> Do not use http exceptions inside your services and repositories as it will couple your implementation to http
> dispatcher. Use domain-specific exceptions and their mapping to http exception instead.

## Page Renderer

By default, the middleware will use `Spiral\Http\ErrorHandler\PlainRenderer` - a simple error page without any styles
attached. If you want to use a custom error page renderer, you can implement
the `Spiral\Http\ErrorHandler\RendererInterface`:

```php app/src/Application/Exception/Renderer/ViewRenderer.php
namespace App\Application\Exception\Renderer;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface as Request;
use Spiral\Http\ErrorHandler\RendererInterface;
use Spiral\Http\Header\AcceptHeader;
use Spiral\Views\Exception\ViewException;
use Spiral\Views\ViewsInterface;

final class ViewRenderer implements RendererInterface
{
    private const GENERAL_VIEW = 'exception/error';
    private const VIEW = 'exception/%s';

    public function __construct(
        private readonly ViewsInterface $views,
        private readonly ResponseFactoryInterface $responseFactory
    ) {
    }

    public function renderException(Request $request, int $code, \Throwable $exception): ResponseInterface
    {
        $acceptItems = AcceptHeader::fromString($request->getHeaderLine('Accept'))->getAll();
        if ($acceptItems && $acceptItems[0]->getValue() === 'application/json') {
            return $this->renderJson($code, $exception);
        }

        return $this->renderView($code, $exception);
    }

    private function renderJson(int $code, \Throwable $exception): ResponseInterface
    {
        $response = $this->responseFactory->createResponse($code);

        $response = $response->withHeader('Content-Type', 'application/json; charset=UTF-8');
        $response->getBody()->write(\json_encode(['status' => $code, 'error' => $exception->getMessage()]));

        return $response;
    }

    private function renderView(int $code, \Throwable $exception): ResponseInterface
    {
        $response = $this->responseFactory->createResponse($code);

        try {
            // Try to find view for specific code
            $view = $this->views->get(\sprintf(self::VIEW, $code));
        } catch (ViewException) {
            // Otherwise use default error page
            $view = $this->views->get(self::GENERAL_VIEW);
        }

        $content = $view->render(['code' => $code, 'exception' => $exception]);
        $response->getBody()->write($content);

        return $response;
    }
}
```
And bind it to the container:

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
use Spiral\Http\ErrorHandler\RendererInterface;
use App\Application\Exception\Renderer\ViewRenderer;

class ExceptionHandlerBootloader extends Bootloader
{
    protected const SINGLETONS = [
        RendererInterface::class => ViewRenderer::class,
        // ...
    ];
    
    // ...
}
```

> **Note**
> `App\Application\Service\ErrorHandler\ViewRenderer` class is included and registered in `spiral/app` by default.

## Logging

The default application includes Monolog handler, which, by default, is subscribed to the messages sent by
the `ErrorHandlerMiddleware`. The http error log is located in `app/runtime/logs/http.log` and configured
in `App\Application\Bootloader\LoggingBootloader`:

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Http\Middleware\ErrorHandlerMiddleware;
use Spiral\Monolog\Bootloader\MonologBootloader;

class LoggingBootloader extends Bootloader
{
    public function boot(MonologBootloader $monolog)
    {
        $monolog->addHandler(
            ErrorHandlerMiddleware::class,
            $monolog->logRotate(directory('runtime') . 'logs/http.log')
        );
    }
}
```

> **See more**
> Read more about Logging in the [The Basics — Logging](../basics/logging.md) section.

<hr>

## Related topics

- [Getting started — Configuration](../start/configuration.md)
- [The Basics — Errors handling](../basics/errors.md)
- [Getting started — Deployment](../start/deployment.md)