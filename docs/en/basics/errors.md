# The Basics — Errors handling

During the development process, it is common for errors and exceptions to arise. Debugging these exceptions can be a
challenging and time-consuming task, but it is a critical aspect of the development process. Spiral offers
a range of tools and techniques for debugging exceptions and identifying the underlying cause of issues.

## Exception rendering

By default, the framework uses the `Spiral\Exceptions\Renderer\PlainRenderer` for rendering exceptions. In some cases,
for example when `DEBUG=true` you may prefer to render a beautiful error page, with code highlighting, such
as [filp/whoops](https://github.com/filp/whoops)
or [yiisoft/error-handler](https://github.com/spiral-packages/yii-error-handler-bridge).

### Available Renderers

The `Spiral\Exceptions\Renderer\ConsoleRenderer`, `Spiral\Exceptions\Renderer\JsonRenderer`,
and `Spiral\Exceptions\Renderer\PlainRenderer` are available out of the box.

| Renderer        | Formats                               |
|-----------------|---------------------------------------|
| ConsoleRenderer | console, cli                          |
| JsonRenderer    | application/json, json                |
| PlainRenderer   | text/plain, text, plain, cli, console |

### Custom Renderer

For example, to integrate with the `yiisoft/error-handler` package, a developer would create a custom `HtmlRenderer`
class that implements the `Spiral\Exceptions\ExceptionRendererInterface` to render the exception.

Here is an example of integration with the package:

```php app/src/Application/ErrorHandler/HtmlRenderer.php
namespace App\Application\ErrorHandler;

use Spiral\Exceptions\ExceptionRendererInterface;
use Spiral\Exceptions\Verbosity;
use Yiisoft\ErrorHandler\Renderer\HtmlRenderer as YiiHtmlRenderer;
use Yiisoft\ErrorHandler\ThrowableRendererInterface;

final class HtmlRenderer implements ExceptionRendererInterface
{
    public function __construct(
        private readonly YiiHtmlRenderer $renderer
    ) {
    }

    public function render(
        \Throwable $exception,
        ?Verbosity $verbosity = Verbosity::BASIC,
        string $format = null,
    ): string {
        if ($verbosity >= Verbosity::VERBOSE) {
            return (string)$this->renderer->renderVerbose($exception);
        }

        return (string)$this->renderer->render($exception);
    }

    public function canRender(string $format): bool
    {
        return \in_array($format, ['html', 'application/html', 'text/html'], true);
    }
}
```

This custom renderer can then be registered with the `Spiral\Exceptions\ExceptionHandler` class via a bootloader.

> **Note**
> When you register a new renderer, it will be add at the top of renderers list.

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;
use App\Application\ErrorHandler\HtmlRenderer;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler): void
    {
        $handler->addRenderer(new HtmlRenderer());
    }
}
```

> **Warning**
> Don't forget to add this bootloader to the top of bootloaders list in `app/src/Application/Kernel.php`:

```php app/src/Application/Kernel.php
protected const LOAD = [
    \App\Application\Bootloader\ExceptionHandlerBootloader::class,
    // ...
];
```

Spiral uses formats to determine which renderer should be used to handle a given exception. The format can
be based on the environment, such as `cli` for console applications or `http` for HTTP requests. This allows for
different renderers to be registered and used depending on the context in which the exception was encountered.

For example, in a console application, it may be more appropriate to use a `Spiral\Exceptions\Renderer\ConsoleRenderer`
that is specifically designed to handle exceptions in a command-line environment. This renderer could include features
such as colorizing the stack trace, to make it more readable and easier to understand, while in a web application, a
more detailed and visually appealing renderer could be used.

In some cases, it may be useful to render exceptions as a `JSON` string, especially when working with APIs or other
services that expect responses in this format.

```php app/src/Application/ErrorHandler/JsonRenderer.php
namespace App\Application\ErrorHandler;
use Spiral\Exceptions\ExceptionRendererInterface;

final class JsonRenderer implements ExceptionRendererInterface
{
    public function render(
        \Throwable $exception,
        ?Verbosity $verbosity = Verbosity::BASIC,
        string $format = null,
    ): string {

        return \json_encode([
            'error'      => \sprintf(
                '[%s] %s as %s:%s',
                $exception::class,
                $exception->getMessage(),
                $exception->getFile(),
                $exception->getLine()
            ),
            'stacktrace' => \iterator_to_array($this->renderTrace($exception->getTrace())),
        ]);
    }

    private function renderTrace(array $trace): array
    {
        // ...
    }
    
    public function canRender(string $format): bool
    {
        return \in_array($format, ['application/json', 'json'], true);
    }
}
```

And register it:

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
public function init(ExceptionHandler $handler): void
{
    $handler->addRenderer(new JsonRenderer());
}
```

To use this renderer for handling exceptions in a web application, we can create a new middleware that will catch all
exceptions and render them using this renderer only when a specific header such as `Accept=application/json` is present
in the request. This allows for a more granular control over how exceptions are handled and displayed to the client,
depending on their desired format.

```php app/src/Application/Http/Middleware/ErrorHandlerMiddleware.php
namespace App\Application\Http\Middleware;

use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\RequestHandlerInterface as Handler;
use Psr\Http\Message\ResponseFactoryInterface;
use Spiral\Exceptions\ExceptionRendererInterface;
use Spiral\Http\Exception\ClientException;
use Spiral\Router\Exception\RouterException;

class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ExceptionRendererInterface $renderer,
        private readonly ResponseFactoryInterface $responseFactory,
    ) {
    }

    public function process(Request $request, Handler $handler): Response
    {
        try {
            return $handler->handle($request);
        } catch (ClientException|RouterException $e) {
            $code = $e instanceof ClientException ? $e->getCode() : 404;
        } catch (\Throwable $e) {
            $code = 500;
        }


        if ($request->getHeaderLine('Accept') !== 'application/json') {
            throw $e;
        }

        
        $response = $this->responseFactory->createResponse($code);
        $response->getBody()->write(
            (string) $this->renderer->render(
                exception: $e,
                format: 'application/json'
            )
        );

        return $response;
    }
}
```

As you can see, different renderers can be used for different environments, such as a console renderer for command-line
applications, or a JSON renderer for API responses. Additionally, different renderers can be used for different formats.

### Verbosity Levels

The verbosity level can be used to control the amount of information that is displayed when an exception is rendered.
You can set `VERBOSITY_LEVEL` in `.env` file:

```dotenv .env
# Verbosity level
VERBOSITY_LEVEL=verbose # basic, verbose, debug
```

The possible values are defined by the `Spiral\Exceptions\Verbosity` enum:

#### basic or 0

Indicates that only basic information about the exception should be shown. If an error occurs, you will see:

```output
[Spiral\Router\Exception\RouteNotFoundException] 
Unable to route `http://127.0.0.1`. in vendor/spiral/framework/src/Router/src/Router.php:75
```

#### verbose or 1

Indicates that more detailed information about the exception should be shown. If an error occurs, you will see:

```output

[Spiral\Router\Exception\RouteNotFoundException]
Unable to route `http://127.0.0.1`. in vendor/spiral/framework/src/Router/src/Router.php:75

 1. Spiral\Router\Router->Spiral\Router\{closure}() at vendor/spiral/framework/src/Router/src/Router.php:75
 2. Spiral\Router\Router->Spiral\Router\{closure}()
 3. ReflectionFunction->invokeArgs() at vendor/spiral/framework/src/Core/src/Internal/Invoker.php:73
 4. ...
```

#### debug or 2

Indicates that the most detailed information about the exception should be shown. If an error occurs, you will see:

```output
[Spiral\Router\Exception\RouteNotFoundException]
Unable to route `http://127.0.0.1`. in vendor/spiral/framework/src/Router/src/Router.php:75

 1. Spiral\Router\Router->Spiral\Router\{closure}() at vendor/spiral/framework/src/Router/src/Router.php:75
   73                 if ($route === null) {
   74                     $this->eventDispatcher?->dispatch(new RouteNotFound($request));
>  75                     throw new RouteNotFoundException($request->getUri());
   76                 }
   77 

 2. ...
```

### Yii Error Renderer

The Yii Error Handler is a bridge package for Spiral that provides integration with the Yii framework's error
handlers.

![screenshot](https://user-images.githubusercontent.com/773481/215085868-a7228f6c-1be0-460d-b910-85fa2cd1195b.png)

#### Installation

To install the component:

```terminal
composer require spiral-packages/yii-error-handler-bridge
```

After package install you need to register bootloader from the package:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\YiiErrorHandler\Bootloader\YiiErrorHandlerBootloader::class,
    // ...
];
```

The `YiiErrorHandlerBootloader` will register all available renderers during initialization. If you wish to register
specific renderers.

#### Built-in renderers

The bridge provides several built-in renderers for displaying errors:

- `HtmlRenderer`: Renders error pages as HTML.
- `JsonRenderer`: Renders error pages as JSON. This can be useful for handling errors in API requests.
- `PlainTextRenderer`: Renders error pages as plain text.

<hr />

## Exception reporting

In addition to renderers, Spiral also allows for the registration of reporters, which can be used to store
information about exceptions in a file or send it to an external service like [Sentry](https://sentry.io).

Reporters can be registered with the `ExceptionHandler` class in a similar way to renderers, by using the addReporter
method and providing an instance of a class that implements the `Spiral\Exceptions\ExceptionReporterInterface`.

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;
use App\Application\ErrorHandler\HtmlRenderer;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler): void
    {
        $handler->addReporter(new HtmlRenderer());
    }
}
```

### Available Reporters

Spiral comes with two built-in reporters that are available out of the box,
the `Spiral\Exceptions\Reporter\LoggerReporter` and the `Spiral\Exceptions\Reporter\FileReporter`.

#### LoggerReporter

The `LoggerReporter` is enabled by default and allows you to log exceptions using a logger registered in the
application. This can be useful for tracking and analyzing errors over time.

#### FileReporter

The `FileReporter` is also enabled by default, it allows you to save detailed information about an exception to a file
known as `snapshot`.

### Sentry Reporter

The Sentry Reporter is a bridge package for Spiral that provides integration with the Sentry error reporting.

#### Installation

To install the component:

```terminal
composer require spiral/sentry-bridge
```

After installing the package, you need to add the bootloader from the package to your application.

There are two bootloaders available:

##### Sentry as a reporter

SentryReporterBootloader registers `Spiral\Sentry\SentryReporter` in the `ExceptionHandler`. The Reporter sends the
exception to Sentry.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

#### Sending additional data

To expose current application logs, PSR-7 request state, etc., you can enable additional debug extensions.

##### Http collector

Use an HTTP collector to send data about the current HTTP request to Sentry.

It will send the following information about the current request:

- `method`
- `url`
- `headers`
- `query params`
- `request body`

To enable the HTTP collector, you first need to register `Spiral\Bootloader\Debug\HttpCollectorBootloader`
before `SentryReporterBootloader`.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Debug\HttpCollectorBootloader::class,
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

Then you need to register the middleware `Spiral\Debug\StateCollector\HttpCollector` in the application.

> **See more**
> Read more how to register middleware in the [HTTP — Routing](../http/routing.md#add-middleware) section.

##### Logs collector

Use the Logs collector to send all received logs to Sentry.

To enable the Logs collector, you just need to register S`piral\Bootloader\Debug\LogCollectorBootloader`
before `SentryBootaloder`.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Debug\LogCollectorBootloader::class,
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

##### Custom data collector

You can use the `Spiral\Debug\StateInterface` class to send custom data to Sentry.

You can request a State object from the container.

```php
$state = $container->get(\Spiral\Debug\StateInterface::class);
```

**Add a tag**

The method will add tags associated with the current scope

```php
$state->addTag('IP address', $currentRequest->getIpAddress());
$state->addTag('Environment', $env->get('APP_ENV'));
```

**Add a variable**

The method will add extra data associated with the current scope

```php
$state->setVariable('query', $currentRequest->getQueryParams());
```

**Add a log event**

The method will add a log event as a breadcrumb to the current scope.

```php
$state->addLogEvent(new \Spiral\Logger\Event\LogEvent(
    time: new \DateTimeImmutable(),
    channel: 'default',
    level: 'info',
    message: 'Something went wrong',
    context: ['foo' => 'bar']
));
```

## Extending the error handler

By default, the framework uses `Spiral\Exceptions\ExceptionHandler` class exceptions handling. If you want to use your
own implementation, you can do it by extending it and register during application creation.

```php app.php
use App\Application\Kernel;

// ...

// Initialize shared container, bindings, directories and etc.
$app = Kernel::create(
    directories: ['root' => __DIR__],
    exceptionHandler: \App\Application\ErrorHandler\Handler::class,
)->run();

// ...
```