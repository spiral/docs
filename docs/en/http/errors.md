# HTTP - Error Pages

Your application will expose some errors and exceptions, some of which must be delivered to the client, and some of them
don't.

The HTTP component includes the default error handling middleware, which used to intercept and log critical errors and
user-level exceptions.

To enable such middleware add the `Spiral\Bootloader\Http\ErrorHandlerBootloader` to your application:

```php
namespace App;

use Spiral\Bootloader as Framework;
use Spiral\DotEnv\Bootloader as DotEnv;
use Spiral\Framework\Kernel;
use Spiral\Prototype\Bootloader as Prototype;
use Spiral\Scaffolder\Bootloader\ScaffolderBootloader;

class App extends Kernel
{
    /*
     * List of components and extensions to be automatically registered
     * within system container on application start.
     */
    protected const LOAD = [
        // ...

        Framework\Http\HttpBootloader::class,
        Framework\Http\RouterBootloader::class,
        Framework\Http\ErrorHandlerBootloader::class,

        // ...
    ];
}
```

## Application Exceptions

The middleware will handle application exceptions and will render them in a developer-friendly mode. To suppress the
delivery of the exception details to the browser, set the env variable `DEBUG` to `false`. In this case, the default 500
error page will be displayed.

> **Note**
> Do not deploy your application to production with enabled debug mode.

## Client Exceptions

There are several exceptions you can throw from your controllers and middleware to cause HTTP level error page, for
example we can trigger `404 Not Found` using `NotFoundException`:

```php
namespace App\Controller;

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

By default, the middleware will use a simple error page without any styles attached. To implement your error page
renderer implement and bind in container the `Spiral\Http\ErrorHandler\RendererInterface` interface:

```php
declare(strict_types=1);

namespace App\ErrorHandler;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface as Request;
use Spiral\Http\ErrorHandler\RendererInterface;
use Spiral\Http\Header\AcceptHeader;
use Spiral\Views\Exception\ViewException;
use Spiral\Views\ViewsInterface;

class ViewRenderer implements RendererInterface
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

Bind it via bootloader:

```php
namespace App\Bootloader;

use App\ErrorHandler\ViewRenderer;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\ErrorHandlerBootloader;
use Spiral\Http\ErrorHandler\RendererInterface;

class ViewRendererBootloader extends Bootloader
{
    public const DEPENDENCIES = [
        ErrorHandlerBootloader::class
    ];

    public const SINGLETONS = [
        RendererInterface::class => ViewRenderer::class
    ];
}
```

> `App\ErrorHandler\ViewRenderer` class included and registered in `spiral/app` by default.

## Logging

The default application includes Monolog handler, which, by default, subscribed to the messages sent by
the `ErrorHandlerMiddleware`. The http error log is located in `app/runtime/logs/http.log` and configured
in `App\Bootloaders\LoggingBootloader`:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Http\Middleware\ErrorHandlerMiddleware;
use Spiral\Monolog\Bootloader\MonologBootloader;

class LoggingBootloader extends Bootloader
{
    /**
     * @param MonologBootloader $monolog
     */
    public function boot(MonologBootloader $monolog)
    {
        $monolog->addHandler(
            ErrorHandlerMiddleware::class,
            $monolog->logRotate(directory('runtime') . 'logs/http.log')
        );
    }
}
```
