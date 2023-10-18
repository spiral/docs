# The Basics — Errors handling

During the development process, it is common for errors and exceptions to arise. Debugging these exceptions can be a
challenging and time-consuming task, but it is a critical aspect of the development process. Spiral offers
a range of tools and techniques for debugging exceptions and identifying the underlying cause of issues.

This documentation will guide you through the features available in Spiral for exception handling, rendering, and
customizations.

## The Exception Handler

Spiral offers a robust mechanism for handling exceptions using the `Spiral\Exceptions\ExceptionHandler`.

## Exception rendering

Spiral uses formats to determine which renderer should be used to handle a given exception. The format can
be based on the environment, such as `cli` for console applications or `http` for HTTP requests. This allows for
different renderers to be registered and used depending on the context in which the exception was encountered.

### How it works

Sometimes, you might want to show errors in a special way, like in JSON for an API. Here's how you can do that:

1. **Make a Renderer**

Here's an example of how to implement a JSON renderer:

```php app/src/Application/Exception/Renderer/JsonRenderer.php
<?php

namespace Spiral\YiiErrorHandler;

use Spiral\Exceptions\ExceptionRendererInterface;
use Spiral\Exceptions\Verbosity;
use Yiisoft\ErrorHandler\Renderer\JsonRenderer as YiiJsonRenderer;
use Yiisoft\ErrorHandler\ThrowableRendererInterface;

final class JsonRenderer implements ExceptionRendererInterface
{
    public const FORMATS = ['application/json', 'json'];

    public function __construct(
        private readonly ?ThrowableRendererInterface $renderer = new YiiJsonRenderer()
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
        return \in_array($format, self::FORMATS, true);
    }
}
```

> **Note**
> `Spiral\YiiErrorHandler\JsonRenderer` is a part of `spiral-packages/yii-error-handler-bridge` package.

2. **Register Your Renderer**

Register the custom renderer using a bootloader

```php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;
use Spiral\YiiErrorHandler\JsonRenderer;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler): void
    {
        $handler->addRenderer(new JsonRenderer());
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

3. **Use Your Renderer**

To use this renderer for handling exceptions in a web application, we can create a new middleware that will catch all
exceptions and render them using this renderer only when a specific header such as `Accept=application/json` is present
in the request. This allows for a more granular control over how exceptions are handled and displayed to the client,
depending on their desired format.

```php
namespace App\Endpoint\Web\Middleware;

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
        
        $response = $this->responseFactory->createResponse($code);
        $response->getBody()->write(
            (string) $this->renderer->render(
                exception: $e,
                format: $request->getHeaderLine('Accept') ?? 'application/json'
            )
        );

        return $response;
    }
}
```

As you can see, different renderers can be used for different environments, such as a console renderer for command-line
applications, or a JSON renderer for API responses. Additionally, different renderers can be used for different formats.

### Existing Renderers

Here are some renderers Spiral already gives you:

| Renderer                                     | Formats                                         |
|----------------------------------------------|-------------------------------------------------|
| `Spiral\Exceptions\Renderer\ConsoleRenderer` | `console`, `cli`                                |
| `Spiral\Exceptions\Renderer\JsonRenderer`    | `application/json`, `json`                      | 
| `Spiral\Exceptions\Renderer\PlainRenderer`   | `text/plain`, `text`, `plain`, `cli`, `console` |

In some cases, for example when `DEBUG=true` you may prefer to render a beautiful error page, with code highlighting,
such as [filp/whoops](https://github.com/filp/whoops)
or [yiisoft/error-handler](https://github.com/spiral-packages/yii-error-handler-bridge).

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

<hr />

## Exception reporting

In Spiral, you can use reporters to keep track of problems, like errors, in your application. Reporters can do two main
things:

- They can save information about these problems in a file. This way, you can look at the file later to figure out what
  went wrong.
- They can also send reports about these problems to other services, like [Sentry](https://sentry.io). These services
  can provide even more details about the issues in your application.

Imagine you have a website, and sometimes things don't work as they should. This can happen because of errors in your
code, like when a file is missing or there's a problem with the database. Reporters help you handle these errors
effectively.

### How Reporters Work

#### 1. Implementing `ExceptionReporterInterface` or using a built-in reporters:

You'll create a class (like `CustomReporter` in the example) that implements
the `Spiral\Exceptions\ExceptionReporterInterface`. Think of this class as a reporter agent that knows what to do when
an exception occurs.

> **Note**
> Read more about available reporters in the [Available Reporters](#available-reporters) section below.

```php app/src/Application/Exception/Reporter/CustomReporter.php
namespace App\Application\Exception\Reporter;

final class CustomReporter implements ExceptionReporterInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {}

    public function report(\Throwable $exception): void
    {
        // Store exception information in a file or send it to an external service
        $this->logger->error($exception->getMessage(), ['exception' => $exception]);
    }
}
```

#### 2. Registration of Reporters:

To use reporters, you first need to register them with the `ExceptionHandler` class in a similar way to renderers, by
using the `addReporter` method and providing an instance of a class that implements
the `Spiral\Exceptions\ExceptionReporterInterface`.

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;
use App\Application\Exception\Reporter\CustomReporter;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler): void
    {
        $handler->addReporter(new CustomReporter());
    }
}
```

#### 3. Using Reporters in Your Code:

Now, in your application code, you can make use of these reporters whenever you expect an exception might occur.

For instance, in the `PingSiteJob` example, if something goes wrong while trying to ping a website (like the website
being down), an exception is caught and reported using the reporter.

```php
<?php

declare(strict_types=1);

namespace App\Job;

use Spiral\Exceptions\ExceptionReporterInterface;

final class PingSiteJob
{
    public function __construct(
        private PingClient $client,
        private ExceptionReporterInterface $reporter,
    ) {
    }

    public function handle(string $url): void
    {
        try {
            $this->client->ping($url);
        } catch (\Throwble $e) {
            $this->reporter->report($e);
        }
    }
}
```

> **Note**
> Reporter will send exception through all registered reporters.

### Available Reporters

Spiral comes with two built-in reporters that are available out of the box,
the `Spiral\Exceptions\Reporter\LoggerReporter`, the `Spiral\Exceptions\Reporter\FileReporter`
and `Spiral\Exceptions\Reporter\StorageReporter`.

#### Logger Reporter

The `Spiral\Exceptions\Reporter\LoggerReporter` is enabled by default and allows you to log exceptions using a logger
registered in the application. This can be useful for tracking and analyzing errors over time.

#### File Reporter

The `Spiral\Exceptions\Reporter\FileReporter` is also enabled by default, it allows you to save detailed information
about an exception to a file known as `snapshot` in `runtime/snapshots` directory.

#### Cloud Storage Reporter

Have you ever faced challenges in storing your app's exception snapshots when working with stateless applications? We've
got some good news. We've made it super easy for you.

By integrating with the `spiral/storage` component, we're giving your stateless apps the power to save exception
snapshots straight into cloud storages, like **S3**.

**Why is this awesome for you?**

1. **Simplified Storage:** No more juggling with complex storage solutions. Save snapshots directly to S3 with ease.
2. **Tailored for Stateless Apps:** Designed specifically for stateless applications, making your deployments smoother
   and hassle-free.
3. **Reliability:** With S3's proven track record, know your snapshots are stored safely and can be accessed whenever
   you need.

The `Spiral\Exceptions\Reporter\StorageReporter` is also enabled by default, it allows you to save detailed information
about an exception to a file known as `snapshot` in `runtime/snapshots` directory.

**To use this reporter, you need:**

1. Set up the `spiral/storage` component. Read more about it in
   the [Component — Storage and Cloud distribution](../advanced/storage.md) section.
2. Register the `Spiral\Bootloader\StorageSnapshotsBootloader`
3. Specify the desired bucket using the `SNAPSHOTS_BUCKET` environment variable where you want to store your snapshots.
4. Register `Spiral\Exceptions\Reporter\StorageReporter` in the `Spiral\Exceptions\ExceptionHandler` class.

## Sentry

Spiral offers a Sentry bridge package that facilitates effortless integration with the [Sentry](https://sentry.io/)
service. This document will guide you through the process of integrating and customizing this tool within your Spiral
application.

### Installation

1. Install the Sentry bridge component:

```terminal
composer require spiral/sentry-bridge
```

2. Once installed, register `Spiral\Sentry\Bootloader\SentryReporterBootloader` bootloader from the package into your

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

It will register `Spiral\Sentry\SentryReporter` in the `Spiral\Exceptions\ExceptionHandler`.

3. Configure the Sentry DSN in the `.env` file:

```dotenv .env
SENTRY_DSN=...
```

### Providing Additional Data

To expose current application logs, such as application logs or PSR-7 request state, enable the debug information
collectors. These collectors gather relevant data about the current application's state.

When an exception occurs, Sentry reporter will request `Spiral\Debug\StateInterface` class from the IoC container and
during creation the object will be filled with information from the registered collectors.

> **Warning**
> Be careful when requesting `Spiral\Debug\StateInterface` from the container. The object will be created on every
> request from the container and you cannot populate it outside collectors. If you need to add additional information
> to the `Spiral\Debug\StateInterface` object, you can use collectors.

#### Http collector

Use the HTTP collector to send data about the current HTTP request to Sentry.

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

#### Logs collector

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

#### Creating Custom Collectors

For specialized data collection, you can create custom collectors. Collector should
implement `Spiral\Debug\StateCollectorInterface` interface.

For example, consider an SQL Collector:

```php app/src/Application/Debug/Collector/SqlCollector.php
namespace App\Application\Debug\Collector;

use Spiral\Logger\Event\LogEvent;
use Spiral\Debug\StateCollectorInterface;

final class SqlCollector implements StateCollectorInterface
{
    public function __construct(
        private readonly Database $db
    ) {
    }

    public function collect(\Spiral\Debug\StateInterface $state): void
    {
       foreach($this->db->getQueries() as $query) {
            $state->addLogEvent(new LogEvent(
                time: $query->getTime(),
                channel: 'sql',
                level: 'info',
                message: $query->getQuery(),
                context: $query->getParameters()
            ));
       }
    }
}
```

> **Warning**
> The above example uses a non-existent Database class, which means you'll need to implement this yourself.

Here are some useful methods of the `Spiral\Debug\StateInterface` object:

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

### Custom collector registration

You can register your collector in a bootloader.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\DebugBootloader;
use App\Application\Exception\Reporter\CustomReporter;

final class AppBootloader extends Bootloader
{
    public function init(DebugBootloader $debug, SqlCollector $sqlCollector): void
    {
        $debug->addStateCollector($sqlCollector);
    }
}
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