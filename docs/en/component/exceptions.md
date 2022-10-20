# Exceptions

The Spiral Framework provides an `Exceptions` component for handling exceptions in an application.
Provides an interface and implementation of `ExceptionHandlerInterface`, functionality of `Reporters` and `Renderers`.
The component available and configured by default in the [application bundle](https://github.com/spiral/app).

## Installation

To enable the component, you need to add `ExceptionHandler` class as `exceptionHandler` parameter when you create
an application instance.

```php
// app.php
$app = App::create(
    directories: ['root' => __DIR__],
    exceptionHandler: \Spiral\Exceptions\ExceptionHandler::class
)->run();
```

> **Note**
> The [application bundle](https://github.com/spiral/app) uses the `App\Exception\Handler` class, 
> which is inherited from `ExceptionHandler`.

## Configuration

The application bundle contains an `App\Bootloader\ExceptionHandlerBootloader` that registers some reporters 
and renderers. You can add your own renderers and reporters in this bootloader.

```php
namespace App\Bootloader;

use App\ErrorHandler\ViewRenderer;
use App\Exception\CollisionRenderer;
use Spiral\Boot\AbstractKernel;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;
use Spiral\Exceptions\ExceptionHandlerInterface;
use Spiral\Exceptions\Renderer\JsonRenderer;
use Spiral\Exceptions\Reporter\FileReporter;
use Spiral\Exceptions\Reporter\LoggerReporter;
use Spiral\Http\ErrorHandler\RendererInterface;
use Spiral\Http\Middleware\ErrorHandlerMiddleware\EnvSuppressErrors;
use Spiral\Http\Middleware\ErrorHandlerMiddleware\SuppressErrorsInterface;
use Spiral\Ignition\IgnitionRenderer;

final class ExceptionHandlerBootloader extends Bootloader
{
    protected const BINDINGS = [
        SuppressErrorsInterface::class => EnvSuppressErrors::class,
        RendererInterface::class => ViewRenderer::class,
    ];

    public function init(AbstractKernel $kernel): void
    {
        $kernel->running(static function (ExceptionHandler $handler): void {
            $handler->addRenderer(new CollisionRenderer());
            $handler->addRenderer(new JsonRenderer());
        });
    }

    public function boot(
        ExceptionHandlerInterface $handler,
        LoggerReporter $logger,
        FileReporter $files,
        IgnitionRenderer $ignition
    ): void {
        $handler->addReporter($logger);
        $handler->addReporter($files);

        $handler->addRenderer($ignition);
    }
}
```

## Reporters

The `Exceptions` component allows to create and register exception `reporters`.
The interface `Spiral\Exceptions\ExceptionReporterInterface` for reporters is very simple, it contains one `report` 
method, which accepts an exception.

```php
namespace Spiral\Exceptions;

interface ExceptionReporterInterface
{
    public function report(\Throwable $exception): void;
}
```

### Available Reporters

The `Spiral\Exceptions\Reporter\LoggerReporter` reporter are available out of the box.
Additionally, the Spiral Framework provides a `Spiral\Exceptions\Reporter\FileReporter` reporter that isn't part 
of the `Exceptions` component but is available by default in the application.

#### LoggerReporter

This reporter adds the ability to log an exception using a logger registered in the application. Logger reporter enabled
by default in the application bundle.

#### FileReporter

This reporter adds the ability to create and save in a file `snapshot` with detailed information from an exception.

In the [application bundle](https://github.com/spiral/app) this reporter enabled by default.

### Adding a Reporter

To add a new reporter, use the `addReporter` method in the `ExceptionHandler` class. By default, this class bound 
to the `ExceptionHandlerInterface` interface and we can get it from the container using it. You can use the 
`Spiral\Boot\Environment\AppEnvironment` class to determine the current application environment and 
register reporters based on that.

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\Environment\AppEnvironment;
use Spiral\Exceptions\ExceptionHandlerInterface;
use Spiral\Exceptions\Reporter\LoggerReporter;
use Spiral\Exceptions\Reporter\FileReporter;
use Spiral\Ignition\IgnitionRenderer;
use Spiral\Sentry\SentryReporter;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function boot(
        ExceptionHandlerInterface $handler,
        LoggerReporter $logger,
        FileReporter $files,
        IgnitionRenderer $ignition,
        SentryReporter $sentry,
        AppEnvironment $env
    ): void {
        $handler->addReporter($logger);

        if ($env->isLocal()) {
            $handler->addRenderer($ignition);
            $handler->addReporter($files);
        } else {
            $handler->addReporter($sentry);
        }
    }
}
```

## Renderers

The `Exceptions` component allows to create and register exception `renderers`.
Renders are used to create a `string representation` of the exception according to the required `format`. 
The application can contain different renders for different formats.
Renderers must implement the `Spiral\Exceptions\ExceptionRendererInterface` interface with two methods `canRender` and
`render`.

```php
namespace Spiral\Exceptions;

interface ExceptionRendererInterface
{
    /**
     * @param string|null $format Preferred format
     */
    public function render(
        \Throwable $exception,
        ?Verbosity $verbosity = Verbosity::BASIC,
        string $format = null,
    ): string;

    public function canRender(string $format): bool;
}
```

### Available Renderers

The `Spiral\Exceptions\Renderer\ConsoleRenderer`, `Spiral\Exceptions\Renderer\JsonRenderer`,  
`Spiral\Exceptions\Renderer\PlainRenderer` renderers are available out of the box.

| Renderer        | Formats                               |
|-----------------|---------------------------------------|
| ConsoleRenderer | console, cli                          |
| JsonRenderer    | application/json, json                |
| PlainRenderer   | text/plain, text, plain, cli, console |


### Adding a Renderer

To add a new renderer, use the `addRenderer` method in the `ExceptionHandler` class. By default, this class bound
to the `ExceptionHandlerInterface` interface and we can get it from the container using it.

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandlerInterface;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function boot(ExceptionHandlerInterface $handler): void 
    {
        $handler->addRenderer(new SomeRenderer());
    }
}
```
