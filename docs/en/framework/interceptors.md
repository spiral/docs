# Framework — Interceptors

One of the key features of Spiral is its support for interceptors, which can be used to add functionality to the 
application without modifying the core code of the application. This can help to keep your codebase more modular
and maintainable.

**Benefits of using interceptors:**

- **Separation of Concerns:** Using interceptors allows you to keep the different parts of your application separate and
  organized. For example, you can use an interceptor to handle authentication without having to add that code to every
  single part of your application that requires authentication.
- **Reusability:** With interceptors, you can write code once and use it in multiple parts of your application, reducing code duplication.
- **Modularity:** The ability to add, remove or replace interceptors without affecting the rest of the application makes
  it more flexible and easy to update.
- **Performance:** Interceptors can be used to optimize the performance of the application by caching responses,
  reducing the number of database queries, and more.
- **Ease of use:** Adding interceptors to your application is relatively easy and straightforward.

You can use interceptors with various components such as:

- [HTTP](../http/interceptors.md)
- [Events](../advanced/events.md#interceptors)
- [gRPC](../grpc/interceptors.md)
- [Websockets](../websockets/interceptors.md)
- [Queue](../queue/interceptors.md)
- [Temporal](../temporal/interceptors.md)

## Interceptor Interface

Interceptors implement the `Spiral\Interceptors\InterceptorInterface`:

```php
namespace Spiral\Interceptors;

use Spiral\Interceptors\Context\CallContextInterface;

interface InterceptorInterface
{
    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed;
}
```

The interface provides a flexible way to intercept calls to any target, whether it's a method, function, or custom handler.

## Call Context

The `CallContextInterface` contains all the information about the intercepted call:

- **Target** — the definition of the call target (method, function, closure, etc.)
- **Arguments** — the list of arguments for the call
- **Attributes** — additional context that can be used to pass data between interceptors

```php
namespace Spiral\Interceptors\Context;

interface CallContextInterface extends AttributedInterface
{
    public function getTarget(): TargetInterface;
    public function getArguments(): array;
    public function withTarget(TargetInterface $target): static;
    public function withArguments(array $arguments): static;
    
    // Methods from AttributedInterface:
    public function getAttributes(): array;
    public function getAttribute(string $name, mixed $default = null): mixed;
    public function withAttribute(string $name, mixed $value): static;
    public function withoutAttribute(string $name): static;
}
```

> **Note**
> `CallContextInterface` is immutable, so calls to `withTarget()` and `withArguments()` return a new instance with the updated values.

## Target Interface

The `TargetInterface` defines the target whose call you want to intercept. It can represent a method, function, closure, or even a path string for RPC or message queue endpoints.

```php
namespace Spiral\Interceptors\Context;

interface TargetInterface extends \Stringable
{
    public function getPath(): array;
    public function withPath(array $path, ?string $delimiter = null): static;
    public function getReflection(): ?\ReflectionFunctionAbstract;
    public function getObject(): ?object;
    public function getCallable(): callable|array|null;
}
```

### Creating Targets

The static factory methods in the `Target` class make it easy to create different types of targets:

```php
use Spiral\Interceptors\Context\Target;

// From a method reflection
$target = Target::fromReflectionMethod(new \ReflectionMethod(UserController::class, 'show'), UserController::class);

// From a function reflection
$target = Target::fromReflectionFunction(new \ReflectionFunction('array_map'));

// From a closure
$target = Target::fromClosure(fn() => 'Hello, World!');

// From a path string (for RPC endpoints or message queue handlers)
$target = Target::fromPathString('user.show');

// From a controller-action pair
$target = Target::fromPair(UserController::class, 'show');
```

## Creating an Interceptor

Let's create a simple interceptor that logs the execution time of a call:

```php
namespace App\Interceptor;

use Psr\Log\LoggerInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

class ExecutionTimeInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {
    }

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $target = $context->getTarget();
        $startTime = \microtime(true);

        try {
            return $handler->handle($context);
        } finally {
            $executionTime = \microtime(true) - $startTime;

            $this->logger->debug(
                'Target executed',
                [
                    'target' => (string)$target,
                    'execution_time' => $executionTime,
                ]
            );
        }
    }
}
```

This interceptor:
1. Records the start time
2. Passes the call to the next handler in the chain
3. Calculates and logs the execution time once the handler is complete

## Building an Interceptor Pipeline

To use interceptors, you need to build an interceptor pipeline using the `PipelineBuilderInterface`:

```php
use Spiral\Interceptors\PipelineBuilder;
use Spiral\Interceptors\Context\CallContext;
use Spiral\Interceptors\Context\Target;
use Spiral\Interceptors\Handler\CallableHandler;
use App\Interceptor\ExecutionTimeInterceptor;
use App\Interceptor\AuthorizationInterceptor;

// Create interceptors
$interceptors = [
    new ExecutionTimeInterceptor($logger),
    new AuthorizationInterceptor($auth),
];

// Build the pipeline
$pipeline = (new PipelineBuilder())
    ->withInterceptors(...$interceptors)
    ->build(new CallableHandler());

// Create call context
$context = new CallContext(
    target: Target::fromPair(UserController::class, 'show'),
    arguments: [42],
    attributes: ['request' => $request]
);

// Execute the pipeline
$result = $pipeline->handle($context);
```

## Handlers

The pipeline should end with a handler that executes the target. Spiral provides several built-in handlers:

### CallableHandler

The `CallableHandler` simply calls the target without any additional processing:

```php
use Spiral\Interceptors\Handler\CallableHandler;

$handler = new CallableHandler();
```

### AutowireHandler

The `AutowireHandler` resolves missing arguments using the container:

```php
use Spiral\Interceptors\Handler\AutowireHandler;

$handler = new AutowireHandler($container);
```

This handler is useful when working with controllers where you want to automatically inject service dependencies.

## Advanced Example

Here's a more comprehensive example of using interceptors:

```php
namespace App\Controller;

use App\Interceptor\AuthorizationInterceptor;
use App\Interceptor\CacheInterceptor;
use App\Interceptor\ExecutionTimeInterceptor;
use App\User\UserService;
use Psr\Container\ContainerInterface;
use Spiral\Core\Attribute\Proxy;
use Spiral\Core\CompatiblePipelineBuilder;
use Spiral\Interceptors\Context\CallContext;
use Spiral\Interceptors\Context\Target;
use Spiral\Interceptors\Handler\AutowireHandler;

class UserController
{
    private $pipeline;

    public function __construct(
        private readonly UserService $userService,
        #[Proxy] ContainerInterface $container
    ) {
        // Build the pipeline with interceptors
        $this->pipeline = (new CompatiblePipelineBuilder())
            ->withInterceptors(
                new ExecutionTimeInterceptor($container->get(LoggerInterface::class)),
                new AuthorizationInterceptor($container->get(AuthInterface::class)),
                new CacheInterceptor($container->get(CacheInterface::class))
            )
            ->build(new AutowireHandler($container));
    }

    public function show(int $id)
    {
        // Create a context for the target method
        $context = new CallContext(
            target: Target::fromReflectionMethod(
                new \ReflectionMethod($this->userService, 'findUser'),
                $this->userService
            ),
            arguments: ['id' => $id]
        );

        // Execute the pipeline
        return $this->pipeline->handle($context);
    }
}
```

In this example:
1. We build a pipeline with execution time logging, authorization checks, and result caching
2. We use the `AutowireHandler` to resolve missing arguments from the container
3. The controller method creates a context targeting the `findUser` method of the `UserService`
4. The pipeline executes all interceptors and then calls the target method

> **Note**
> To learn about Container Scopes and Proxy objects, see the [IoC Scopes](../container/scopes.md) section in our documentation.

## Comparison with Legacy Interceptors

> **Note**
> The old implementation of interceptors based on `spiral/hmvc` is no longer recommended.
> You can find the old documentation at [https://spiral.dev/docs/framework-interceptors/3.13](https://spiral.dev/docs/framework-interceptors/3.13).

In Spiral 3.14.0, a new implementation of interceptors was introduced in the `spiral/interceptors` package.
Here's how the new implementation differs from the legacy one:

:::: tabs

::: tab Legacy Interceptors
```php
namespace App\Interceptor;

use Psr\SimpleCache\CacheInterface;
use Spiral\Core\CoreInterface;
use Spiral\Core\CoreInterceptorInterface;

class CacheInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly CacheInterface $cache,
        private readonly int $ttl = 3600,
    ) {}

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        // Step 1: Generate a cache key based on controller, action, and parameters
        $cacheKey = $this->generateCacheKey($controller, $action, $parameters);

        // Step 2: Check if the result is already cached
        if ($this->cache->has($cacheKey)) {
            // Return cached result if available
            return $this->cache->get($cacheKey);
        }

        // Step 3: Execute the controller action if no cached result
        $result = $core->callAction($controller, $action, $parameters);

        // Step 4: Cache the result for future requests
        if ($this->isCacheable($result)) {
            $this->cache->set($cacheKey, $result, $this->ttl);
        }

        return $result;
    }

    private function generateCacheKey(string $controller, string $action, array $parameters): string
    {
        // Create a deterministic cache key from controller, action, and parameters
        return \md5($controller . '::' . $action . '::' . \serialize($parameters));
    }

    private function isCacheable(mixed $result): bool
    {
        // Only cache serializable results
        return !\is_resource($result) && (
                \is_scalar($result) ||
                \is_array($result) ||
                $result instanceof \Serializable ||
                $result instanceof \stdClass
            );
    }
}
```
:::

::: tab New Interceptors
```php
namespace App\Interceptor;

use Psr\SimpleCache\CacheInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\Context\TargetInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

final class CacheInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly CacheInterface $cache,
        private readonly int $ttl = 3600,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        // Step 1: Generate a cache key using target path and arguments
        $cacheKey = $this->generateCacheKey($context->getTarget(), $context->getArguments());

        // Step 2: Check if the result is already cached
        if ($this->cache->has($cacheKey)) {
            // Return cached result if available
            return $this->cache->get($cacheKey);
        }

        // Step 3: Execute the target if no cached result
        $result = $handler->handle($context);

        // Step 4: Cache the result for future requests
        if ($this->isCacheable($result)) {
            $this->cache->set($cacheKey, $result, $this->ttl);
        }

        return $result;
    }

    private function generateCacheKey(TargetInterface $target, array $args): string
    {
        // Create a deterministic cache key from target string and arguments
        return \md5((string) $target . '::' . serialize($args));
    }

    private function isCacheable(mixed $result): bool
    {
        // Only cache serializable results
        return !\is_resource($result) && (
                \is_scalar($result) ||
                \is_array($result) ||
                $result instanceof \Serializable ||
                $result instanceof \stdClass
            );
    }
}
```
:::

::::

### Interface Changes

**Legacy Interface:**
```php
interface CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed;
}
```

**New Interface:**
```php
interface InterceptorInterface
{
    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed;
}
```

The main differences:
- The `$controller` and `$action` parameters have been replaced with a more flexible `Target` concept
- The `$parameters` array is now part of the `CallContext`
- Instead of a `CoreInterface`, there's a `HandlerInterface` which offers more flexibility

### Compatibility

In Spiral 3.x, both legacy interceptors (`CoreInterceptorInterface`) and new interceptors (`InterceptorInterface`) are supported.
However, the legacy implementation is deprecated and will be excluded in Spiral 4.x.

If you need to use both implementations together, use the `CompatiblePipelineBuilder`:

```php
use Spiral\Core\CompatiblePipelineBuilder;
use Spiral\Core\CoreInterceptorInterface; // Legacy interceptor
use Spiral\Interceptors\InterceptorInterface; // New interceptor

$pipeline = (new CompatiblePipelineBuilder())
    ->withInterceptors(
        new LegacyStyleInterceptor(), // Implements CoreInterceptorInterface
        new NewStyleInterceptor()     // Implements InterceptorInterface
    )
    ->build(new CallableHandler());
```

### Migration Guidelines

When migrating from the legacy implementation:
- Replace `CoreInterceptorInterface` with `InterceptorInterface`
- Use a `Target` instead of `$controller` and `$action` parameters
- Move `$parameters` to the `CallContext`
- Replace `$core->callAction()` with `$handler->handle()`

## Events

| Event                                        | Description                                     |
|----------------------------------------------|-------------------------------------------------|
| Spiral\Interceptors\Event\InterceptorCalling | Fired before calling an interceptor.            |

> **Warning**
> The `Spiral\Core\Event\InterceptorCalling` event is only dispatched by the deprecated `\Spiral\Core\InterceptorPipeline` 
> from the legacy implementation. The framework's new implementation does not use this event.

> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
