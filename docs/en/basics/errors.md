# The Basics — Errors handling

During the development process, it is common for errors and exceptions to arise. Debugging these exceptions can be a
challenging and time-consuming task, but it is a critical aspect of the development process. Spiral offers
a range of tools and techniques for debugging exceptions and identifying the underlying cause of issues.

This documentation will guide you through the features available in Spiral for exception handling, rendering, and
customizations.

## The Exception Handler

Spiral offers a robust mechanism for handling exceptions provided by the `Spiral\Exceptions\ExceptionHandler` class.

It's designed to manage both global and runtime errors, offering a structured approach to exception handling in your
application.

#### Key features

- **Exception rendering:** Render exceptions in a variety of formats, including HTML, JSON, and plain text.
- **Exception reporting:** Report exceptions to external services, such as [Sentry](#sentry-integration)
  or [S3 storage](#cloud-storage-reporter).
- **Global Error Handling:** The class is used to handle global errors, such as fatal errors and shutdown errors.
- **Customizable:** The class allows for adding custom renderers and reporters.

### Customizing the Exception handler

For applications requiring specific error handling strategies, Spiral offers the flexibility to substitute this
default handler with a custom implementation.

First, create a class that extends the `Spiral\Exceptions\ExceptionHandler` class:

```php app/src/Application/Exception/Handler.php
<?php

declare(strict_types=1);

namespace App\Application\Exception;

use Spiral\Exceptions\ExceptionHandler;
use Throwable;

final class Handler extends ExceptionHandler
{
    // Custom configuration and methods
}
```

Next, specify the class in the `app.php` file:

```php app.php
use App\Application\Kernel;
use App\Application\Exception\Handler;

// ...

$app = Kernel::create(
    directories: ['root' => __DIR__],
    exceptionHandler: Handler::class, // <--
)->run();

// ...
```

#### Customization Points

When a handler is initialized, it will call the `bootBasicHandlers` method, which is one of the ways to customize the
handler. This method is used to register basic renderers and reporters.

```php app/src/Application/Exception/Handler.php
final class Handler extends ExceptionHandler
{
    protected function bootBasicHandlers(): void
    {
        parent::bootBasicHandlers();
        
        // Register your renderers and reporters here
        // $this->addRenderer(new MyRenderer());
        // $this->addReporter(new MyReporter());
    }
}
```

Handler is a great place to handle exceptions that occur during the application's boot process. For example, if you
want to skip reporting some exceptions, you can override the `report` method and handle them there.

```php app/src/Application/Exception/Handler.php
<?php

declare(strict_types=1);

namespace App\Application\Exception;

use Spiral\Exceptions\ExceptionHandler;
use Spiral\Http\Exception\ClientException;

final class Handler extends ExceptionHandler
{
    /**
     * @var class-string<\Throwable>[]
     */
    private array $nonReportableExceptions = [
        ClientException::class,
        // ...
    ];

    public function report(\Throwable $exception): void
    {
        foreach ($this->nonReportableExceptions as $nonReportableException) {
            if ($exception instanceof $nonReportableException) {
                return;
            }
        }

        parent::report($exception);
    }
}
```

> **Note**
> This manual approach to filtering exceptions is still supported but using the [Non-reportable
> Exceptions](#non-reportable-exceptions) feature is more convenient and maintainable.

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

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\ExceptionHandlerBootloader::class,
        // ...
    ];
}
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\ExceptionHandlerBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

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

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\YiiErrorHandler\Bootloader\YiiErrorHandlerBootloader::class,
        // ...
    ];
}
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\YiiErrorHandler\Bootloader\YiiErrorHandlerBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

The `YiiErrorHandlerBootloader` will register all available renderers during initialization. If you wish to register
specific renderers.

#### Built-in renderers

The bridge provides several built-in renderers for displaying errors:

- `HtmlRenderer`: Renders error pages as HTML.
- `JsonRenderer`: Renders error pages as JSON. This can be useful for handling errors in API requests.
- `PlainTextRenderer`: Renders error pages as plain text.

### Verbosity Levels

The verbosity level controls the amount of information displayed when an exception is rendered. Spiral provides three
levels through the `Spiral\Exceptions\Verbosity` enum.

You can configure the verbosity level using the `VERBOSITY_LEVEL` environment variable:

```dotenv .env
# Verbosity level
VERBOSITY_LEVEL=verbose # basic, verbose, or debug
```

The `Verbosity` enum implements `InjectableEnumInterface`, which means it can be automatically injected into your
classes based on the environment configuration:

```php
use Spiral\Exceptions\Verbosity;
use Spiral\Exceptions\ExceptionRendererInterface;

class CustomRenderer implements ExceptionRendererInterface
{
    public function __construct(
        private readonly Verbosity $verbosity,
    ) {}
    
    public function render(\Throwable $exception, ?Verbosity $verbosity = null, ?string $format = null): string
    {
        // Use injected verbosity as default if none provided
        $verbosity ??= $this->verbosity;
        
        // Rendering logic...
    }
}
```

The available verbosity levels are:

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

### Non-reportable Exceptions

Certain exceptions don't require reporting because they're part of normal application flow or represent expected client
errors. Spiral allows you to exclude specific exceptions from being reported to your error tracking services.

By default, these exceptions are non-reportable:

- `Spiral\Http\Exception\ClientException\BadRequestException`
- `Spiral\Http\Exception\ClientException\ForbiddenException`
- `Spiral\Http\Exception\ClientException\NotFoundException`
- `Spiral\Http\Exception\ClientException\UnauthorizedException`
- `Spiral\Filters\Exception\AuthorizationException`
- `Spiral\Filters\Exception\ValidationException`

You can exclude exceptions from reporting in several ways:

#### Using the NonReportable Attribute

Add the `Spiral\Exceptions\Attribute\NonReportable` attribute to any exception class to prevent it from being reported:

```php app/src/Exception/AccessDeniedException.php
namespace App\Exception;

use Spiral\Exceptions\Attribute\NonReportable;

#[NonReportable]
class AccessDeniedException extends \Exception
{
    // Exception implementation
}
```

This approach works automatically—the exception handler checks for this attribute and skips reporting if present.

#### Using the dontReport Method

Register non-reportable exceptions in a bootloader by calling the `dontReport` method:

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
namespace App\Application\Bootloader;

use App\Exception\EntityNotFoundException;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler): void
    {
        $handler->dontReport(EntityNotFoundException::class);
    }
}
```

This method allows you to configure non-reportable exceptions without modifying the exception classes themselves.

#### Extending the ExceptionHandler

For more control, extend the `ExceptionHandler` class and override the `shouldNotReport` method or the
`nonReportableExceptions` property:

```php app/src/Application/Exception/Handler.php
namespace App\Application\Exception;

use App\Exception\BusinessLogicException;
use Spiral\Exceptions\ExceptionHandler;

final class Handler extends ExceptionHandler
{
    protected array $nonReportableExceptions = [
        BusinessLogicException::class,
        // Add more exception classes here
    ];
}
```

Then configure your custom handler in the application kernel:

```php app.php
use App\Application\Kernel;
use App\Application\Exception\Handler;

$app = Kernel::create(
    directories: ['root' => __DIR__],
    exceptionHandler: Handler::class,
)->run();
```

> **Note**
> Exception subclasses are also excluded when their parent class is non-reportable. For example, if `ClientException` is
> non-reportable, all classes extending it will also be excluded from reports.

### How Reporters Work

#### 1. Implementing `ExceptionReporterInterface` or using a built-in reporters:

You'll create a class (like `CustomReporter` in the example) that implements
the `Spiral\Exceptions\ExceptionReporterInterface`. Think of this class as a reporter agent that knows what to do when
an exception occurs.

> **Note**
> Read more about available reporters in the [Available Reporters](#available-reporters) section below.

```php app/src/Application/Exception/Reporter/CustomReporter.php
namespace App\Application\Exception\Reporter;

use Spiral\Exceptions\ExceptionReporterInterface;
use Psr\Log\LoggerInterface;

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

You can register reporters as class instances or as closures for simple reporting logic:

**Using a reporter class:**

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
namespace App\Application\Bootloader;

use App\Application\Exception\Reporter\CustomReporter;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler, CustomReporter $reporter): void
    {
        $handler->addReporter($reporter);
    }
}
```

**Using a closure:**

For simple reporting logic, you can register a closure instead of creating a full reporter class:

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
namespace App\Application\Bootloader;

use Psr\Log\LoggerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler, LoggerInterface $logger): void
    {
        $handler->addReporter(function (\Throwable $exception) use ($logger) {
            $logger->critical('Critical exception occurred', [
                'exception' => $exception::class,
                'message' => $exception->getMessage(),
                'file' => $exception->getFile(),
                'line' => $exception->getLine(),
            ]);
        });
    }
}
```

> **Note**
> Closures are perfect for simple reporting tasks, while implementing `ExceptionReporterInterface` is better for complex
> reporting logic that requires dependency injection or multiple methods.

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
> Reporter will send an exception through all registered reporters.

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

## Sentry Integration

[Sentry](https://sentry.io/) is a powerful error tracking and performance monitoring platform that helps developers
identify, diagnose, and fix issues in production. Spiral provides a bridge package for seamless integration with
Sentry.

> **See more**
> For more information about Sentry features and capabilities, visit
> the [official Sentry documentation](https://docs.sentry.io/).

### Installation

Install the Sentry bridge component:

```terminal
composer require spiral/sentry-bridge
```

After installation, register the bootloader in your application kernel. The bridge provides two bootloaders depending on
your needs:

**For exception reporting only:**

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
        // ...
    ];
}
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

**For snapshot creation (full exception dumps to Sentry):**

If you want exceptions to create detailed snapshots that are sent to Sentry:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Sentry\Bootloader\SentryBootloader::class,
        // ...
    ];
}
```

:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Sentry\Bootloader\SentryBootloader::class,
    // ...
];
```

:::

::::

> **Note**
> `SentryReporterBootloader` registers the reporter with the exception handler.
> `SentryBootloader` additionally registers `SentrySnapshotter` as the snapshot handler, making it the primary
> snapshotter for the application.

### Configuration

Configure the Sentry integration using environment variables or a configuration file.

#### Environment Variables

The minimum required configuration is the DSN (Data Source Name):

```dotenv .env
SENTRY_DSN=https://examplePublicKey@o0.ingest.sentry.io/0
```

**Available Environment Variables:**

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `SENTRY_DSN` | `string` | Required | Sentry Data Source Name from your project settings |
| `SENTRY_ENVIRONMENT` | `string` | `APP_ENV` value | Environment name (e.g., `production`, `staging`) |
| `SENTRY_RELEASE` | `string` | `APP_VERSION` value | Release version identifier |
| `SENTRY_SAMPLE_RATE` | `float` | `1.0` | Error sampling rate (0.0 to 1.0) |
| `SENTRY_TRACES_SAMPLE_RATE` | `float` | `null` | Performance tracing sample rate (0.0 to 1.0) |
| `SENTRY_SEND_DEFAULT_PII` | `bool` | `false` | Whether to send personally identifiable information |

**Example configuration:**

```dotenv .env
SENTRY_DSN=https://examplePublicKey@o0.ingest.sentry.io/0
SENTRY_ENVIRONMENT=production
SENTRY_RELEASE=1.0.0
SENTRY_SAMPLE_RATE=0.5
SENTRY_TRACES_SAMPLE_RATE=0.1
SENTRY_SEND_DEFAULT_PII=false
```

#### Configuration File

For more advanced configuration, create a `config/sentry.php` file:

```php config/sentry.php
use Sentry\Event;
use Sentry\EventHint;

return [
    'dsn' => env('SENTRY_DSN'),
    'environment' => 'production',
    'release' => '1.0.0',
    'sample_rate' => 1.0,
    'traces_sample_rate' => 0.1,
    'send_default_pii' => false,
    
    // Exceptions to ignore (won't be sent to Sentry)
    'ignore_exceptions' => [
        \Spiral\Http\Exception\ClientException\NotFoundException::class,
    ],
    
    // Callback to modify or filter events before sending
    'before_send' => function (Event $event, ?EventHint $hint): ?Event {
        // Filter sensitive data
        // Return null to prevent sending
        return $event;
    },
];
```

**Configuration Options:**

| Option | Type | Description |
|--------|------|-------------|
| `dsn` | `string` | Sentry project DSN |
| `environment` | `string\|null` | Deployment environment identifier |
| `release` | `string\|null` | Application release version |
| `sample_rate` | `float` | Percentage of errors to send (0.0 = 0%, 1.0 = 100%) |
| `traces_sample_rate` | `float\|null` | Percentage of transactions to trace for performance monitoring |
| `send_default_pii` | `bool` | Include personally identifiable information (IP addresses, usernames) |
| `ignore_exceptions` | `array` | Exception class names to exclude from reporting |
| `before_send` | `callable\|null` | Callback to modify events before sending: `function(Event, ?EventHint): ?Event` |

> **Note**
> The `before_send` callback is executed for every event. Return `null` to prevent the event from being sent to Sentry.
> This is useful for filtering sensitive data or implementing custom filtering logic.

### Sentry SDK Integrations

Sentry uses integrations to extend its functionality. The Spiral bridge automatically configures several integrations
and allows you to register custom ones.

#### Built-in Integrations

The bridge automatically configures these Sentry integrations:

- **RequestIntegration** - Captures HTTP request data (URL, method, headers, body)
- **All default Sentry integrations** except:
    - `ErrorListenerIntegration` (disabled - Spiral handles errors)
    - `ExceptionListenerIntegration` (disabled - Spiral handles exceptions)
    - `FatalErrorListenerIntegration` (disabled - Spiral handles fatal errors)

#### Registering Custom Integrations

Register application-specific integrations using the `ClientBootloader`:

```php app/src/Application/Bootloader/SentryBootloader.php
namespace App\Application\Bootloader;

use Sentry\Integration\IntegrationInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Sentry\Bootloader\ClientBootloader;

final class SentryBootloader extends Bootloader
{
    public function init(ClientBootloader $client): void
    {
        // Register custom integration
        $client->addIntegration(new CustomIntegration());
    }
}
```

**Available integrations from Sentry SDK:**

- `FrameContextifierIntegration` - Adds source code context to stack traces
- `EnvironmentIntegration` - Captures environment variables
- `ModulesIntegration` - Lists installed packages/modules
- `TransactionIntegration` - Groups events by transaction

> **See more**
> For a complete list of available integrations, see the
> [Sentry PHP SDK Integrations documentation](https://docs.sentry.io/platforms/php/integrations/).

### HTTP Request Data Collection

The Sentry bridge automatically captures HTTP request information through the `RequestIntegration`. For enhanced user
tracking with IP addresses, use the optional `SetRequestIpMiddleware`.

#### Request IP Middleware

This middleware captures the user's IP address when `send_default_pii` is enabled:

```php app/src/Application/Bootloader/HttpBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Sentry\Http\SetRequestIpMiddleware;

final class HttpBootloader extends Bootloader
{
    public function boot(BaseRoutesBootloader $routes): void
    {
        $routes->addMiddleware(SetRequestIpMiddleware::class);
    }
}
```

The middleware automatically checks the `send_default_pii` configuration and only sets IP addresses when enabled.

> **Note**
> Read more about middleware in the [HTTP — Routing](../http/routing.md#add-middleware) section.

**Captured Request Data:**

When the `RequestIntegration` is active, Sentry captures:

- HTTP method
- Request URL
- Query parameters
- Request headers
- Request body (when applicable)
- User IP address (when `SetRequestIpMiddleware` is registered and `send_default_pii` is `true`)

> **Warning**
> Be cautious when enabling `send_default_pii` as it will include personally identifiable information in error reports.
> Ensure this complies with your privacy policy and data protection regulations (GDPR, CCPA, etc.).

### Container Bindings

The Sentry bridge provides several container bindings for advanced integration and control:

| Binding | Description |
|---------|-------------|
| `Sentry\Options` | Configuration container for Sentry SDK options |
| `Sentry\State\HubInterface` | Central hub for Sentry state and context management |
| `Sentry\ClientInterface` | Direct interface to the Sentry client |
| `Sentry\Integration\RequestFetcherInterface` | Custom request fetcher for PSR-7 integration |

These bindings enable deep integration with the Sentry SDK for advanced use cases.

#### Using the Sentry Hub

The `HubInterface` provides access to Sentry's scope management:

```php app/src/Endpoint/Web/SomeController.php
namespace App\Endpoint\Web;

use Sentry\State\HubInterface;
use Sentry\State\Scope;

final class SomeController
{
    public function __construct(
        private readonly HubInterface $hub,
    ) {}

    public function index(): void
    {
        // Configure the current scope
        $this->hub->configureScope(function (Scope $scope): void {
            $scope->setTag('page', 'checkout');
            $scope->setUser([
                'id' => 123,
                'email' => 'user@example.com',
            ]);
            $scope->setContext('order', [
                'total' => 99.99,
                'items' => 3,
            ]);
        });
        
        // Scope configuration persists for subsequent errors in this request
    }
}
```

#### Using the Sentry Client

Direct access to the Sentry client for advanced operations:

```php
use Sentry\ClientInterface;

final class CustomErrorHandler
{
    public function __construct(
        private readonly ClientInterface $client,
    ) {}
    
    public function captureMessage(string $message): void
    {
        $this->client->captureMessage($message, \Sentry\Severity::warning());
    }
}
```

> **See more**
> For more information about the Sentry SDK API, visit the
> [Sentry PHP SDK documentation](https://docs.sentry.io/platforms/php/).

### Enriching Error Context with State Collectors

State collectors allow you to automatically attach contextual information to every exception sent to Sentry. When an
exception occurs, the Sentry reporter requests `Spiral\Debug\StateInterface` from the container, which is populated by
registered collectors.

> **Warning**
> The `StateInterface` object is created fresh on each request from the container. You cannot populate it directly
> outside of collectors. Always use collectors to add contextual information.

#### Debug State Interface

The `StateInterface` provides methods to enrich error context:

**Tags** - Key-value pairs for filtering and grouping errors:

```php
$state->setTag('environment', 'production');
$state->setTag('server', 'web-01');
$state->setTags([
    'version' => '1.0.0',
    'region' => 'us-east-1',
]);
```

**Variables** - Additional context data:

```php
$state->setVariable('user_id', 12345);
$state->setVariable('session_data', $sessionData);
$state->setVariables([
    'request_id' => $requestId,
    'trace_id' => $traceId,
]);
```

**Log Events** - Breadcrumbs showing events leading to the error:

```php
use Spiral\Logger\Event\LogEvent;

$state->addLogEvent(new LogEvent(
    time: new \DateTimeImmutable(),
    channel: 'database',
    level: 'info',
    message: 'Query executed',
    context: ['query' => $sql, 'duration' => $duration]
));
```

These breadcrumbs appear in Sentry's timeline, helping you understand the sequence of events that led to the error.

#### Built-in State Collectors

Spiral provides several built-in collectors that you can enable by registering their bootloaders.

##### HTTP Collector

Captures HTTP request information including method, URL, headers, query parameters, and request body.

> **Note**
> Since the bridge v2.2, the HTTP collector is optional because the `RequestIntegration` provides better HTTP data
> collection. Consider using `RequestIntegration` instead for most use cases.

**To enable:**

Register `Spiral\Bootloader\Debug\HttpCollectorBootloader` before `SentryReporterBootloader`:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Debug\HttpCollectorBootloader::class,
        \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
        // ...
    ];
}
```

:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Debug\HttpCollectorBootloader::class,
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

:::

::::

Then register the middleware:

```php
$routes->addMiddleware(\Spiral\Debug\StateCollector\HttpCollector::class);
```

> **See more**
> Read more how to register middleware in the [HTTP — Routing](../http/routing.md#add-middleware) section.

##### Logs Collector

Captures all application logs as breadcrumbs in Sentry, providing a complete timeline of events leading to an error.

**To enable:**

Register `Spiral\Bootloader\Debug\LogCollectorBootloader` before `SentryReporterBootloader`:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Debug\LogCollectorBootloader::class,
        \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
        // ...
    ];
}
```

:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Debug\LogCollectorBootloader::class,
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

:::

::::

Once enabled, all logs from your application will automatically appear as breadcrumbs in Sentry error reports.

#### Creating Custom Collectors

Create custom collectors to capture application-specific context. Collectors must implement
the `Spiral\Debug\StateCollectorInterface`.

**Example - Database Query Collector:**

```php app/src/Application/Debug/Collector/DatabaseCollector.php
namespace App\Application\Debug\Collector;

use Spiral\Debug\StateCollectorInterface;
use Spiral\Debug\StateInterface;
use Spiral\Logger\Event\LogEvent;

final class DatabaseCollector implements StateCollectorInterface
{
    private array $queries = [];
    
    public function recordQuery(string $query, array $params, float $duration): void
    {
        $this->queries[] = compact('query', 'params', 'duration');
    }

    public function collect(StateInterface $state): void
    {
        // Add database statistics as tags
        $state->setTag('db.query_count', (string) count($this->queries));
        
        // Add detailed query information as context
        $state->setVariable('database_queries', $this->queries);
        
        // Add each query as a breadcrumb
        foreach ($this->queries as $query) {
            $state->addLogEvent(new LogEvent(
                time: new \DateTimeImmutable(),
                channel: 'database',
                level: 'debug',
                message: 'Query executed',
                context: [
                    'query' => $query['query'],
                    'params' => $query['params'],
                    'duration_ms' => $query['duration'],
                ]
            ));
        }
    }
}
```

**Register the collector:**

```php app/src/Application/Bootloader/DatabaseBootloader.php
namespace App\Application\Bootloader;

use App\Application\Debug\Collector\DatabaseCollector;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\DebugBootloader;

final class DatabaseBootloader extends Bootloader
{
    public function init(
        DebugBootloader $debug,
        DatabaseCollector $collector,
    ): void {
        $debug->addStateCollector($collector);
    }
}
```

**State Collector Best Practices:**

- Keep collectors lightweight - they run on every error
- Use tags for filterable values (status codes, user roles, environments)
- Use variables for detailed context data (session data, configuration)
- Use log events for timeline information (requests, queries, cache operations)
- Avoid collecting sensitive data (passwords, tokens, credit cards)
- Consider memory usage when storing large amounts of data

### Using the Sentry Client Wrapper

The bridge provides a `Spiral\Sentry\Client` wrapper that simplifies sending exceptions with application state:

```php
use Spiral\Sentry\Client;

final class PaymentService
{
    public function __construct(
        private readonly Client $sentry,
    ) {}
    
    public function processPayment(Payment $payment): void
    {
        try {
            // Process payment
        } catch (\Throwable $e) {
            // Send exception with all collected state (tags, variables, logs)
            $this->sentry->send($e);
            throw $e;
        }
    }
}
```

The `Client::send()` method:

1. Retrieves the current `StateInterface` from the container
2. Configures the Sentry scope with tags, variables, and breadcrumbs
3. Captures the exception and returns the event ID

This is particularly useful when you want to manually report exceptions while including the full application context.

### Snapshots Integration

When using `SentryBootloader` instead of `SentryReporterBootloader`, the bridge registers a `SentrySnapshotter` that
implements `SnapshotterInterface`. This makes Sentry the primary handler for creating exception snapshots:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Sentry\Bootloader\SentryBootloader::class,
    // ...
];
```

With this configuration:

- Exceptions automatically create snapshots in Sentry
- Snapshot IDs are Sentry event IDs
- You can reference errors by their Sentry event ID
- Other snapshot handlers (file, storage) are bypassed

**When to use snapshots:**

- Use `SentryReporterBootloader` if you want exceptions reported to Sentry AND saved locally
- Use `SentryBootloader` if Sentry should be the only destination for exception data

### Best Practices

**Security:**

- Never enable `send_default_pii` unless required and compliant with privacy regulations
- Use `ignore_exceptions` to exclude exceptions containing sensitive data
- Implement `before_send` callback to scrub sensitive information from events
- Be cautious with state collectors that might capture sensitive data

**Performance:**

- Use appropriate `sample_rate` values (0.1-0.5) for high-traffic applications
- Enable `traces_sample_rate` only when needed for performance monitoring
- Keep state collectors lightweight and avoid expensive operations
- Consider the performance impact of breadcrumb collection

**Debugging:**

- Use tags to make errors searchable in Sentry (environment, version, feature flags)
- Add contextual variables to help reproduce issues (user ID, request ID, session data)
- Include breadcrumbs to understand the sequence of events leading to errors
- Leverage Sentry's release tracking to identify when issues were introduced

**Organization:**

- Create custom integrations for application-specific concerns
- Use separate Sentry projects for different environments (production, staging)
- Configure appropriate `environment` values to distinguish between deployments
- Use `release` tracking to correlate errors with specific code versions